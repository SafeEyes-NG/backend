# SafeEyes NG: Backend

> API, verification workflow, sightings, geofenced alerts, messaging gateway and evidence anchoring for SafeEyes NG.

Part of the [SafeEyes NG](https://github.com/safeeyes-ng/docs) project: an open-source, verified crowd-sighting and alert network for kidnapping cases in Nigeria. Read the [docs repo](https://github.com/safeeyes-ng/docs) first for the mission, threat model and design principles.

> **Status:** Phase 0 / early scaffolding. Everything below describes the intended design. Update it as things are built.

---

## What this service does

| Module | Responsibility |
|---|---|
| `cases` | Create cases, run the verification workflow (two-person rule for high-impact alerts), expire and close cases |
| `sightings` | Receive photos, video, voice notes, text and location pins. Cluster them and score confidence |
| `alerts` | Compute geofences and fan out minimal, auto-expiring alerts |
| `messaging` | SMS/USSD, Telegram bot and WhatsApp Business gateway adapters |
| `auth` | Accounts, roles (reporter, verifier, responder, admin), anonymous reporting tokens |
| `media` | Encrypted upload, storage, derived thumbnails, optional face and plate blurring |
| `anchoring` | Hash evidence, anchor hash and timestamp on Stellar, verify anchors |
| `audit` | Append-only audit log of every sensitive action |

## Tech stack (proposed defaults, open to discussion via ADR)

- **Language:** TypeScript on Node.js LTS
- **Framework:** Fastify (or NestJS, to be decided in an ADR)
- **Database:** PostgreSQL with **PostGIS** for geospatial queries
- **Queue / cache:** Redis (BullMQ for jobs)
- **Object storage:** S3-compatible (MinIO locally)
- **Validation and types:** Zod, with client types generated from `openapi.yaml` in the docs repo
- **Stellar:** official Stellar JS SDK, Soroban RPC
- **Messaging providers:** SMS/USSD via a provider with Nigerian coverage (e.g. Africa's Talking), Telegram Bot API, WhatsApp Business API. Providers sit behind adapters so they can be swapped
- **Testing:** Vitest, Testcontainers for integration tests
- **Tooling:** ESLint, Prettier, Husky, commitlint

## Repository layout

```
backend/
├── src/
│   ├── cases/
│   ├── sightings/
│   ├── alerts/
│   ├── messaging/
│   │   ├── sms/
│   │   ├── telegram/
│   │   └── whatsapp/
│   ├── auth/
│   ├── media/
│   ├── anchoring/
│   ├── audit/
│   ├── common/            # config, logging, errors, shared utils
│   └── server.ts
├── migrations/            # SQL migrations (PostGIS enabled)
├── test/
│   ├── unit/
│   ├── integration/
│   └── fixtures/          # synthetic data only
├── infra/
│   ├── docker-compose.yml # postgres+postgis, redis, minio
│   └── Dockerfile
├── scripts/
├── .env.example
├── CLAUDE.md
├── CONTRIBUTING.md        # links to docs repo; backend-specific notes here
└── README.md
```

## Getting started

### Prerequisites
- Node.js (current LTS) and npm or pnpm
- Docker and Docker Compose
- A GitHub Codespace works out of the box once a `.devcontainer` is added (see below)

### Setup
```bash
git clone https://github.com/safeeyes-ng/backend.git
cd backend
cp .env.example .env          # fill in local-only values
docker compose -f infra/docker-compose.yml up -d
npm install
npm run migrate
npm run dev
```

### Common scripts (define these in `package.json`)
| Script | Purpose |
|---|---|
| `npm run dev` | Start the API with reload |
| `npm run build` | Compile TypeScript |
| `npm test` | Run unit tests |
| `npm run test:integration` | Run integration tests against containers |
| `npm run lint` | Lint and format check |
| `npm run migrate` | Apply database migrations |
| `npm run gen:types` | Generate API types from `openapi.yaml` |

## Configuration

All config is via environment variables. Never commit `.env`.

| Variable | Description |
|---|---|
| `DATABASE_URL` | Postgres connection string (PostGIS enabled) |
| `REDIS_URL` | Redis connection string |
| `S3_ENDPOINT`, `S3_BUCKET`, `S3_ACCESS_KEY`, `S3_SECRET_KEY` | Object storage |
| `MEDIA_ENCRYPTION_KEY` | Key management approach to be decided in an ADR; use a local dev key only |
| `STELLAR_NETWORK` | `testnet` for development, `mainnet` only after audit |
| `STELLAR_ANCHOR_SECRET` | **Testnet keys only** in dev. Production keys live in a secrets manager, never in the repo |
| `SOROBAN_RPC_URL` | Soroban RPC endpoint |
| `SMS_PROVIDER_*`, `TELEGRAM_BOT_TOKEN`, `WHATSAPP_*` | Messaging credentials |

## API

The API contract lives in the docs repo at `api/openapi.yaml`. **Spec first:** change the spec, then implement. Both backend and frontend build against it, and types are generated rather than hand-written.

Initial endpoints to implement:

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/v1/cases` | Open a case (family, leader, NGO) |
| `POST` | `/v1/cases/{id}/verify` | Verifier approves or rejects |
| `GET` | `/v1/cases/{id}` | Case details (role-gated) |
| `POST` | `/v1/cases/{id}/sightings` | Submit a sighting with media |
| `GET` | `/v1/cases/{id}/sightings` | List and cluster sightings (responders only) |
| `POST` | `/v1/cases/{id}/close` | Close a case |
| `GET` | `/v1/evidence/{hash}/anchor` | Verify an on-chain anchor |

## Safety and privacy rules (non-negotiable)

These come from the project threat model. PRs that break them will not be merged.

1. **No public alert without verification.** Enforce in code, not by convention.
2. **Public alerts are minimal and expire.** No suspect photos or names in any public channel.
3. **Role-based access everywhere.** Full case data is visible only to verified responders, the family and verifiers.
4. **Consent-only live location.** Never track a person who has not opted in, and stop immediately on opt-out or case close.
5. **Data minimisation.** Strip unneeded metadata (such as EXIF) on upload. Short, configurable retention. Support deletion on request where law and case needs allow.
6. **Anonymous reporting supported.** Do not log information that unmasks anonymous reporters.
7. **Audit everything sensitive.** Verification decisions, access to case data and exports go into the append-only audit log.
8. **No real data in the repo.** Fixtures, tests and docs use synthetic data only. Secret scanning is on.
9. **On-chain data is hashes and timestamps only.** Never personal data, never file contents.

## Evidence anchoring flow

1. The client hashes the file on-device (SHA-256) and sends the hash with the upload.
2. The backend re-hashes the stored file and checks the hashes match.
3. The `anchoring` module submits the hash and timestamp to the `evidence-anchor` contract (see the `contracts` repo).
4. The transaction ID is stored with the sighting, so anyone with the file can verify it was not altered.

## Testing

- Unit tests for domain logic (verification rules, geofence math, confidence scoring)
- Integration tests with real Postgres/PostGIS and Redis via containers
- Contract-test the Stellar layer against **testnet** only
- Abuse-case tests: spam, replayed submissions, duplicate cases, unauthorised access attempts
- Target: every safety rule above has at least one test that fails if the rule is broken

## Contributing

1. Read the [docs repo CONTRIBUTING.md](https://github.com/safeeyes-ng/docs/blob/main/CONTRIBUTING.md) and the threat model.
2. Pick an issue labelled `good first issue`, `backend` or `help wanted`, and comment to claim it.
3. Branch naming: `feat/…`, `fix/…`, `docs/…`.
4. Use conventional commits (`feat:`, `fix:`, `docs:`).
5. Open a PR with tests. Require passing CI and at least one review.

**Good first issues to open:**
- Set up Fastify skeleton, config loader and health check
- Docker Compose with Postgres+PostGIS, Redis and MinIO
- Migration for `cases`, `sightings` and `audit_log` tables
- `POST /v1/cases` with validation against the OpenAPI spec
- Geofence query: find alert recipients within a radius of a point
- Telegram bot adapter that sends a test alert
- Strip EXIF from uploaded images
- SHA-256 hashing helper with tests

## Working with Claude Code in a Codespace

1. Open the repo in a Codespace. Add a `.devcontainer/devcontainer.json` with Node, Docker-in-Docker and the Postgres client.
2. Install Claude Code (`npm install -g @anthropic-ai/claude-code`, or see Anthropic's docs for the current method) and run `claude` in the repo root.
3. Create a `CLAUDE.md` with:
   - The mission and *eyes, not fists* principle
   - The **Safety and privacy rules** section above, verbatim
   - Stack, folder layout, and the scripts table
   - "Spec first: update `openapi.yaml` before changing endpoints"
   - "Synthetic data only. Never write real names, numbers, locations or secrets"
4. Example first prompts:
   - *"Scaffold the Fastify server with config loading, structured logging and a `/health` route. Add tests."*
   - *"Write the first migration for cases, sightings and audit_log with PostGIS geometry columns."*
   - *"Implement `POST /v1/cases` following the OpenAPI spec, with role checks and an audit log entry."*
5. Review every change. Treat AI output on auth, media handling and anchoring as untrusted until a human has read and tested it.

## Security

Never open a public issue for a vulnerability. See `SECURITY.md` in the docs repo for private reporting.

## License

Apache-2.0 (confirm and add a `LICENSE` file before the first release).
