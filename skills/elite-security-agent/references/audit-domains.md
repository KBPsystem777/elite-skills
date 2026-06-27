# Audit domains

The cross-layer checklist. For every domain, the question is: what is the
worst realistic outcome if an adversary works this surface, and how hard is
it to reach? Walk each domain even if a prior scanner reported nothing —
scanners do not understand business logic, tenancy, or availability.

---

## 1. Network and edge / availability and DDoS

The most-asked, most-ignored domain. Code scanners do not detect this.

- **Rate limits per IP, per account, per route.** Login, signup, password
  reset, SMS/email send, file upload, LLM endpoints. Missing limits = account
  takeover, SMS pumping ($$$), unbounded LLM bills.
- **WAF / CDN in front of the origin.** Cloudflare, Fastly, AWS WAF.
  Direct-to-origin must be blocked at the network layer, not just hidden.
- **Autoscaling caps and connection limits.** Autoscaling without a ceiling
  is a billing DoS, not protection.
- **Expensive-endpoint cost protection.** Any route that runs a query, fans
  out, calls an LLM, generates a PDF, transcodes media → cap concurrency,
  cap result size, cap input size, charge against a quota.
- **Idempotency keys** on every write that the client may retry. Without
  them, a bot army duplicates orders.
- **Timeouts everywhere.** HTTP client, DB query, LLM call, external API. A
  hanging upstream consumes a worker forever.
- **Circuit breakers and graceful degradation.** When the LLM provider is
  down, the app degrades — it does not 500 the whole site.
- **Slowloris / large-body / decompression bombs.** Body-size limits at the
  edge AND in the framework. Validate `Content-Length`. Cap zip/gzip
  expansion ratio.

## 2. Authentication and access control

- **Authentication strength.** MFA available and enforced for admin roles.
  Password policy not a joke. SSO assertions verified (signature, audience,
  expiry, nonce).
- **Session and token handling.** Cookies `HttpOnly`, `Secure`, `SameSite`.
  JWTs with short expiry, refresh tokens revocable. No JWT in localStorage
  for sensitive apps. `alg: none` rejected. Algorithm fixed server-side.
- **Broken access control / IDOR.** Every authenticated request must
  re-derive authorization server-side from the session. Never trust an
  object ID from the client without checking the requester owns it. Test:
  swap an ID, confirm 403.
- **Multi-tenant isolation.** Tenant ID resolved server-side from the
  session, never read from the request body or URL. Postgres RLS on every
  tenant-scoped table. Indexed by tenant first.
- **Deny by default.** New routes inherit no access. Authorization is a
  positive grant, not a negative filter.
- **Admin surface.** Separate audit trail, MFA enforced, IP allowlist for
  the most dangerous actions, rate-limited.

## 3. Injection and untrusted input

- **SQL.** Parameterized queries everywhere. No string concatenation. ORMs
  also vulnerable when raw mode is used. Hunt: `query(`+`)`, f-strings in
  SQL.
- **NoSQL.** Mongo operator injection (`{$gt: ""}`). Never pass raw request
  bodies into queries.
- **Command.** No `shell=True`, no `eval`, no `exec`. Use `execFile`/argv
  arrays. Validate filenames against an allowlist.
- **SSRF.** Outbound HTTP from user-controlled URLs is SSRF until proven
  otherwise. Block RFC1918, link-local, `169.254.169.254` (cloud metadata),
  `localhost`. Resolve once and pin (TOCTOU).
- **Deserialization.** No `pickle`, `Marshal.load`, Java native serialization
  on untrusted input.
- **Template injection.** No user input rendered as a template (Jinja,
  Handlebars, Liquid). Use the data slot.
- **Path traversal.** Resolve, then check `startswith(safe_root)`. Strip
  `..` is not enough.
- **XSS / output encoding.** Auto-escape on by default. `dangerouslySetInnerHTML`,
  `v-html`, `bypassSecurityTrust*` flagged. CSP with `default-src 'self'`,
  no `unsafe-inline`.
- **Prompt injection.** Treat all retrieved content (docs, search results,
  tool outputs) as untrusted input to the model. See domain 10.

## 4. Data protection

- **Encryption in transit.** TLS 1.2+ everywhere, HSTS, no mixed content.
  Internal service-to-service also TLS — not just edge.
- **Encryption at rest.** DB-level, backup-level, object storage SSE.
  Verify keys are KMS-managed, not committed.
- **PII / PHI handling.** Minimize collection, define retention, support
  deletion, audit access. PHI never in logs, never in URLs, never in
  analytics events.
- **Data residency.** GDPR (EU), HIPAA (US health), PCI (cards), data
  sovereignty for regulated geos. The wrong region for the wrong data is a
  legal breach, not a security one.
- **RLS / row-level security.** For Postgres/Supabase, every tenant table
  has RLS enabled AND a policy. `using (true)` is not a policy. Service
  role keys never reach the browser.
- **Backup and recovery.** Backups encrypted, tested, stored separately
  (cross-region), restorable within stated RPO/RTO. Ransomware-survivable.

## 5. Secrets management

- **Committed secrets.** Run gitleaks/trufflehog over full history. Any hit
  is `Critical` — rotate, do not just delete.
- **Client bundles.** Grep the built JS for `_KEY`, `SECRET`, `TOKEN`,
  service URLs. Anything prefixed `NEXT_PUBLIC_`/`VITE_`/`REACT_APP_` is
  public — confirm it is supposed to be.
