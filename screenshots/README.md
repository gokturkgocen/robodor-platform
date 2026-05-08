# Screenshots

No screenshots are committed by default. Publish only real, release-approved
mobile, web-admin or fleet captures after redaction; do not use mockups or
generated visuals. The app can expose real devices, BLE identifiers, tenant
names, operator names, command logs, cloud URLs and support contact data.

When release-approved screenshots are available, use this set:

- `dashboard.png` — main dashboard with door visualisation + KPI tiles
- `charts.png` — telemetry charts screen
- `settings.png` — settings menu list
- `alerts.png` — alert centre
- `more.png` — more / profile
- `login.png` — login screen
- `web-admin.png` — web admin console, only with demo tenant/device data

## Redaction checklist

- No real device serials, BLE MAC addresses, UUIDs or gateway hostnames.
- No real customer, tenant, facility, operator or support-person names.
- No API Gateway, Cognito, S3, CloudFront or internal admin URLs.
- No command/audit rows that show production timings or operator actions.
- Use demo data for screenshots whenever possible.

Use native-resolution mobile captures for app screens and wide desktop captures
(1920 × 1080 or larger) for web-admin surfaces. Compress before committing so
the README stays fast on GitHub.
