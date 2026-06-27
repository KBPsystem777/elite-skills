# Scoring and report

This file is the single source of truth for severity, the posture score, and
the report structure. Apply it identically every run so any later re-audit
delta is honest rather than flattering.

---

## Severity model

Severity is `likelihood × blast radius`, not a CVSS lookup.

| Severity | Definition | Example |
| -------- | ---------- | ------- |
| **Critical** | Trivially reachable by an unauthenticated attacker AND results in full breach, full takeover, full data loss, or sustained outage. Fix before next deploy. | Unverified Stripe webhook signature; `using (true)` RLS on a PHI table; committed prod DB password; `pull_request_target` + PR checkout. |
| **High** | Reachable with modest effort (an authenticated user, a misconfigured admin, a known-exploit CVE) AND material impact: partial breach, tenant crossover, takeover of high-value account, significant cost burn. Fix this sprint. | IDOR exposing other tenants' invoices; missing rate limit on the LLM endpoint; unbounded file upload size; root container in prod. |
| **Medium** | Reachable but requires chaining or unusual conditions; OR easy to reach but limited impact. Fix this quarter. | Missing HSTS; CSP without `frame-ancestors`; SSO assertion not checking audience strictly; outdated dep with a low-severity CVE. |
| **Low** | Defense-in-depth gap. No direct exploit path, but removes a layer. | Verbose error pages disabled in prod but stack traces leak in one route; cookies missing `SameSite` on a non-auth endpoint. |
| **Info** | Best-practice deviation with no current risk. Note, do not action. | Missing security.txt; outdated copyright year in a security header comment. |

If you cannot place a finding confidently, write it down as **unconfirmed**
and lower-bound the severity. A credible report beats a scary one.

## Likelihood inputs

Multiply qualitatively:

- **Reachability**: public-internet > authenticated-user > admin-only >
  insider-only > requires-physical-access.
- **Exploit complexity**: known PoC, scripted > custom but documented >
  novel research required.
- **Preconditions**: none > weak (any user) > strong (specific role, prior
  compromise, specific config).

## Blast radius inputs

- **Data**: all tenants > one tenant's all users > one user's all records >
  one record.
- **Money**: unbounded > capped at user balance > capped at one
  transaction.
- **Availability**: whole product > one region > one feature > graceful
  degradation only.
- **Trust**: full takeover > admin impersonation > account impersonation >
  data read-only.

---

## Posture score (1–100)

Single number summarizing the system. Compute deterministically.

```
start  = 100
deduct = (Critical × 25) + (High × 10) + (Medium × 3) + (Low × 1)
score  = max(0, start - deduct)
```

Cap rules (apply AFTER the formula):

- Any Critical caps the score at **40**.
- Three or more High caps the score at **60**.
- No Critical and no High caps the score at no less than **70** (a clean
  posture is allowed to score well).

Grade band:

| Score | Band | Meaning |
| ----- | ---- | ------- |
| 90–100 | A | Strong posture. Maintenance and monitoring focus. |
| 75–89 | B | Solid with addressable gaps. |
| 60–74 | C | Material gaps. Sprint of hardening recommended. |
| 40–59 | D | Active exposure. Hardening roadmap is the next priority. |
| 0–39 | F | At least one Critical. Stop shipping non-fix work until closed. |

### Worked example

5 findings: 1 Critical, 2 High, 3 Medium, 4 Low.

```
deduct = (1 × 25) + (2 × 10) + (3 × 3) + (4 × 1) = 25 + 20 + 9 + 4 = 58
raw    = 100 - 58 = 42
cap    = 40 (Critical present)
score  = min(42, 40) = 40 → D
```

---

## Report structure

Write `security/SECURITY_AUDIT.md` (create the folder if needed) with these
sections in this order. Do not reorder — re-audits compare section by
section.

```markdown
# Security audit — <project name> — <YYYY-MM-DD>

## Summary
- Posture score: <N>/100 (<grade band>)
- Findings: Critical <n>, High <n>, Medium <n>, Low <n>, Info <n>
- Top 3 worst exposures:
  1. <one line>
  2. <one line>
  3. <one line>
- Top 3 roadmap items by risk-reduced-per-effort:
  1. <one line>
  2. <one line>
  3. <one line>

## Scope and system map
- Languages / frameworks: ...
- Runtime / hosting: ...
- Data stores: ...
- External services: ...
- Network edge: ...
- Shipping pipeline: ...
- What is OUT of scope (and why): ...

## Threat model
- Crown jewels (top assets): ...
- Entry points (exhaustive list): ...
- Trust boundaries (table): ...
- Crown-jewel data flow: ...
- Adversary set considered: ...
- STRIDE coverage notes: ...

## Findings

### F-001 — <title> — <Severity>
- **Domain**: <one of the 11 audit domains>
- **Asset at risk**: <what gets breached>
- **Attack**: <how an adversary reaches it>
- **Evidence**: <file:line, command output excerpt, config snippet — never
  expose a discovered secret in full>
- **Blast radius**: <worst realistic outcome>
- **Likelihood**: <reachability, complexity, preconditions>
- **Recommended fix**: <concrete, copy-pasteable snippet or config block>
- **Effort**: <S / M / L>

### F-002 — ...
... (repeat for every finding)

## Hardening roadmap (prioritized)

Sequence by risk reduced per unit of effort. Kill the worst exposure first.

| # | Finding | Severity | Effort | Why this order |
| - | ------- | -------- | ------ | -------------- |
| 1 | F-001   | Critical | S      | Trivial fix, removes a breach class. |
| 2 | F-005   | High     | S      | One-liner header, removes Medium too. |
| 3 | ...     | ...      | ...    | ... |

## Scanners run and coverage
- <scanner>: <version>, <findings count>, <output path>
- <scanner that could not run>: <reason>, manual review notes: ...

## Unconfirmed / needs follow-up
List anything the audit could not verify in this run (no cluster access,
production-only behavior, third-party policy not visible). State exactly
what would confirm or deny each item.

## Methodology
- Audit date: <YYYY-MM-DD>
- Audit scope: <as scoped above>
- Posture formula: see references/scoring-and-report.md
- This run made no changes to project code or configuration.
```

---

## Chat summary (after writing the file)

Keep it tight. The file is the deliverable; chat is the index card.

```
Posture: 40/100 (D)
Findings: 1 C, 2 H, 3 M, 4 L
Worst:
  1. <one-line>
  2. <one-line>
  3. <one-line>
Start the roadmap with: <#1 item>
Full report: security/SECURITY_AUDIT.md
```

Stop there. The user reads the file. Do not edit code.

---

## Rules that override everything

- **Never expose a discovered secret in full.** Truncate, reference the
  location, recommend rotation. A committed secret is compromised even
  after deletion because it lives in git history.
- **Do not invent numbers.** Counts come from the actual findings list.
- **Findings must be real and reachable.** Mark anything you could not
  verify as `unconfirmed`. A credible report is more valuable than a
  scary one.
- **The formula is the formula.** Do not adjust the deduction weights to
  flatter or punish. Re-audit deltas are only honest if the math is
  identical across runs.
