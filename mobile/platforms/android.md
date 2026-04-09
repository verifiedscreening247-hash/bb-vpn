# Android Plan (Kotlin)

## Stack
- Kotlin + Jetpack Compose
- `com.wireguard.android:tunnel`

## MVP tasks
0. Add `mobile/assets/logo.png` into app resources and use in splash/welcome/settings header.
1. Intent/deep-link handling for `bbvpn://activate?t=<token>` plus manual token paste fallback UI.
2. API client (Ktor/OkHttp + kotlinx.serialization) for request-link + activate-all endpoints.
3. Entitlement validation flow at app launch, app resume, and pre-connect checkpoint.
4. On invalid entitlement: disconnect tunnel, mark account inactive, block connect until re-activation.
5. Convert config text into WireGuard tunnel objects and persist metadata (Room/DataStore).
6. Dashboard with region picker + connect state.
