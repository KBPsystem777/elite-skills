---
name: elite-lean-audit
description: >-
  Repo-wide "trim the fat" audit for any language or stack. Finds and safely removes dead code,
  unused template and demo scaffolding, orphaned modules, duplicate logic, junk files, and oversized
  files, then proves nothing broke by re-running the project's own type-check, tests, and build.
  Language-agnostic: detects the toolchain (TS/JS, Python, Rust, Go, Solidity, mixed) and uses the
  right verification gate for each. Inspired by Elon's "delete the part" rule and a16z-grade leanness:
  the best line of code is the one that no longer exists. Trigger on "audit the repo", "trim the fat",
  "lean audit", "remove dead code", "delete junk", "what can we delete", "shrink the codebase", "cut
  the bloat", "find unused code", "declutter", "de-bloat", "elite-lean-audit", or any request to
  reduce, slim, or shrink a codebase. This is about DELETION across the WHOLE repo. It is not a
  security audit (use elite-security-agent for that) and not a single-file refactor (use elite-codes).
---

# elite-lean-audit — Trim the Fat

Cut the codebase down to only what earns its place. Every file, dependency, and line is presumed
guilty until proven load-bearing. The goal is a smaller, faster, more honest repo, not a prettier one.

No emojis. No sugar-coating. Report real numbers. Never claim a cut is safe without proof.