- **Server logs.** No tokens, no PII, no full request bodies for sensitive
  endpoints.
- **Key scope and rotation.** Per-service credentials, least privilege, no
  shared admin keys, documented rotation cadence, rotation actually
  performed.
- **Secret storage.** Vault / KMS / Doppler / cloud secrets manager — not
  `.env` checked into the runtime image.

## 6. Supply chain

- **Lockfile present and committed.** Reproducible builds.
- **CVEs.** SCA scanner output triaged: direct deps fixed, transitive deps
  upgraded via overrides where blocked.
- **Unmaintained deps.** Last release > 2 years + open critical CVEs = risk.
- **Typosquatting.** Diff installed packages against the manifest — extra
  packages with one-letter-off names are an attack.
- **Provenance.** SLSA / sigstore signatures where ecosystem supports it.
- **Postinstall scripts.** Audit which packages run code at install. Block
  with `--ignore-scripts` in CI where possible.

## 7. Infrastructure and cloud posture

- **Public exposure.** S3/GCS buckets, DBs, Redis, Elasticsearch — none
  should be public unless explicitly intended. Run cloud-native config
  audits.
- **IAM least privilege.** No `*:*` policies. Service identities scoped to
  exact resources. No long-lived access keys when role assumption works.
- **Network segmentation.** Private subnets for data tier. Security groups
  / firewall rules scoped to the source they need.
- **Default-open config.** Default security group, default VPC, default
  IAM — replace, do not extend.

## 8. Containers and IaC

- **Non-root user.** `USER` directive in Dockerfile, runtime enforced.
- **Read-only root filesystem** where possible.
- **No secrets in images.** Multi-stage builds; `--secret` mount for build-time.
- **Resource limits.** CPU + memory caps on every container. Without them,
  one pod OOMs the node.
- **Pinned base images** by digest, not by `:latest`.
- **`hostNetwork`/`hostPID`/`privileged`** off unless the workload truly
  needs them, and then justified in writing.

## 9. CI/CD and build

- **`pull_request_target` with PR checkout** = remote code execution with
  secrets. Hard "Critical" if present.
- **Third-party actions pinned by SHA.** Tags are mutable.
- **OIDC federation** for cloud deploys instead of stored long-lived keys.
- **Branch protection.** Required reviewers, status checks, no force push
  to main, signed commits if claimed.
- **Artifact signing** (cosign, GPG) for anything that ships.
- **Self-hosted runners** isolated; never reused across repos with different
  trust levels.

## 10. AI and LLM attack surface

First-class for any AI product. Most apps fail at least three of these.

- **Prompt injection.** Treat ALL non-system text as untrusted: user input,
  retrieved docs, tool outputs, search results, web pages, PDFs. The model
  will obey instructions hidden in them.
- **Output handling treated as trusted.** Never `eval` model output. Never
  render model output as HTML without sanitization. Never feed model output
  to a SQL query without parameterization. Model output is user input to
  the next system.
- **Tool / function-call authorization.** Authorization is checked
  server-side at the tool boundary, not in the prompt. "The system prompt
  said the model can only do X" is not a control.
- **Data exfiltration via the model.** RAG retrievals must be scoped to the
  caller's permissions BEFORE going into the context. Cross-tenant data in
  context = cross-tenant leak.
- **Unbounded token cost.** Per-user / per-org token quotas. Max
  conversation length. Max retrieval-context size. Without these, one
  abusive user empties the budget.
- **Indirect prompt injection.** Documents uploaded by user A read by user
  B's session can carry instructions targeting B's data.
- **Model and weights exfil.** If self-hosting, the model file is a crown
  jewel — same protections as a DB dump.

## 11. Observability and incident readiness

- **Audit log.** Every privileged action (auth, role change, data export,
  admin action). Immutable / append-only. Survives the actor.
- **No PII in logs.** Structured, redacted, sampled.
- **Anomaly detection.** Failed-auth spikes, geographic anomalies, sudden
  data export volume, sudden cost spikes — alerts wired to a human, not
  just to a dashboard nobody opens.
- **Incident playbook.** Documented, rehearsed, with contact lists current.
  "Who has prod DB access at 3am" answered before 3am.
- **Detection latency.** Mean time to detect target stated and measured.

## AI-slop patterns LLM-written code keeps reintroducing

These regress in every Copilot/Cursor/Claude-generated PR. Grep for them:

- `<input>.toString()` interpolated into a SQL query.
- `child_process.exec(<user input>)` instead of `execFile`.
- `requests.get(<user url>)` with no allowlist or metadata-IP block.
- `eval`, `Function()`, `vm.runInNewContext` on model or user input.
- `dangerouslySetInnerHTML={ { __html: <user|model output> } }`.
- JWT secret hardcoded as `"secret"`, `"changeme"`, or env var with no
  validation that it is set.
- CORS `Access-Control-Allow-Origin: *` with `Allow-Credentials: true`
  (browser will refuse, but the intent is wrong).
- S3 buckets with `acl: public-read` "for now".
- Webhook handlers that read `req.body` before verifying the signature
  against the raw body bytes.
- Supabase `using (true)` RLS policies "to unblock dev."
- LLM `function_call` handlers that execute the function without
  re-validating the caller's permission.
