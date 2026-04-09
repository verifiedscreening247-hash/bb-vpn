# Region WireGuard inventory (current snapshot)

Captured from live servers via:
- `hostname -f || hostname`
- `curl -4 ifconfig.me`
- `wg show wg0 public-key`

| Region | Hostname | Public IP | Server public key |
|---|---|---|---|
| EU | `bbvpn-eu1` | `45.76.39.110` | `fQ5RvDPDZ0WISZWPOkqJnEug/vcnussHp0Kh3FHbTlU=` |
| US | `bbvpn-us1` | `45.63.7.99` | `TFEIqMKIifROGMJ6vEmb7s63p+o+ZUwJCmWz65jztHw=` |
| SG | `bbvpn-sg1` | `45.76.144.180` | `hYmRXvY40msAcN59FoQiSjuK3Mclo7E2KX2IOnFDtTY=` |

## Validation policy
- Live server output is source of truth.
- Activation/config-generation output must match this table by region.
- If a key or endpoint changes, update this file before app rollout.
