# BB VPN App Suite (iOS + Android + Windows)

This is the starter blueprint for building three first-party apps on top of the existing BB VPN backend and WireGuard architecture.

## Targets
- iOS app (Swift + WireGuardKit)
- Android app (Kotlin + `com.wireguard.android:tunnel`)
- Windows desktop app (C# + embeddable WireGuard Windows service)

## Product Principles
1. Reuse existing backend activation flow (`/api/request-new-link` + `/api/activate-all`).
2. Preserve one-time activation token behavior.
3. Match BB visual identity from the website (neon pink/gold/cyan, sticker-style UI, bold typography).
4. Keep connect/disconnect one-tap and obvious.

## Shared UX Flow
1. Welcome screen with BB branding.
2. Enter purchase email -> request new device link.
3. Open activation link and extract token (`t` query param).
4. Name device.
5. Fetch all region configs from `/api/activate-all`.
6. Import configs into native WireGuard stack.
7. Show region picker (EU / US / SG) + connect toggle.

## MVP Screen List (all platforms)
- Splash / Welcome
- Request Link
- Enter Activation Token (or deep-link handler)
- Device Naming + Provision
- Tunnel Dashboard (connect/disconnect, region, status)
- Settings (privacy/terms/AUP links, support email)

## Confirmed Region Network Ranges
- EU: `10.8.0.0/24`
- US: `10.9.0.0/24`
- SG: `10.10.0.0/24`

## API and Contracts
See `mobile/contracts/api.md`.

## Region Keys
See `mobile/contracts/region-key-inventory.md` for current per-region hostname, public IP, and WireGuard server public key.

## Activation and Entitlement (required behavior)
- Support deep-link activation via `bbvpn://activate?t=<token>`.
- Support manual token paste on the activation screen as fallback.
- Validate entitlement with backend at **launch**, **resume**, and **pre-connect** checkpoints.
- If entitlement is revoked/canceled/failed-payment: disconnect active tunnel, mark account inactive, and block connect until re-activation.

## Styling
See `mobile/theme/bb-theme.md` and `mobile/theme/tokens.json`.
Use `mobile/assets/logo.png` as the primary app logo in UI screens.
If the image is not present yet, see `mobile/assets/README.md` placeholder instructions.

## Platform Notes
- iOS: `mobile/platforms/ios.md`
- Android: `mobile/platforms/android.md`
- Windows: `mobile/platforms/windows.md`

## Next Build Steps
1. Implement deep-link parsing for activation links.
2. Build API client wrapper with strict error typing.
3. Create tunnel manager abstraction with platform-specific backends.
4. Build MVP screens with shared visual language.

## Backend Readiness Checklist
See `mobile/contracts/remaining-info-needed.md` for the remaining backend/ops decisions before production app work.
See the "Kid-simple checklist" there for exactly what to collect next and where to collect it.
Token rotation procedure is documented in `mobile/runbooks/provision-token-rotation.md`.

## Worker secret/binding status
See `mobile/contracts/api.md` (Worker runtime secrets section) for currently confirmed Cloudflare secret names and the remaining binding details still needed.
Wrangler route is confirmed as `bb-vpn.com/api/*` for worker `bbvpn-provision`.
KV bindings for `TOKENS` and `CUSTOMERS` are now confirmed in the contracts docs.
Entitlement endpoint implementation spec is now provided in `mobile/contracts/entitlement-endpoint.md`; deploy it in Worker code next.
Rotate `PROVISION_TOKEN` immediately if any real token was exposed in chat/logs/screenshots.
Worker emails currently send the HTTPS activate link (`https://bb-vpn.com/activate.html?t=...`), so app handoff must use universal links or template updates.
