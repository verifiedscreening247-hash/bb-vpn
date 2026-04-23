# BB VPN — Техзадание для Семена: Cloudflare Integration

> **Задача:** связать готовую админку (`admin.html`) с реальными данными через Cloudflare Workers + Stripe + Resend.
> Всё уже на Cloudflare — нужно дописать Workers и настроить сервисы.

---

## Стек проекта

| Сервис | Назначение |
|--------|-----------|
| **Cloudflare Workers** | API-сервер (уже используется) |
| **Cloudflare D1** | SQLite база данных (пользователи, тикеты, рефералы) |
| **Cloudflare KV** | Кэш токенов и промокодов |
| **Cloudflare Email Routing** | Входящие письма на support@bb-vpn.com |
| **Stripe** | Платежи (уже подключён) |
| **Resend** | Исходящие письма (уже используется) |

---

## Что нужно сделать

### 1. Cloudflare D1 — База данных

Создай базу: `dash.cloudflare.com` → D1 → Create database → `bbvpn-db`

Выполни миграцию:

```sql
CREATE TABLE IF NOT EXISTS users (
  id TEXT PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  plan TEXT NOT NULL DEFAULT 'monthly',
  status TEXT NOT NULL DEFAULT 'active',
  stripe_customer_id TEXT,
  stripe_subscription_id TEXT,
  autopay INTEGER DEFAULT 1,
  created_at TEXT DEFAULT (datetime('now')),
  next_billing_at TEXT,
  referral_code TEXT UNIQUE,
  referred_by TEXT
);

CREATE TABLE IF NOT EXISTS devices (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  name TEXT,
  region TEXT NOT NULL,
  wg_public_key TEXT,
  created_at TEXT DEFAULT (datetime('now')),
  FOREIGN KEY(user_id) REFERENCES users(id)
);

CREATE TABLE IF NOT EXISTS payments (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  stripe_charge_id TEXT,
  amount INTEGER NOT NULL,
  currency TEXT DEFAULT 'usd',
  plan TEXT,
  status TEXT NOT NULL,
  created_at TEXT DEFAULT (datetime('now')),
  FOREIGN KEY(user_id) REFERENCES users(id)
);

CREATE TABLE IF NOT EXISTS refunds (
  id TEXT PRIMARY KEY,
  payment_id TEXT NOT NULL,
  user_id TEXT NOT NULL,
  amount INTEGER NOT NULL,
  reason TEXT,
  created_at TEXT DEFAULT (datetime('now')),
  FOREIGN KEY(payment_id) REFERENCES payments(id)
);

CREATE TABLE IF NOT EXISTS promo_codes (
  id TEXT PRIMARY KEY,
  code TEXT UNIQUE NOT NULL,
  type TEXT NOT NULL DEFAULT 'regular',
  value TEXT NOT NULL,
  used_count INTEGER DEFAULT 0,
  valid_from TEXT,
  valid_to TEXT,
  active INTEGER DEFAULT 1,
  created_at TEXT DEFAULT (datetime('now'))
);

CREATE TABLE IF NOT EXISTS referral_payouts (
  id TEXT PRIMARY KEY,
  referrer_id TEXT NOT NULL,
  amount INTEGER NOT NULL,
  status TEXT DEFAULT 'pending',
  paid_at TEXT,
  FOREIGN KEY(referrer_id) REFERENCES users(id)
);

CREATE TABLE IF NOT EXISTS support_tickets (
  id TEXT PRIMARY KEY,
  user_email TEXT NOT NULL,
  subject TEXT NOT NULL,
  status TEXT DEFAULT 'open',
  created_at TEXT DEFAULT (datetime('now'))
);

CREATE TABLE IF NOT EXISTS support_messages (
  id TEXT PRIMARY KEY,
  ticket_id TEXT NOT NULL,
  sender TEXT NOT NULL,
  body TEXT NOT NULL,
  created_at TEXT DEFAULT (datetime('now')),
  FOREIGN KEY(ticket_id) REFERENCES support_tickets(id)
);
```

---

### 2. Cloudflare Worker — Admin API

Создай воркер `bbvpn-admin-api`:

