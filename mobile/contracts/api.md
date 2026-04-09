# BB VPN API Contract (validated from Worker + FastAPI + WireGuard scripts)

## Base URLs
- Public worker base: `https://bb-vpn.com`
- Regional API bases: `VPN_API_EU`, `VPN_API_US`, `VPN_API_SG`

## Worker endpoints (mobile-facing)

### `POST /api/request-new-link`
Request:
```json
{ "email": "user@example.com" }
```

Success (`200`):
```json
{ "ok": true, "left": 3 }
```

Known errors:
- `400 {"error":"Missing email"}`
- `403 {"error":"Device limit reached","left":0,"details":{...}}`
- `500 {"error":"Missing env/bindings","missing":[...]}`
- `500 {"error":"device_count failed",...}`
- `500 {"error":"resend_failed",...}`

Behavior notes:
- Worker checks `device_count` against EU region API before issuing a token.
- Token is stored in KV for 24 hours.
- Link format: `https://bb-vpn.com/activate.html?t=<token>`.

### `POST /api/activate-all`
Request:
```json
{ "token": "<one-time-token>", "device": "iphone15" }
```

Success (`200`):
```json
{
  "ok": true,
  "device": "iphone15",
  "email": "user@example.com",
  "configs": {
    "eu": "[Interface]...",
    "us": "[Interface]...",
    "sg": "[Interface]..."
  },
  "errors": {}
}
```

Known errors:
- `400 {"error":"Missing token or device"}`
- `403 {"error":"Invalid or expired link"}`
- `400 {"error":"All regions failed","errors":{...}}`

Behavior notes:
- Device value is sanitized to `[a-zA-Z0-9_-]` and max length 24.
- Partial success is allowed (`configs` + `errors` together).
- Token is deleted once at least one region succeeds.

## Regional API endpoints (worker-to-server)

All require:
`Authorization: Bearer <PROVISION_TOKEN>`

### `POST /provision`
Request:
```json
{ "email": "user@example.com", "device": "iphone15" }
```

Success (`200`):
```json
{
  "ok": true,
  "config": "[Interface]\n...",
  "conf_path": "/etc/wireguard/accounts/user@example.com/iphone15.conf"
}
```

Errors:
- `401` missing/invalid bearer format
- `403` wrong token
- `400` shell provisioning failure (`add_device.sh` output)
- `500` generated config file not found

Provision behavior from `add_device.sh`:
- Max devices per account: `5`.
- Account/device are sanitized and lowercased in shell script.
- File/peer creation is idempotent for same device key.
- Uses lock `/var/lock/bbvpn-ipalloc.lock` for IP allocation safety.
- Allocates next free IP by reading live `wg show wg0 allowed-ips`.
- Writes peer to live interface and persists to `wg0.conf`.
- Client config values include:
  - `AllowedIPs = 0.0.0.0/0`
  - `PersistentKeepalive = 25`
  - `DNS = 1.1.1.1`

### `GET /device_count?email=...`
Success (`200`):
```json
{
  "ok": true,
  "email": "user@example.com",
  "count": 2,
  "limit": 5,
  "left": 3
}
```

### `POST /revoke_all`
Request:
```json
{ "email": "user@example.com" }
```

Success (`200`):
```json
{ "ok": true, "email": "user@example.com" }
```

Revoke behavior from `revoke_all.sh`:
- Logs to `/var/log/bbvpn-revokes.log`.
- Removes peers from live interface.
- Removes corresponding `[Peer]` blocks from `wg0.conf`.
- Deletes account directory.
- Returns success if account directory does not exist.

## Confirmed regional WireGuard networking
- EU: `wg0` = `10.8.0.1/24`, NAT `10.8.0.0/24` via `enp1s0`, UDP `51820` listening.
- US: `wg0` = `10.9.0.1/24`, NAT `10.9.0.0/24` via `enp1s0`, UDP `51820` listening.
- SG: `wg0` = `10.10.0.1/24`, NAT `10.10.0.0/24` via `enp1s0`, UDP `51820` listening.
- All regions: `net.ipv4.ip_forward = 1`.

### Confirmed region identity inventory
- EU: `bbvpn-eu1` (`45.76.39.110`) key `fQ5RvDPDZ0WISZWPOkqJnEug/vcnussHp0Kh3FHbTlU=`
- US: `bbvpn-us1` (`45.63.7.99`) key `TFEIqMKIifROGMJ6vEmb7s63p+o+ZUwJCmWz65jztHw=`
- SG: `bbvpn-sg1` (`45.76.144.180`) key `hYmRXvY40msAcN59FoQiSjuK3Mclo7E2KX2IOnFDtTY=`
- Inventory reference: `mobile/contracts/region-key-inventory.md`.

