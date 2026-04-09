# PROVISION_TOKEN rotation runbook

Use this runbook to rotate `PROVISION_TOKEN` without breaking provisioning.

## Where token is stored
- Cloudflare Worker secret: `PROVISION_TOKEN`
- Regional API servers: systemd unit environment in `/etc/systemd/system/bbvpn-api.service`
- Regional FastAPI process reads env via `os.environ.get("PROVISION_TOKEN")`

## Safe rotation order
1. Generate a new token (64+ hex chars recommended).
2. Update **regional APIs first** (EU/US/SG):
   - edit `/etc/systemd/system/bbvpn-api.service`
   - set new `Environment=PROVISION_TOKEN=...`
   - run: `sudo systemctl daemon-reload && sudo systemctl restart bbvpn-api`
3. Validate each regional API directly (`/device_count` or `/provision`) with the new token.
4. Update Cloudflare Worker secret `PROVISION_TOKEN` to the new value.
5. Deploy/publish Worker.
6. Validate end-to-end through worker:
   - `POST /api/request-new-link`
   - `POST /api/activate-all`
7. Record rotation metadata (who/when/why) in ops log.

## Validation checklist
- Regional APIs reject old token and accept new token.
- Worker calls to regional `/provision` and `/revoke_all` succeed with new token.
- Activation flow and new-link flow succeed from public app-facing endpoints.

## Rollback
If failures occur after Worker deploy:
1. Revert Worker secret to previous token and redeploy Worker.
2. If needed, revert regional API units to previous token and restart services.
3. Re-run validation checklist.

## Security note
If token appears in chat/logs/screenshots, treat it as compromised and rotate immediately.
