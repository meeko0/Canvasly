# Canvasly — real-time collaborative whiteboard on AWS

Multiple users draw, add shapes, sticky notes and text on the same infinite
canvas at the same time, see each other's live cursors, and have boards saved
to their account. Simultaneous edits merge without conflicts via CRDTs (Yjs).

> **Status:** full project scaffold with a working **vertical slice** —
> auth → WebSocket connect → join board → draw → Yjs sync → live cursors →
> S3 snapshot. Remaining features are stubbed with clear `TODO` markers.

## Monorepo layout

```
canvasly/
  shared/      TypeScript types + the WebSocket message protocol (source of truth)
  infra/       AWS CDK (TypeScript) — Cognito, DynamoDB, S3, API GW WebSocket, Lambdas
  backend/     Node.js Lambda handlers for the WebSocket routes
  frontend/    React + TypeScript + Konva + Yjs client
  scripts/     load test (20+ concurrent users), helpers
  docs/        architecture diagram, design notes
```

## Architecture

```
                           ┌─────────────────────────────┐
         Google / email    │        Amazon Cognito       │
            sign-in  ─────▶ │  (JWT issued to the client) │
                           └──────────────┬──────────────┘
                                          │ JWT in $connect query
   ┌──────────────┐   WebSocket  ┌────────▼─────────┐   invoke   ┌──────────────┐
   │ React client │◀────────────▶│ API GW WebSocket │──────────▶ │   Lambdas    │
   │ Konva + Yjs  │   messages   │   ($connect ...) │            │ (per route)  │
   └──────┬───────┘              └──────────────────┘            └──────┬───────┘
          │ Amplify Hosting                                             │
          │                                   ┌─────────────────────────┼───────────┐
          │                                   ▼            ▼            ▼            ▼
          │                             DynamoDB      DynamoDB      DynamoDB        S3
          │                            Connections     Boards      Snapshots    snapshots/
          │                                                         + Members   images/exports
          └── board snapshots (compressed Yjs state) fetched via presigned S3 URLs
```

Full-resolution diagram: `docs/architecture.md`.

### WebSocket routes
`$connect` `$disconnect` `joinBoard` `leaveBoard` `syncUpdate` `cursorMove`
`presence` `ping`. Protocol defined in `shared/src/protocol.ts`.

## Setup

```bash
# 1. install everything
cd canvasly
npm install            # root workspaces: shared, infra, backend, frontend

# 2. build shared types (everything imports these)
npm run build -w shared

# 3. frontend dev server
npm run dev -w frontend

# 4. backend unit tests
npm test -w backend
```

## Deploy (you run this — it touches your AWS account)

> This repo ships deploy-ready CDK. It does **not** deploy for you: deploying
> creates billable resources in *your* account. Review, then run it yourself.

```bash
cd infra
npm run build
npx cdk bootstrap            # once per account/region
npx cdk deploy -c env=dev    # dev stack
npx cdk deploy -c env=prod   # prod stack (separate stack, separate tables)
```

Outputs (WebSocket URL, Cognito pool/client ids, buckets) are printed and
written to `infra/cdk-outputs.json`; feed them to the frontend via
`frontend/.env` (see `frontend/.env.example`).

## Design decisions

**Why CRDTs (Yjs) over last-write-wins.** Two users dragging the same sticky
or typing in the same text box at the same time must both keep their intent.
Last-write-wins silently discards one edit. Yjs merges concurrent edits
deterministically on every client, so everyone converges to the same board
without a central lock or a server round-trip to resolve each change. Offline
edits merge the same way on reconnect.

**Why snapshots + an update log.** Replaying every edit since a board was
created would make `joinBoard` slow and expensive as boards age. Instead the
server appends Yjs updates and periodically compacts them into one snapshot in
S3 (every ~50 updates or 30s idle). A joining client gets `latest snapshot +
updates since that snapshot` — bounded work regardless of board age. Snapshots
also power version history and restore.

**How presence avoids the database.** Live cursors and selection highlights
change ~20×/sec per user. Writing those to DynamoDB would be costly and
pointless — they are ephemeral. `cursorMove` and selection are broadcast only,
never persisted. Only durable facts (who is connected, to which board) live in
the Connections table, with a TTL so stale rows self-clean.

## Estimated monthly AWS cost

Rough order-of-magnitude for a small team (≈50 daily active users, a few
boards each, moderate editing). Region us-east-1, on-demand.

| Service | Driver | Est. / month |
|---|---|---|
| API GW WebSocket | messages + connection-minutes | $3–8 |
| Lambda | per-message invocations, 128–256 MB | $2–5 |
| DynamoDB | on-demand, small item sizes | $2–5 |
| S3 | snapshots + images + thumbnails | $1–3 |
| CloudWatch | logs + metrics + alarms | $2–5 |
| Cognito | < 50k MAU = free tier | $0 |
| Amplify Hosting | build + CDN for a small SPA | $1–3 |
| **Total** | | **≈ $12–30 / month** |

Costs scale with active connection-minutes and message volume, not stored
boards. A 20-user load test for a few hours is cents. Validate with
`scripts/load-test.ts` + CloudWatch before trusting any projection.

## Testing

- `npm test -w backend` — Lambda handler + permission-check unit tests.
- `npm run load-test -w scripts` — simulate 20+ concurrent users on one board,
  reports p50/p95/p99 message latency.

## Environments

`dev` and `prod` are separate CDK stacks (`-c env=...`) with separate tables,
buckets and Cognito pools. Nothing is shared between them.
