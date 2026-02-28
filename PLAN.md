# Digital Footprint Dashboard Hackathon MVP Plan (Decision Complete)

## Summary
Build a production-shaped hackathon MVP that verifies email ownership, verifies live human presence, ingests breach intelligence, computes multi-dimensional risk, and shows actionable mitigation impact in a cyber-style dashboard.

This plan is locked to your chosen defaults:
1. Scope priority: `Full feature push`.
2. Verification model: `Hybrid` (OTP required; liveness required for “Verified” and full report, with OTP-only limited fallback).
3. Final score scale: `0–100 normalized`.
4. Retention: `30 days` with user delete-now.
5. Data sources: `HIBP + Gravatar + disposable-domain detection`.
6. Legal engine: `Static templates with variable injection`.
7. Account scope: `Single-user dashboard tied to verified email`.

## Success Criteria
1. User can complete OTP flow and run a scan from a verified inbox.
2. User can complete liveness challenge and unlock full “Verified” dashboard.
3. Dashboard renders all four panels: Risk Overview, Exposure Breakdown, Attack Surface Map, Mitigation Simulator.
4. Risk score updates in real time when mitigation toggles change.
5. GDPR/CCPA templates generate with copy/download.
6. No face images are persisted.
7. Scan endpoint is rate-limited and all sensitive endpoints require auth.

## Scope
### In Scope
1. Next.js 14 app router app with Tailwind + shadcn/ui.
2. Supabase Auth (email OTP), Supabase Postgres, RLS.
3. Client-side MediaPipe liveness check.
4. Server-side integrations for HIBP and Gravatar; local disposable-domain dataset.
5. Risk engine, visualization layer, mitigation simulator.
6. Legal template generation and export (`.txt`).
7. 30-day retention purge job and manual deletion UX.
8. Demo-ready seed/fixture mode for predictable presentation.

### Out of Scope
1. Dark-web crawling, broker scraping, face search.
2. Biometric storage/embeddings.
3. AI-generated legal advice.
4. Enterprise multi-tenant admin features.
5. Additional paid threat intel providers.

## Architecture
1. Frontend: Next.js 14, React Server Components + client components for camera/chart/simulation.
2. Backend: Next.js route handlers for scan orchestration and legal exports.
3. Auth/DB: Supabase Auth + Postgres + RLS.
4. Visualization: Recharts for radar/timeline; `react-force-graph-2d` for attack surface map.
5. Liveness: `@mediapipe/tasks-vision` in browser only.
6. Rate limiting: Upstash Redis + `@upstash/ratelimit`.
7. Validation: `zod` for API payload validation.

## Product Flow Specification
1. Landing page with CTA `Analyze My Digital Exposure`.
2. Email input page validates syntax and blocks obvious invalid patterns.
3. OTP screen sends and verifies Supabase email OTP.
4. Consent screen captures privacy/camera/processing consent.
5. Liveness screen runs random challenge (`blink twice`, `turn left and center`, `turn right and center`).
6. Scan screen calls `/api/scans` and shows step progress (`intake`, `breach lookup`, `modeling`, `render`).
7. Results dashboard shows:
   1. Risk Overview card with score, level, confidence, verified badge.
   2. Exposure Breakdown radar chart.
   3. Exposure timeline + velocity + trend.
   4. Attack surface graph.
   5. Mitigation simulator toggles and recalculated score.
   6. Legal actions panel (GDPR/CCPA copy/download).
8. History page lists scans from last 30 days.
9. Settings page supports `Delete current scan`, `Delete all data`, and retention notice.

## Data Model (Supabase)
1. `users`:
   1. `id uuid pk` (matches Supabase auth UID).
   2. `email text unique not null`.
   3. `created_at timestamptz default now()`.
2. `verification_sessions`:
   1. `id uuid pk`.
   2. `user_id uuid fk users(id)`.
   3. `otp_verified_at timestamptz not null`.
   4. `liveness_status text check in ('passed','failed','skipped')`.
   5. `liveness_challenge text`.
   6. `liveness_metrics jsonb`.
   7. `created_at timestamptz default now()`.
3. `scans`:
   1. `id uuid pk`.
   2. `user_id uuid fk users(id)`.
   3. `verification_session_id uuid fk verification_sessions(id)`.
   4. `report_type text check in ('verified_full','limited_fallback')`.
   5. `risk_score int check (risk_score between 0 and 100)`.
   6. `risk_level text check in ('low','moderate','high')`.
   7. `confidence text check in ('low','medium','high')`.
   8. `takeover_risk int`.
   9. `theft_risk int`.
   10. `phishing_risk int`.
   11. `exposure_risk int`.
   12. `signal_snapshot jsonb`.
   13. `mitigation_baseline jsonb`.
   14. `exposure_velocity numeric`.
   15. `trend text check in ('increasing','stable','declining')`.
   16. `created_at timestamptz default now()`.
   17. `expires_at timestamptz default now() + interval '30 days'`.
