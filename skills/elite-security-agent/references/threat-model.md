# Threat model

Build this BEFORE you run a single scanner. The model decides which findings matter and why.

## 1. Enumerate assets

What is worth attacking? List the top 5–10 for the system:

- **PII / PHI** — names, emails, addresses, medical records, government IDs.
- **Credentials** — user passwords, API keys, OAuth tokens, session cookies.
- **Money movement** — payments, payouts, refunds, balance ledgers, crypto keys.
- **Model assets** — proprietary weights, fine-tunes, system prompts, training data.
- **Admin control** — superuser routes, feature flags, billing dashboards, RBAC tables.
- **Availability itself** — the service being up IS an asset, especially for paid SaaS.
- **Reputation** — anything whose breach makes the front page.

For each asset, name: where it lives (DB table, blob storage, env var, in memory),
what touches it (which services, which roles), and what regulation covers it (HIPAA,
PCI, GDPR, SOC 2). Regulation turns a "Medium" finding into a breach class.

## 2. Enumerate entry points

Every place untrusted bytes enter the system. Be exhaustive — the one you miss is
the one that ships:

- Public HTTP routes (REST, GraphQL, RPC).
- Webhooks from third parties (Stripe, GitHub, Twilio, etc.).
- File uploads (direct, presigned, multi-part).
- Queue / event consumers (SQS, Kafka, Pub/Sub) — payloads from upstream are still
  untrusted.
- Third-party OAuth callbacks and SSO assertions.
- CLI / admin tools that read flags, env, or stdin.
- Admin panels (often the softest target — internal "trusted" UIs run with high
  privilege).
- LLM tool calls and function-call results (model output is untrusted input to
  downstream code).

## 3. Map trust boundaries

A boundary is anywhere data crosses between zones with different trust levels.
Common boundaries:

| From | To | Assumption that fails first |
| ---- | -- | ---------------------------- |
| Browser | Server | "the client validated it" — never trust client validation alone. |
| Service A | Service B | "internal traffic is safe" — lateral movement is the norm. |
| App | Database | "the ORM escapes everything" — raw SQL, dynamic queries, JSONB injection. |
| App | LLM | "the prompt is data" — prompt injection turns data into instructions. |
| LLM | Tool call | "the model only calls safe tools" — tool authorization must be enforced server-side. |
| Your code | Dependency | "the package is fine" — supply-chain compromise, typosquatting, transitive CVEs. |
| CI | Cloud | "the runner is trusted" — untrusted PRs running with secrets is the bug. |

For each boundary, write the validation contract: what is checked, by whom, and
what is assumed.

## 4. Data flow for the crown jewel

Pick the single highest-value asset and trace it end to end. Diagram or list:
where it enters, every hop, every store, every export. Mark each hop with the
trust boundary it crosses and the validation that runs there. Anywhere it
crosses a boundary unvalidated is a finding before you've opened a scanner.

Example (PHI in a telehealth app): `Patient browser → TLS → Edge (CDN/WAF) →
API gateway (authn/z) → app server (validate body) → ORM (parameterized) →
Postgres (RLS on the table) → audit log (no PHI) → backup (encrypted at rest).`

If any hop is missing or unverified, that is the first place to dig.

## 5. Adversary set

Be specific. Different adversaries reach different surfaces:

- **Unauthenticated internet** — gets every public endpoint and webhook URL.
  Asks: rate limits, auth, DoS surface.
- **Authenticated but malicious user** — gets every authenticated route, can
  craft input freely, can probe IDOR and tenant boundaries.
- **Compromised dependency** — runs in your build and your runtime. Asks: lock
  files, provenance, what secrets the build sees.
- **Insider** (current or former employee, contractor) — has or had legitimate
  access. Asks: least privilege, audit logs, offboarding.
- **Bot army / commodity attacker** — credential stuffing, scraping, expensive
  endpoint abuse, DDoS for ransom.
- **Targeted attacker** — known to the company, motivated, will chain low-sev
  bugs. Asks: defense in depth.

## 6. STRIDE per boundary

For each trust boundary, walk STRIDE and write the worst realistic outcome:

| Letter | Threat | Typical control |
| ------ | ------ | ---------------- |
| **S** | Spoofing identity | Strong authn, mutual TLS, signed webhooks, request signing. |
| **T** | Tampering with data | Integrity checks, signed tokens, server-side authorization, immutable logs. |
| **R** | Repudiation | Audit logs that survive the actor, non-repudiable signatures. |
| **I** | Information disclosure | Encryption at rest and in transit, least privilege, scoped tokens, no secrets in logs. |
| **D** | Denial of service | Rate limits, quotas, timeouts, autoscaling caps, cost ceilings, circuit breakers. |
| **E** | Elevation of privilege | Deny-by-default authz on every request, no client-supplied role claims, sandboxing. |

Skip a row only if you can say why it is not reachable on this boundary. "Not
applicable" without a reason is hand-waving and hides findings.

## 7. Worked mini-example

> System: a Next.js SaaS app on Vercel, Supabase Postgres, Stripe webhooks, an
> LLM "ask the docs" feature via OpenAI.
>
> Crown jewel: customer document content.
>
> Boundary 1 (Browser → Next.js API): browser sends a doc query. Validation
> contract: signed session cookie, server resolves user, RLS scopes to user's
> org. STRIDE-D: no rate limit on the LLM-backed route — a single user can
> burn the OpenAI bill. Finding.
>
> Boundary 2 (Next.js → OpenAI): prompt is built from user input + retrieved
> doc chunks. STRIDE-T/E: a doc with adversarial markdown can instruct the
> model to exfiltrate other docs in the same context. Finding (prompt
> injection).
>
> Boundary 3 (Stripe → webhook): webhook handler reads `req.body`. STRIDE-S:
> if signature verification is missing or done on a re-serialized body, an
> attacker forges payments. Critical finding if present.

The model points the scanner. The scanner does not replace the model.
