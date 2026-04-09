# Entitlement endpoint implementation (Worker)

This adds a public app-facing endpoint so clients can validate entitlement on:
- app launch
- app resume
- pre-connect

## Proposed endpoint
`POST /api/entitlement-check`

Request:
```json
{ "email": "user@example.com" }
```

Success (`200`):
```json
{
  "ok": true,
  "email": "user@example.com",
  "entitled": true,
  "reason": "active_subscription"
}
```

Inactive (`200`):
```json
{
  "ok": true,
  "email": "user@example.com",
  "entitled": false,
  "reason": "no_active_subscription"
}
```

## Worker code patch (drop into `src/index.js`)
```js
if (url.pathname === "/api/entitlement-check" && request.method === "POST") {
  const cors = { "Access-Control-Allow-Origin": "https://bb-vpn.com" };
  const body = await request.json().catch(() => null);
  const emailRaw = body?.email ? String(body.email) : "";
  const email = emailRaw.trim().toLowerCase();

  if (!email) {
    return new Response(JSON.stringify({ error: "Missing email" }), {
      status: 400,
      headers: { ...cors, "Content-Type": "application/json" },
    });
  }

  try {
    const entitled = await hasActiveSubscriptionByEmail(env, email);
    return new Response(
      JSON.stringify({
        ok: true,
        email,
        entitled,
        reason: entitled ? "active_subscription" : "no_active_subscription",
      }),
      { headers: { ...cors, "Content-Type": "application/json" } }
    );
  } catch (e) {
    return new Response(JSON.stringify({ error: "entitlement_check_failed", detail: e?.message || String(e) }), {
      status: 500,
      headers: { ...cors, "Content-Type": "application/json" },
    });
  }
}

async function hasActiveSubscriptionByEmail(env, email) {
  const search = await fetch("https://api.stripe.com/v1/customers/search", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${env.STRIPE_SECRET_KEY}`,
      "Content-Type": "application/x-www-form-urlencoded",
    },
    body: new URLSearchParams({ query: `email:'${email.replace(/'/g, "\\'")}'` }),
  });

  const s = await search.json();
  if (!search.ok) throw new Error("stripe customer search failed: " + JSON.stringify(s));

  const customers = Array.isArray(s.data) ? s.data : [];
  for (const c of customers) {
    const subRes = await fetch(
      `https://api.stripe.com/v1/subscriptions?customer=${encodeURIComponent(c.id)}&status=all&limit=20`,
      { headers: { Authorization: `Bearer ${env.STRIPE_SECRET_KEY}` } }
    );
    const subs = await subRes.json();
    if (!subRes.ok) throw new Error("stripe subscriptions lookup failed: " + JSON.stringify(subs));

    const active = (subs.data || []).some((x) => ["active", "trialing", "past_due"].includes(x.status));
    if (active) return true;
  }

  return false;
}
```

## App behavior
- If `entitled=false`: disconnect tunnel, mark account inactive, block connect.
- Recheck at launch/resume/pre-connect.
