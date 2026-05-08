# Robodor Platform

End-to-end IoT door-automation platform: native iOS + Android mobile apps, AWS-native backend, and a web admin console.

> This repository is a **public showcase**. The actual product source (BLE protocol, Modbus register schemas, infrastructure IDs, hardware-specific tunings) is kept private. The code samples here are reusable infrastructure patterns I built during the project — none of them carry any of the proprietary product detail.
>
> The private product repositories contain the real commit history, release tags
> and implementation. This public repo is a sanitized evidence pack: architecture
> notes, selected reusable snippets, security decisions and platform highlights.
> It is not a source mirror.

---

## What it is

Robodor sells industrial door controllers. Operators on site control them locally; tenants and service teams manage them remotely. The platform that ships with the product covers:

- **iOS app** (SwiftUI) — local BLE control, settings configuration, telemetry, alerts, charts.
- **Android app** (Jetpack Compose) — feature-equivalent on Android, the original "golden contract" for device behaviour.
- **AWS backend** (CDK + Lambda + Cognito + DynamoDB + S3) — accounts, organizations, roles, telemetry sync, remote command queue, encrypted backups.
- **Web admin console** (React + Vite) — fleet view, per-device telemetry timeline, remote operations.

My work spans the iOS app, the V2 cross-platform UI redesign, the AWS backend
stack, and the shared design-system direction. Source code and hardware-tuned
details stay private; the docs here describe the engineering shape.

## Public evidence

This public repo proves the work without exposing the product:

- [`docs/architecture.md`](docs/architecture.md) — local-first mobile control,
  cloud sync, tenant isolation, command idempotency, encrypted backups.
- [`docs/ios-highlights.md`](docs/ios-highlights.md) — SwiftUI concurrency,
  Keychain session storage, telemetry persistence, Live Activities.
- [`docs/android-highlights.md`](docs/android-highlights.md) — Compose,
  StateFlow, foreground BLE service, settings schema, backup flow.
- [`docs/server-highlights.md`](docs/server-highlights.md) — Cognito JWT,
  DynamoDB tenant scope, S3/KMS, API throttling and logging hygiene.
- [`docs/design-system.md`](docs/design-system.md) — shared iOS/Android token
  model, component vocabulary, accessibility and idle-aware animation.
- [`snippets/`](snippets/) — small, generic patterns extracted from the work.

Screenshots are intentionally gated behind redaction. Anything showing real
devices, BLE identifiers, tenant names, operators, command logs, cloud URLs, or
admin data should not be published casually.

---

## Tech stack

### iOS
- Swift 6, SwiftUI, Swift Concurrency (async/await, `@MainActor`, `CheckedContinuation`)
- Combine for repository → view streaming
- Swift Charts for telemetry visualisation
- CoreBluetooth + custom Modbus-over-BLE transport
- Keychain Services, ASWebAuthenticationSession (Google Sign-In)
- Live Activities, App Shortcuts, URL scheme deep-linking
- Swift Package Manager
- 5-language localization (TR / EN / DE / FR / IT)

### Android
- Kotlin, Jetpack Compose, Material 3
- Hilt + KSP for DI
- Coroutines + StateFlow
- Custom BLE transport over Modbus RTU
- Foreground service for resilient device sessions
- 5-language string resources

### Backend
- AWS CDK (TypeScript) for the entire stack
- AWS Lambda (Node.js 22)
- Amazon Cognito (User Pools, federated Google Sign-In, JWT verification)
- Amazon DynamoDB (single-table design with GSI)
- Amazon S3 (cold telemetry, encrypted user backups)
- AWS KMS (customer-managed key, envelope encryption)
- API Gateway HTTP API with throttling
- CloudFront + S3 static hosting for the web console

### Tooling
- Xcode + xcodebuild
- Gradle, Android Studio
- ESBuild (Lambda bundling), tsc
- GitHub Actions (CI hooks)

---

## Architecture at a glance

```
[ iOS App ]          [ Android App ]           [ Web Admin ]
       \\                  /                      /
        \\— BLE+Modbus —// (local control path)  /
         \\              /                      /
          \\____________/______________________/
                 │
                 │ HTTPS (Bearer JWT)
                 ▼
       ┌──────────────────────────┐
       │   API Gateway HTTP API   │
       │   (throttled, public)    │
       └────────────┬─────────────┘
                    │
              ┌─────▼──────┐
              │   Lambda   │   Node.js 22
              │  (TS)      │   single fn, fluid compute
              └──┬──┬──┬──┬┘
                 │  │  │  │
        ┌────────┘  │  │  └─────────┐
        ▼           ▼  ▼            ▼
   ┌────────┐  ┌────────┐  ┌────────────┐
   │Cognito │  │DynamoDB│  │     S3     │
   │  IDP   │  │  (GSI) │  │  (KMS-CMK) │
   └────────┘  └────────┘  └────────────┘
```

Local control (BLE) and cloud sync are intentionally decoupled: the device is fully usable when offline, and the server never holds a live socket to the hardware.

For more, see [`docs/architecture.md`](./docs/architecture.md).

---

## Highlights

### Mobile

- **Single-source architecture** — both iOS and Android funnel UI through one repository (`Repository` pattern) that owns BLE state, register cache, polling cadence, and metrics. Views observe; they never reach into the transport.
- **Manual mode safety path** — separate write path for live open/close/stop with shorter timeouts, immediate state drop on timeout, and best-effort STOP write on disconnect.
- **Background session resilience** — declared `bluetooth-central` background mode + scene phase coordination so a screen-lock doesn't kill the active connection.
- **Telemetry history with downsampled buffers** — session / 1h / 24h / 7d rolling buffers in memory, persisted to disk under Application Support with detached writes so the main actor stays unblocked.
- **Design system** — `RobodorPalette`, `RobodorTypography`, `RobodorRadii`, `RobodorShadows` tokens, an `R*` component library, dark + light themes shared across both platforms.
- **5-language localization** — full parity in TR / EN / DE / FR / IT, including dynamic content like door state labels.

### Backend

- **JWT-based auth** with Cognito access tokens, verified server-side via `aws-jwt-verify` + JWKS caching.
- **Server-controlled user profile** — `role` and `organizationId` live in DynamoDB so tokens cannot be used to self-promote into other tenants.
- **Idempotent commands** — clients can supply an `idempotencyKey` and the server uses a DynamoDB `ConditionExpression` to dedupe retries without race conditions.
- **GSI-indexed tenant queries** — card lookups by organization use a partition Query rather than a Scan.
- **Encrypted backups** — settings backups are AES-GCM encrypted on the device, the ciphertext travels through S3, and decryption only ever happens on the user's own device.
- **Hardened Cognito policy** — minimum 12-character passwords with full character-class requirements.
- **API Gateway throttling** — default route rate / burst limits as a baseline DDoS guard.

### Operational

- **Reproducible infrastructure** — the entire stack synthesises from CDK and deploys as a single `cdk deploy`. No manual console clicks.
- **No personal AWS dependency** — IDs and account references are externalised so the same stack drops onto a company-owned account.

---

## Repository layout

```
docs/                Architecture, design system, platform-specific notes
snippets/            Generic, reusable infrastructure patterns I authored
screenshots/         Redaction checklist for release-approved images
```

The actual product source — BLE/Modbus transport, register catalogue, settings schema, hardware tunings, AWS resource IDs — is **not in this repo** and never will be. What you see here is purely the patterns and infrastructure-side work, sanitised for public viewing.

---

## License

All rights reserved. The samples in this repository are published for portfolio / illustration purposes only and are not licensed for use, redistribution, or modification without prior written consent.