```js
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    const path = url.pathname;
    const headers = {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': 'https://bb-vpn.com',
    };

    // GET /api/admin/stats
    if (path === '/api/admin/stats') {
      const users   = await env.DB.prepare('SELECT COUNT(*) as c FROM users WHERE status="active"').first();
      const revenue = await env.DB.prepare('SELECT SUM(amount) as s FROM payments WHERE status="paid"').first();
      const tickets = await env.DB.prepare('SELECT COUNT(*) as c FROM support_tickets WHERE status="open"').first();
      return Response.json({ users: users.c, revenue: revenue.s, openTickets: tickets.c }, { headers });
    }

    // GET /api/admin/users
    if (path === '/api/admin/users') {
      const { results } = await env.DB.prepare('SELECT * FROM users ORDER BY created_at DESC').all();
      return Response.json(results, { headers });
    }

    // GET /api/admin/users/:id
    if (path.startsWith('/api/admin/users/')) {
      const id   = path.split('/').pop();
      const user = await env.DB.prepare('SELECT * FROM users WHERE id=?').bind(id).first();
      const pays = await env.DB.prepare('SELECT * FROM payments WHERE user_id=? ORDER BY created_at DESC').bind(id).all();
      const devs = await env.DB.prepare('SELECT * FROM devices WHERE user_id=?').bind(id).all();
      return Response.json({ user, payments: pays.results, devices: devs.results }, { headers });
    }

    // GET /api/admin/payments
    if (path === '/api/admin/payments') {
      const { results } = await env.DB.prepare(
        'SELECT p.*, u.email FROM payments p JOIN users u ON p.user_id=u.id ORDER BY p.created_at DESC'
      ).all();
      return Response.json(results, { headers });
    }

    // GET /api/admin/refunds
    if (path === '/api/admin/refunds') {
      const { results } = await env.DB.prepare(
        'SELECT r.*, u.email FROM refunds r JOIN users u ON r.user_id=u.id ORDER BY r.created_at DESC'
      ).all();
      return Response.json(results, { headers });
    }

    // POST /api/admin/promo
    if (path === '/api/admin/promo' && request.method === 'POST') {
      const body = await request.json();
      await env.DB.prepare(
        'INSERT INTO promo_codes (id,code,type,value,valid_from,valid_to) VALUES (?,?,?,?,?,?)'
      ).bind(crypto.randomUUID(), body.code, body.type, body.value, body.validFrom, body.validTo).run();
      return Response.json({ ok: true }, { headers });
    }

    // DELETE /api/admin/promo/:code
    if (path.startsWith('/api/admin/promo/') && request.method === 'DELETE') {
      const code = path.split('/').pop();
      await env.DB.prepare('UPDATE promo_codes SET active=0 WHERE code=?').bind(code).run();
      return Response.json({ ok: true }, { headers });
    }

    // GET /api/admin/tickets
    if (path === '/api/admin/tickets') {
      const { results } = await env.DB.prepare('SELECT * FROM support_tickets ORDER BY created_at DESC').all();
      return Response.json(results, { headers });
    }

    // POST /api/admin/support-reply
    if (path === '/api/admin/support-reply' && request.method === 'POST') {
      const { ticketId, toEmail, subject, body } = await request.json();
      await env.DB.prepare(
        'INSERT INTO support_messages (id,ticket_id,sender,body) VALUES (?,?,?,?)'
      ).bind(crypto.randomUUID(), ticketId, 'admin', body).run();
      await env.DB.prepare('UPDATE support_tickets SET status="resolved" WHERE id=?').bind(ticketId).run();
      await fetch('https://api.resend.com/emails', {
        method: 'POST',
        headers: { 'Authorization': `Bearer ${env.RESEND_API_KEY}`, 'Content-Type': 'application/json' },
        body: JSON.stringify({
          from: 'BB VPN Support <support@bb-vpn.com>',
          to: [toEmail],
          subject: `Re: ${subject}`,
          html: `<p>${body.replace(/\n/g,'<br>')}</p><hr><p style="color:#888;font-size:12px">BB VPN Support</p>`,
        }),
      });
      return Response.json({ ok: true }, { headers });
    }

    // GET /api/admin/referrals
    if (path === '/api/admin/referrals') {
      const { results } = await env.DB.prepare(`
        SELECT u.email, u.referral_code,
          COUNT(r.id) as referred_count,
          SUM(p.amount) as revenue
        FROM users u
        LEFT JOIN users r ON r.referred_by = u.id
        LEFT JOIN payments p ON p.user_id = r.id AND p.status='paid'
        WHERE u.referral_code IS NOT NULL
        GROUP BY u.id
      `).all();
      return Response.json(results, { headers });
    }

    return new Response('Not found', { status: 404 });
  },

  // Входящая почта → тикет
  async email(message, env) {
    const subject = message.headers.get('subject') || '(без темы)';
    const from    = message.from;
    const body    = await new Response(message.raw).text();
    const ticketId = crypto.randomUUID();
    await env.DB.prepare(
      'INSERT INTO support_tickets (id,user_email,subject) VALUES (?,?,?)'
    ).bind(ticketId, from, subject).run();
    await env.DB.prepare(
      'INSERT INTO support_messages (id,ticket_id,sender,body) VALUES (?,?,?,?)'
    ).bind(crypto.randomUUID(), ticketId, from, body).run();
  }
};
```