### Operations anomaly confirmed
- US live `wg0` config currently includes a peer with `AllowedIPs = 10.8.0.2/32` (EU range), indicating drift/stale state that should be remediated.




## Worker behavior confirmed from `src/index.js`
- CORS allows origin `https://bb-vpn.com` and methods `POST, OPTIONS`.
- `POST /api/stripe-webhook` handles:
  - `checkout.session.completed` (creates token + emails activation link)
  - `customer.subscription.deleted` and `invoice.payment_failed` (calls `/revoke_all` across all configured regions)
- `POST /api/activate-all` returns JSON configs for available regions and deletes token only after at least one success.
- `POST /api/activate` (single-region legacy flow) is still enabled.
- `POST /api/request-new-link` checks `/device_count` on EU API, then sends one-time link email.
- Token TTL is 24 hours (`expirationTtl: 24 * 60 * 60`).
- Activation links emailed by worker currently use HTTPS web URL: `https://bb-vpn.com/activate.html?t=...` (not `bbvpn://` directly).
- `VPN_API_BASE` is not referenced in this worker code path (EU/US/SG env vars are used directly).

## Worker deployment config (wrangler)
Confirmed from shared `wrangler.toml`:
- `name = "bbvpn-provision"`
- `main = "src/index.js"`
- `compatibility_date = "2026-03-13"`
- route: `bb-vpn.com/api/*` on zone `bb-vpn.com`


## Security note on `PROVISION_TOKEN` handling
Regional API services currently use `PROVISION_TOKEN` from systemd environment.
If token values are exposed in chat/screenshots/terminal logs, rotate immediately across **all** regional APIs and the Worker secret to avoid unauthorized provisioning/revocation calls.
Runbook: `mobile/runbooks/provision-token-rotation.md`.

## Worker runtime secrets (Cloudflare)
Confirmed secrets present in Cloudflare Worker settings:
- `FROM_EMAIL`
- `PROVISION_TOKEN`
- `RESEND_API_KEY`
- `STRIPE_SECRET_KEY`
- `STRIPE_WEBHOOK_SECRET`
- `VPN_API_BASE` (legacy/single-region compatibility)
- `VPN_API_EU`
- `VPN_API_US`
- `VPN_API_SG`

Required non-secret bindings (confirmed):
- Binding `TOKENS` -> KV namespace `bbvpn_tokens` (`e448458258e941cc902dabb27e21f74a`)
- Binding `CUSTOMERS` -> KV namespace `bbvpn_customers` (`ba3f248bcedf497997be5d5c187d6b1a`)

## Mobile implementation requirements
- Pre-sanitize and lowercase device names client-side.
- Treat expired token as recoverable and route user to request-new-link.
- Continue onboarding when partial region provisioning succeeds.

### Activation/deeplink behavior (final)
- Primary activation path: `bbvpn://activate?t=<token>`.
- Manual token paste remains supported as fallback.
- Future universal links must resolve into the same token activation flow.

### Entitlement/account lifecycle behavior (final)
- If backend revokes account (`revoke_all` triggered by cancellation/payment failure), app must:
  - disconnect active tunnel
  - display inactive subscription state
  - block further connects until entitlement is restored
- Entitlement checks should run on launch/resume, pre-connect, and after validation failures.

## Integration caveat to fix
FastAPI reads `/{email}/{device}.conf` using the raw incoming `device`, while `add_device.sh` writes sanitized/lowercased filenames. Unsanitized client device strings can cause false `500 Config not found` responses even if shell provisioning succeeded.



## Activation link mismatch caveat
Product requirement says app activation should support `bbvpn://activate?t=<token>` as primary.
Current worker email templates generate `https://bb-vpn.com/activate.html?t=<token>`.

To align behavior, choose one:
1. Keep web link and use universal links/app links to hand off token into app.
2. Change email link generation to app deep link for mobile-targeted emails.
3. Keep both (web fallback + app deep link button) in email template.

## Entitlement validation contract (required app behavior)
Current requirement is to validate entitlement with backend at:
- app launch
- app resume
- pre-connect

If backend indicates revoked/canceled/failed-payment, app must:
1. disconnect active tunnel
2. mark account inactive
3. block further connects until re-activation

> Note: this shared `src/index.js` still does **not** expose a dedicated entitlement status endpoint. A concrete implementation plan is in `mobile/contracts/entitlement-endpoint.md` (`POST /api/entitlement-check`).
