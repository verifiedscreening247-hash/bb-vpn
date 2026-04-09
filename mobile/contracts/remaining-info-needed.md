# Handoff status: decisions finalized + remaining execution tasks

This document tracks what is now decided and what still needs implementation/verification before production app rollout.

## Finalized decisions (from current handoff)

### Region endpoint + key inventory
- Region key + hostname + public IP inventory is now captured in `mobile/contracts/region-key-inventory.md`.
- Canonical source of truth remains:
  1) live server (`wg show wg0 public-key`)
  2) config-generation output returned to activation clients
- DNS remains standardized to `1.1.1.1` across EU/US/SG unless intentionally overridden.

### Deep-link and token UX
- Canonical deep link: `bbvpn://activate?t=<token>`.
- If universal links are introduced later, they must resolve into the **same token flow**.
- UX decision is final:
  - deep link is primary activation path
  - manual token paste remains available as fallback

### Account lifecycle behavior
- After server-side `revoke_all` (cancellation/failed payment), app should:
  - mark account inactive (subscription ended/payment failed)
  - disconnect active tunnel
  - block reconnect until user re-subscribes/re-provisions
- Entitlement checks should run:
  - on launch/resume
  - before connect
  - when existing profile/token validation fails
- Server remains source of truth for entitlement.

### Firewall consistency target
- Standardize all regions to:
  - default `FORWARD` policy = `DROP`
  - explicit allow rules for wg0 <-> uplink traffic
  - `ESTABLISHED,RELATED` accepts
  - NAT masquerade for each WireGuard subnet

### US route anomaly
- Confirmed in live US `wg showconf wg0`: a peer currently has `AllowedIPs = 10.8.0.2/32` (EU subnet).
- Treat this as active drift/stale state and remove after verifying no intentional cross-region design.

## Remaining execution tasks
1. Keep `mobile/contracts/region-key-inventory.md` current whenever host/IP/key changes.
2. Implement deep-link handling in apps with manual token fallback UX.
3. Implement entitlement polling/check strategy at launch/resume + pre-connect.
4. Apply firewall standardization and verify parity across EU/US/SG.
5. Resolve US route anomaly and document root cause.
6. Fix backend device filename mismatch risk (`/provision` raw device vs sanitized shell filename).

## Confirmed Cloudflare KV namespaces
- `TOKENS` binding uses namespace `bbvpn_tokens` (`e448458258e941cc902dabb27e21f74a`).
- `CUSTOMERS` binding uses namespace `bbvpn_customers` (`ba3f248bcedf497997be5d5c187d6b1a`).

## Newly confirmed from worker source
- Worker currently has no dedicated entitlement status endpoint.
- `POST /api/activate` remains active (single-region flow), while `/api/activate-all` is the multi-region path.
- `VPN_API_BASE` appears unused in current worker code.

## What else is needed now
Given the confirmed Cloudflare secrets **and** `wrangler.toml`, the remaining high-value items are:

Wrangler route confirmed: `bb-vpn.com/api/*` -> worker `bbvpn-provision` (`main=src/index.js`, `compatibility_date=2026-03-13`).

1. Entitlement endpoint: **spec done**, implementation/deploy still pending (`mobile/contracts/entitlement-endpoint.md`).
2. `PROVISION_TOKEN` rotation runbook: **now documented** in `mobile/runbooks/provision-token-rotation.md`; adopt in ops source-of-truth repo/wiki.
3. `VPN_API_BASE` cleanup decision: pending full repo/deploy-config search confirmation.
4. Activation link format alignment decision (`https://bb-vpn.com/activate.html?t=...` vs `bbvpn://activate?t=...` path into app).


## Kid-simple checklist (what to get + where)
You already have: worker route + TOKENS/CUSTOMERS bindings in Cloudflare.

Still needed:
1. **Entitlement check endpoint contract**
   - Status: spec provided in `mobile/contracts/entitlement-endpoint.md`, pending implementation/deploy.
   - Where to put it: Worker `src/index.js` (public app-facing endpoint).
2. **PROVISION_TOKEN rotation runbook**
   - What: step-by-step doc for changing the token without breaking service.
   - Where to get it: ops/deploy docs (or create one in your infra repo / team runbooks).
3. **VPN_API_BASE cleanup decision**
   - What: confirm if this secret can be deleted safely.
   - Where to get it: search code/deploy configs for `VPN_API_BASE`; if unused everywhere, remove from Cloudflare secrets.

Cloudflare pages you already used correctly:
- Worker overview/bindings page: confirms route + KV bindings.
- Worker variables/secrets page: confirms secret names exist.


## Where to find #2 (token rotation runbook)
Primary runbook is now documented at `mobile/runbooks/provision-token-rotation.md`.
Mirror this into your ops source of truth (infra repo/wiki) if needed.


### Current observed location of `PROVISION_TOKEN`
- Regional API service file: `/etc/systemd/system/bbvpn-api.service`
- Loaded by `bbvpn_api.py` via `os.environ.get("PROVISION_TOKEN")`

### Safer rotation checklist (systemd + worker)
1. Generate a new token value.
2. Update `PROVISION_TOKEN` in all regional `bbvpn-api.service` files (EU/US/SG).
3. `sudo systemctl daemon-reload && sudo systemctl restart bbvpn-api` on each region.
4. Update Worker `PROVISION_TOKEN` secret in Cloudflare.
5. Deploy Worker, then smoke test `/api/request-new-link` and `/api/activate-all`.
6. Remove old token from notes/logs and record rotation timestamp/operator.

## #3 cleanup steps for `VPN_API_BASE`
You can clean it now with this safe order:
1. Search code/configs for `VPN_API_BASE` references (worker repo + deploy scripts).
2. Confirm no runtime reads remain (current shared `src/index.js` has none).
3. Delete `VPN_API_BASE` from Cloudflare Worker secrets.
4. Deploy Worker and run smoke tests (`/api/request-new-link`, `/api/activate-all`).

Note: latest shared worker code confirms emailed activation links are HTTPS web links today.

URGENT: if a real token value was shared in chat/screenshots, rotate it immediately before further testing.


## VPN_API_BASE cleanup verification checklist
Before deleting `VPN_API_BASE` from Cloudflare secrets, run a full search in:
- Worker repo source files
- deployment config files (`wrangler.toml`, CI/CD env templates)
- infra scripts / provisioning scripts

If search is empty everywhere, remove the secret and run smoke tests on:
- `POST /api/request-new-link`
- `POST /api/activate-all`
