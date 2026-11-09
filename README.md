# Child Shield backend (TypeScript)

Receives pseudonymous event metadata from devices, escalates suspicious chat snippets to an LLM
under a per-child cost cap, and routes alerts with risk-based triage. No images, no stored chat text.

```
Device (Android/Win/Mac)               API (Hono, serverless)                Queue            Worker
 on-device OCR/text + classifier  -->  POST /v1/events                  -->  SQS     -->     processClassifyJob
 (sends snippet only if suspicious)    - auth (HMAC token, role)                              - LLM budget check (per child/day)
                                       - strict schema (rejects images)                       - Claude Haiku 5.5 classify
                                       - idempotency (eventId)                                - triage -> alert (metadata only)
                                       - conversation risk score (decay)                      - notify guardian / safeguarding lead
 <-- GET /v1/config (ETag)             - enqueue when score >= threshold
School dashboard / guardian app  -->   GET /v1/admin/alerts, PATCH /v1/admin/alerts/:id, GET /v1/admin/stats
```

## Run
```
npm install
npm run typecheck
npm run smoke                                   # 8 checks, no network needed
AUTH_SECRET=$(openssl rand -hex 32) npm run dev # uses HeuristicClassifier unless ANTHROPIC_API_KEY is set
```

## Layout
- `src/domain.ts` event schema (zod, `.strict()`), types
- `src/auth.ts` HMAC tokens (Web Crypto), role middleware; tenantId always comes from the token
- `src/repos.ts` storage interface + in-memory impl. `migrations/001_init.sql` = Postgres schema with RLS
- `src/services/risk.ts` decaying per-conversation score
- `src/services/classifier.ts` Anthropic classifier (`claude-haiku-5-5`) + local stub
- `src/services/triage.ts` severity + who gets notified
- `src/services/worker.ts` queue consumer, daily LLM cap, rules fallback, cost tracking
- `src/routes/*` ingest, remote config, admin (alerts, review, stats)

## Not built yet (decide before production)
- Postgres `Repos` implementation, SQS `Queue` implementation, real notifier (FCM/APNs/email)
- Enrollment and consent flow (issuing device/guardian tokens), token rotation/revocation
- Rate limiting at the gateway, WAF, secrets manager, audit log of who viewed which alert
- Retention policy and PDPA review: what is stored (metadata only today), how long, who can see it.
  Queue messages hold chat snippets: use a short retention + SSE encryption
- Whether guardians/lead should see any excerpt of the chat (none today). Ask your lawyer
- Evaluate the classifier on real Thai data (precision/recall) before trusting the severity thresholds

## Lambda adapter sketch
```ts
import { handle } from "hono/aws-lambda";
export const handler = handle(createApp({ repos: new PgRepos(pool), queue: new SqsQueue(url), authSecret }));
// separate Lambda subscribed to SQS calls processClassifyJob(deps, JSON.parse(record.body))
```
