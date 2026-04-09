# iOS Plan (Swift)

## Stack
- SwiftUI app shell
- WireGuardKit (official wireguard-apple project)
- Network Extension target for tunnel operations

## MVP tasks
0. Add `mobile/assets/logo.png` into app resources and use in splash/welcome/settings header.
1. Deep link handler for `bbvpn://activate?t=<token>` plus manual token paste screen fallback.
2. API client for `/api/request-new-link` and `/api/activate-all`.
3. Entitlement validation flow at app launch, app resume, and pre-connect checkpoint.
4. On invalid entitlement: disconnect tunnel, mark account inactive, block connect until re-activation.
5. Parse `.conf` text into WireGuard tunnel model and persist per-region profiles.
6. Connect/disconnect toggle with live state UI and region picker.