> **The Algorithm (Elon's 5 steps, applied to code):**
> 1. **Question every requirement** — every file, dep, and abstraction must name who needs it.
> 2. **Delete it** — if you are not adding back more than 10% of what you cut, you did not cut enough.
> 3. **Simplify** — only after deleting; never optimize a thing that should not exist.
> 4. **Accelerate** — speed up build, test, and CI once the fat is gone.
> 5. **Automate** — encode the checks so the fat never grows back.
> Most teams skip step 2 and over-invest in step 3. Delete first.

---

## Iron rules (read before touching anything)

1. **Green baseline first.** Capture a passing type-check, test, and build BEFORE any cut. You cannot
   prove you broke nothing without a known-good starting point.
2. **The compiler and test suite are the gate, not the scanner.** Reference-counting greps have false
   positives (see "Detection traps"). After every batch, re-run the toolchain. A broken import surfaces
   instantly. Restore that one item with `git checkout` and move on.
3. **Deletion-only changes per batch.** Do not mix refactors into a trim pass. A clean diff of pure
   deletions is reviewable and trivially revertible.
4. **Dead is not the same as unused-but-load-bearing.** Code wired into live screens or routes (even if
   it serves mock data) is load-bearing. Flag it; do not delete it in a safe pass.
5. **Respect work-in-progress.** Never touch files already modified in the working tree. Diff
   `git status` at the start and steer clear.
6. **Do not delete what you did not write without looking.** If a file's contents contradict "junk"
   (real domain logic, a documented intent), surface it as a flag instead of deleting.
7. **Never auto-commit.** Leave all changes staged or unstaged for the human to review.
8. **Verify, do not assert.** "Done" means type-check, tests, and build are green and you have the
   before and after numbers.

---

## Step 0 — Detect the toolchain (do this first)

The repo tells you how to verify it. Fingerprint the stack, then bind the right commands from the
matrix below. A mixed repo (for example a Next.js front end with a Rust service) uses more than one row.

```bash
git rev-parse --show-toplevel 2>/dev/null || pwd
ls package.json tsconfig.json pnpm-lock.yaml requirements.txt pyproject.toml \
   Cargo.toml go.mod foundry.toml hardhat.config.* 2>/dev/null
cat CLAUDE.md 2>/dev/null | grep -iE 'line|size|limit|lint' | head    # the repo's own size rule
git status --short                                                    # WIP to avoid
```

### Verification matrix (bind the gate to the stack)

| Stack | Type/lint gate | Test gate | Build gate | Dead-code tools | Unused-dep tools |
| ----- | -------------- | --------- | ---------- | --------------- | ---------------- |
| **TS / JS** | `npx tsc --noEmit` | `npx vitest run` or `npm test` | `npm run build` (or `pnpm build`) | `npx knip`, `npx ts-prune` | `npx depcheck` |
| **Python** | `mypy .` or `pyright` | `pytest -q` | `python -m build` if packaged | `vulture .`, `ruff check --select F401,F841` | `deptry .` or `pip-extra` |
| **Rust** | `cargo check` | `cargo test` | `cargo build --release` | `cargo +nightly udeps`, `cargo clippy -- -W dead_code` | `cargo machete` |
| **Go** | `go vet ./...` | `go test ./...` | `go build ./...` | `deadcode ./...`, `staticcheck ./...` | `go mod tidy` (diff the result) |
| **Solidity** | `forge build` or `hardhat compile` | `forge test` or `hardhat test` | same as type gate | manual + `slither . --print human-summary` | n/a |

If a tool is not installed, install it on the fly when the environment allows; if it cannot install,
fall back to the grep method in Step 2 and say so in the report. The grep fallback is universal.

---

## Step 1 — Measure (establish the baseline)

Get hard numbers. You will cite these in the report. Adapt the file globs to the detected language.

```bash
# Size of the codebase (example: TS/TSX; swap extensions per stack)
EXT='-name *.ts -o -name *.tsx'        # py: *.py | rust: *.rs | go: *.go | sol: *.sol
ROOT=src                                 # detect: src / lib / app / packages / the crate root
find "$ROOT" -type f \( $EXT \) | wc -l                       # file count
find "$ROOT" -type f \( $EXT \) -exec cat {} + | wc -l        # LOC

# Files over the project's own size limit (read it from CLAUDE.md / eslint; default 350)
LIMIT=350
find "$ROOT" -type f \( $EXT \) -not -name '*.test.*' \
  | while read -r f; do l=$(wc -l < "$f"); [ "$l" -gt "$LIMIT" ] && echo "$l $f"; done | sort -rn

# LOC by top-level folder — find the elephant
for d in "$ROOT"/*/; do l=$(find "$d" -type f \( $EXT \) -exec cat {} + 2>/dev/null | wc -l); echo "$l $d"; done | sort -rn

# GREEN baseline: run the three gates from the matrix, record exit codes + counts
```

**Spot the template tell.** Scan `package.json`, `Cargo.toml`, and folder names. Purchased admin
templates (Left4code/Midone, tw-starter, Tailwind kits) and starter scaffolds ship huge demo surfaces:
`src/fakers`, `src/data`, demo `DashboardOverviewN`, `Icon`/`Modal`/`Button` showcase pages, `Chat`,
`Billing`, `PointOfSale`. This is usually the single biggest fat deposit.

---

## Step 2 — Detect dead code (presumed guilty)

Prefer the language-aware tool from the matrix; it understands the module graph. Use the grep method as
a universal fallback or a cross-check. A module is **dead** if nothing references it outside itself.

```bash
# Universal fallback: zero-inbound-ref top-level dirs (example: TS pages)
for d in src/pages/*/; do
  name=$(basename "$d")
  refs=$(grep -rl "pages/$name\b" src --include='*.ts' --include='*.tsx' 2>/dev/null \
         | grep -v "/pages/$name/" | wc -l)
  [ "$refs" -eq 0 ] && echo "DEAD $(find "$d" -name '*.ts*' -exec cat {} + | wc -l)LOC $d"
done | sort -t' ' -k2 -rn
```

Run the same pattern for component or module dirs, and cross-check deps in the manifest against
`grep -r "from '<dep>'"` (JS), `use <crate>` (Rust), or `import <pkg>` (Python/Go).

### Detection traps (these WILL bite you)

Text-matching greps catch static imports, dynamic imports, and string paths, but they have two
false-positive classes. Treat the output as a *candidate list*, never a delete-list.

- **Directory-index imports.** `import Layout from "@/themes"` resolves to `src/themes/index.tsx`. A
  scan for `themes/index` finds zero refs and reports a false orphan. The compiler catches this.
- **Relative intra-folder imports.** A leaf imports `./categories` (not `fakers/categories`), so a
  path-prefix scan misses it. Deleting the leaf breaks the parent. The compiler catches this.

**Mitigation:** delete in batches, then re-run the type/build gate. Every false positive becomes a
"cannot find module" error naming the exact file to restore.

---

## Step 3 — Classify each candidate

| Bucket | Signal | Action |
| ------ | ------ | ------ |
| **Junk** | `.DS_Store`, empty dirs, committed build artifacts, stale lockfiles, `target/`, `dist/`, `__pycache__` checked in | Delete (note if already gitignored, then it is not repo fat). |
| **Dead** | Zero inbound refs, confirmed by the gate after deletion | Delete in a verified batch. |
| **Load-bearing-but-ugly** | Referenced by live screens or routes; serves mock or fake data; oversized | **FLAG.** Real refactor, not a trim. Deleting breaks the app. |
| **Domain/infra, 0-ref** | Zero refs but contents are real domain logic or auth/PWA/migration infra | **FLAG.** 0-ref may mean "feature wired incompletely," not "junk." |
| **False orphan** | Flagged by scanner, but the gate proves it is used | Keep. |

The hardest call is **load-bearing fake data**. If real pages render faker output, the whole fake-data
tree is one refactor ("replace mock with real API, then delete wholesale"), never a piecemeal cut,
because the leaves share types and imports.

---

## Step 4 — Execute in verified batches

Order batches safest-first. After EACH batch, run the type/build gate. If red, `git checkout` the
named file and re-run.

1. Junk files and empty dirs.
2. Dead leaf pages, components, or modules (biggest LOC, lowest risk).
3. Cascade re-scan: deleting consumers orphans their helpers, so delete newly-dead, then re-verify.
4. Dead single-file utils or hooks (only the unambiguous ones; flag domain/infra).

Then the **final gate** — all three from the matrix must be green. Example for TS:

```bash
npx tsc --noEmit && npx vitest run && (npm run build || pnpm build)
```

Confirm the diff is deletion-only and WIP files are untouched:

```bash
git diff --numstat --diff-filter=D | awk '{s+=$2} END {print s" lines in "NR" deleted files"}'
git status --short | grep -vE '^ ?D'   # should show only pre-existing WIP, nothing you edited
```

---

## Step 5 — Report (write it into the repo)

Write `docs/audits/<date>-elite-lean-audit.md` with:

- **Before and after**: file count, LOC, percent removed, build and test status on both sides.
- **What was cut**: grouped (template pages, dead components, junk), with LOC each.
- **What was flagged, NOT cut**: load-bearing fakes, 0-ref domain code, oversized files over the line
  limit, unused deps needing manual confirmation, test-coverage gaps. Each with a one-line "why it is
  risky" and the recommended real fix.
- **Proof**: the exact commands run and their exit codes and pass counts.

---

## Step 6 — Automate (so the fat never returns)

Recommend, do not silently add, guardrails matched to the stack:

- A dead-code check in CI that fails the build on new orphans (`knip`/`ts-prune`/`depcheck` for JS,
  `vulture`/`deptry` for Python, `cargo udeps`/`cargo machete` for Rust, `deadcode` for Go).
- A max-file-size lint rule matching the project's own limit (ESLint `max-lines`, ruff, clippy).
- A bundle-size or binary-size budget so demo weight cannot creep back in.

---

## Anti-patterns (do not do these)

- Deleting from the scanner's list without gate verification.
- `rm -rf` of a whole folder (for example `src/fakers`) when part of it is live.
- Mixing "while I am here" refactors into a deletion pass.
- Lowering test or coverage thresholds to force a green checkmark.
- Reporting "removed N lines" using `git diff` totals that include someone else's WIP. Attribute only
  the files YOU deleted (`--diff-filter=D`).
- Claiming a cut is safe without running the build.
