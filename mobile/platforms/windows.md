# Windows Plan (C#)

## Stack
- .NET (WPF or WinUI 3)
- WireGuard Windows embeddable DLL/service integration

## MVP tasks
0. Add `mobile/assets/logo.png` into app resources and use in splash/welcome/settings header.
1. Activation flow supporting `bbvpn://activate?t=<token>` and manual token paste fallback.
2. API client for app-facing endpoints.
3. Entitlement validation at app launch, app resume, and pre-connect checkpoint.
4. On invalid entitlement: disconnect tunnel, mark account inactive, block connect until re-activation.
5. Write per-region `.conf` to secure app data directory and hook connect/disconnect to service.
6. Show tunnel state and simple diagnostics.