**Переменные окружения воркера** (Settings → Variables → Add):
```
DB             → привязать D1 базу bbvpn-db (Bindings → D1)
RESEND_API_KEY → ключ из resend.com/api-keys
```

---

### 3. Stripe Webhook → D1

Дополни существующий Stripe webhook-воркер:

```js
if (event.type === 'checkout.session.completed') {
  const s = event.data.object;
  await env.DB.prepare(
    'INSERT OR IGNORE INTO payments (id,user_id,stripe_charge_id,amount,status,plan) VALUES (?,?,?,?,?,?)'
  ).bind(crypto.randomUUID(), s.client_reference_id, s.payment_intent, s.amount_total, 'paid', s.metadata?.plan).run();
}

if (event.type === 'charge.refunded') {
  const c = event.data.object;
  await env.DB.prepare(
    'INSERT INTO refunds (id,user_id,payment_id,amount) VALUES (?,?,?,?)'
  ).bind(crypto.randomUUID(), c.metadata?.user_id, c.id, c.amount_refunded).run();
}
```

---

### 4. Email Routing — входящий support@bb-vpn.com

1. `dash.cloudflare.com` → **Email** → **Email Routing** → включить для `bb-vpn.com`
2. Custom addresses → Add → `support@bb-vpn.com` → Action: **Send to Worker** → `bbvpn-admin-api`
3. Воркер уже содержит обработчик `email()` из шага 2

---

### 5. Cloudflare Access — защита /admin

1. `dash.cloudflare.com` → **Zero Trust** → **Access** → **Applications** → Add
2. Тип: **Self-hosted**
3. Domain: `bb-vpn.com` / Path: `admin*`
4. Policy → Allow → Emails:
   - `verifiedscreening247@gmail.com`
   - твой email (Семен)
   - email Палмера
5. Save

После этого `bb-vpn.com/admin.html` откроется только для трёх email через Google OAuth.

---

### 6. Подключить admin.html к API

Заменить моковые данные на fetch. Пример:

```js
async function loadStats() {
  const data = await fetch('/api/admin/stats').then(r => r.json());
  document.getElementById('stat-users').textContent   = data.users;
  document.getElementById('stat-revenue').textContent = '$' + (data.revenue / 100).toFixed(2);
  document.getElementById('stat-tickets').textContent = data.openTickets;
}
loadStats();
```

Все endpoints описаны в шаге 2.

---

## Порядок выполнения

- [ ] Создать D1 базу `bbvpn-db` и выполнить SQL-миграцию
- [ ] Создать воркер `bbvpn-admin-api` с кодом из шага 2
- [ ] Привязать D1 binding и RESEND_API_KEY к воркеру
- [ ] Дополнить Stripe webhook сохранением в D1 (шаг 3)
- [ ] Настроить Email Routing → support@bb-vpn.com → воркер (шаг 4)
- [ ] Настроить Cloudflare Access для /admin* (шаг 5)
- [ ] Заменить mock-данные в admin.html на fetch-запросы (шаг 6)

---

Вопросы — пиши в чат или создавай PR в ветку `semen`.

## Цель

Сделать GitHub единственным источником правок и автоматически деплоить код на серверы.

## Как работать

1. Делай изменения локально.
2. `git add .`
3. `git commit -m "..."`
4. `git push origin main`

После пуша GitHub Actions автоматически развернёт сайт на сервере.

## Автоматический деплой

Рабочий процесс настроен в `.github/workflows/deploy.yml`.

### Требуемые secrets

В репозитории GitHub нужно добавить:

- `VULTR_HOST` — IP или хост сервера
- `VULTR_USER` — SSH-пользователь
- `VULTR_DESTINATION` — путь на сервере, куда копировать файлы (например `/var/www/bb-vpn`)
- `VULTR_SSH_KEY` — приватный SSH-ключ для доступа к серверу
- `VULTR_PORT` — порт SSH (необязательно, по умолчанию 22)

## Архитектура деплоя

1. Ты пушишь изменения в ветку `main`.
2. GitHub Actions клонирует репозиторий.
3. Файлы копируются на сервер через `rsync`.
4. На сервере остаётся только один источник правок — GitHub.

## Серверная настройка

На сервере достаточно один раз подготовить директорию и разрешения:

```bash
mkdir -p /var/www/bb-vpn
chown -R $USER:$USER /var/www/bb-vpn
```

Если нужно, можно настроить веб-сервер (nginx/apache) на эту папку.

## Когда нельзя редактировать на сервере

- не меняй файлы напрямую на сервере;
- любые изменения должны приходить через GitHub и деплой;
- это гарантирует, что все серверы остаются в одном состоянии.