4. `breaches`:
   1. `id uuid pk`.
   2. `scan_id uuid fk scans(id) on delete cascade`.
   3. `breach_name text`.
   4. `breach_domain text`.
   5. `breach_date date`.
   6. `pwn_count bigint`.
   7. `data_classes text[]`.
   8. `is_verified boolean`.
   9. `is_sensitive boolean`.
5. `consent_events`:
   1. `id uuid pk`.
   2. `user_id uuid fk users(id)`.
   3. `scan_id uuid null fk scans(id)`.
   4. `event_type text check in ('privacy','terms','camera','processing')`.
   5. `version text`.
   6. `created_at timestamptz default now()`.
6. `legal_exports`:
   1. `id uuid pk`.
   2. `scan_id uuid fk scans(id)`.
   3. `regime text check in ('gdpr','ccpa')`.
   4. `template_version text`.
   5. `created_at timestamptz default now()`.

RLS policy: user can only read/write rows where `user_id = auth.uid()`.

## Public APIs, Interfaces, and Types
### Route Handlers
1. `POST /api/verification/liveness`:
   1. Auth required.
   2. Request: `{ challengeType, metrics, passed }`.
   3. Response: `{ verificationSessionId, livenessStatus }`.
2. `POST /api/scans`:
   1. Auth required.
   2. Request: `{ verificationSessionId, hygieneInputs }`.
   3. Response: `{ scanId, reportType, scoreBundle, breaches, graphData, legalContext }`.
3. `GET /api/scans/:id`:
   1. Auth required.
   2. Response: full stored scan payload.
4. `GET /api/scans`:
   1. Auth required.
   2. Response: list of user scans not expired.
5. `POST /api/scans/:id/simulate`:
   1. Auth required.
   2. Request: `{ enable2FA, stopReuse, removeGravatar, usePasswordManager }`.
   3. Response: `{ baselineScore, simulatedScore, delta, simulatedDimensions }`.
6. `GET /api/scans/:id/legal?regime=gdpr|ccpa&breachId=<id>`:
   1. Auth required.
   2. Response: `{ subject, bodyText, filename }`.
7. `DELETE /api/scans/:id`:
   1. Auth required.
   2. Response: `{ deleted: true }`.
8. `DELETE /api/account/data`:
   1. Auth required.
   2. Response: `{ deleted: true }`.
9. `POST /api/internal/purge-expired`:
   1. Secret-token protected for cron.
   2. Response: `{ purgedScans, purgedBreaches }`.

### Shared Types
1. `HygieneInputs`: `{ uses2FA: boolean; reusesPasswords: boolean; usesPasswordManager: boolean }`.
2. `RiskDimensions`: `{ takeover: number; theft: number; phishing: number; exposure: number }`.
3. `ScoreBundle`: `{ finalScore: number; level: 'low'|'moderate'|'high'; confidence: 'low'|'medium'|'high'; dimensions: RiskDimensions }`.
4. `SignalSnapshot`: normalized booleans/counters used by the engine.
5. `AttackGraphData`: `{ nodes: Node[]; edges: Edge[] }`.

## Risk Engine Specification
1. Input signals:
   1. `breachCount`, `mostRecentBreachDate`.
   2. `hasPassword`, `hasPasswordHash`.
   3. `hasName`, `hasPhone`, `hasAddress`, `hasDOB`.
   4. `hasGravatar`.
   5. `isDisposableDomain`.
   6. `isPredictableEmailPattern`.
   7. hygiene inputs.
2. Dimension formulas (all clamped `0..100`):
   1. Takeover:
      1. `+40` password exposed.
      2. `+30` hash exposed.
      3. `+10 * breachCount` capped at `+60`.
      4. `+20` if most recent breach < 2 years.
      5. `+30` if password reuse.
      6. `+20` if no 2FA.
      7. `-15` if password manager.
   2. Identity Theft:
      1. `+20` name leaked.
      2. `+25` phone leaked.
      3. `+30` address leaked.
      4. `+35` DOB leaked.
   3. Phishing:
      1. `+20` email+name combo present.
      2. `+15` public Gravatar.
      3. `+15` recent breach < 2 years.
      4. `+10` breachCount > 3.
      5. `+10` disposable domain.
   4. Public Exposure:
      1. `+20` public Gravatar.
      2. `+20` predictable email pattern.
      3. `+15` disposable domain.
      4. `+15` any breach exists.
      5. `+10` breachCount > 3.
3. Final score:
   1. `final = round(0.35*takeover + 0.25*theft + 0.20*phishing + 0.20*exposure)`.
4. Risk levels:
   1. `0–34 low`.
   2. `35–64 moderate`.
   3. `65–100 high`.
5. Confidence:
   1. `coverage = availableSignals / totalSignals`.
   2. `high` if `coverage >= 0.70`.
   3. `medium` if `0.40 <= coverage < 0.70`.
   4. `low` if `< 0.40`.
6. Limited fallback report:
   1. Triggered when OTP is valid and liveness not passed.
   2. Returns score bundle with forced `confidence <= medium`.
   3. Shows banner requiring liveness for verified badge and full trust statement.

## Visualization and Analytics Rules
1. Radar chart displays four dimensions exactly.
2. Exposure timeline groups breaches by year.
3. Exposure velocity:
   1. `yearsActive = max(1, currentYear - earliestBreachYear + 1)`.
   2. `velocity = breachCount / yearsActive`.
4. Trend:
   1. Compare breaches in last 24 months vs prior 24 months.
   2. `increasing` if recent window is greater.
   3. `declining` if recent window is lower.
   4. else `stable`.
5. Attack surface graph:
   1. Node types: `email`, `breach`, `data_class`.
   2. Edges: `email->breach` labeled `Leaked In`; `breach->data_class` labeled `Contains`.

## Security, Privacy, and Compliance
1. OTP auth mandatory for scan creation.
2. Scan rate limit: `5/hour/user` and `20/day/IP`.
3. Liveness attempts limit: `10/hour/user`.
4. HIBP key only server-side via env var.
5. No raw face image persistence, no embeddings.
6. Camera frames remain in memory only and are immediately discarded after challenge resolution.
7. Input sanitization via zod and server-side schema checks.
8. Logging policy excludes raw email in app logs; use hashed surrogate ID.
9. Data retention:
   1. `expires_at` set at insert.
   2. Daily cron calls purge endpoint.
   3. User can delete scan or all account data instantly.
10. Legal templates include explicit disclaimer: informational template, not legal advice.

## Implementation Plan (Execution Order)
1. Phase 1: Scaffold app, auth, and base UI shell.
2. Phase 2: OTP flow, protected routes, user bootstrap in DB.
3. Phase 3: Liveness challenge component and verification session storage.
4. Phase 4: Breach ingestion adapters (HIBP, Gravatar, disposable-domain) and scan orchestration API.
5. Phase 5: Risk engine module with deterministic unit-tested scoring.
6. Phase 6: Dashboard modules (overview, radar, timeline, attack graph, simulator).
7. Phase 7: Legal templates, copy/download, and legal export logging.
8. Phase 8: Retention purge endpoint + cron + delete-now UX.
9. Phase 9: Hardening (RLS checks, rate limits, error states, loading states, demo fixtures).
10. Phase 10: End-to-end validation against demo script and acceptance checklist.

## Test Cases and Scenarios
1. Unit tests:
   1. Score formula correctness for baseline, max-risk, and low-risk profiles.
   2. Clamp behavior for each dimension.
   3. Confidence classification boundaries.
   4. Exposure velocity and trend calculation.
2. Integration tests:
   1. `POST /api/scans` with valid verified session returns full payload.
   2. `POST /api/scans` with OTP-only session returns limited report.
   3. HIBP `404` returns zero-breach state without crashing.
   4. HIBP `429/5xx` returns degraded mode with retry messaging.
   5. RLS blocks cross-user scan read.
   6. Rate limit returns `429`.
3. E2E tests:
   1. Full happy path from landing to verified dashboard.
   2. Camera denied path to limited report.
   3. Mitigation toggles update score in real time.
   4. GDPR/CCPA copy/download functions work.
   5. Delete scan removes it from history.
4. Security tests:
   1. Ensure liveness payload cannot include image blobs persisted server-side.
   2. Ensure unauthorized requests to scan/legal endpoints fail `401/403`.
   3. Ensure logs do not contain raw emails.

## Acceptance Criteria
1. Demo flow completes end-to-end in under 2 minutes.
2. Final dashboard shows score, four dimensions, timeline, graph, and simulator.
3. Simulator can show meaningful reduction (example target: ~40% drop with all mitigations).
4. Verified badge only appears when liveness passed.
5. Limited fallback is clearly labeled and functional.
6. Data deletion actions are visible and working.
7. Purge mechanism removes expired records older than 30 days.
8. No biometric data persistence is verifiable in DB schema and code path.

## Assumptions and Defaults Locked
1. App is deployed on Vercel with Supabase project configured.
2. HIBP API key is available; if absent, app runs in degraded mock mode for demo.
3. Disposable-domain detection uses a local packaged list, not an external API.
4. WHOIS/domain-age is excluded from MVP scoring.
5. Single locale and English-only copy for hackathon.
6. Mobile support is responsive but primary demo target is desktop Chrome.
7. “Full feature push” remains the build strategy unless you explicitly re-scope.
