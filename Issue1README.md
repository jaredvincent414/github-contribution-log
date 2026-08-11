# Contribution [810]
[Quality filters: Flaws in the current logic]

**Contribution Number:** 582
**Student:** Vincent Jared
**Issue:** https://github.com/Listenarrs/Listenarr/issues/582
**Status:** Phase IV | PR #752 open against upstream, awaiting maintainer review

---

## Why I Chose This Issue

I am eager to contribute to Listenarr mostly because I have been learning and sharpening my C#
and .NET skills, and working in a real-world codebase for a tool I'd actually use would do me
good in strengthening those skills. Quality filtering is core to the user experience — it's the
logic that decides what actually lands in your library.

## Understanding the Issue

### Problem Description
When a completed download is imported for an audiobook that already has files, Listenarr is
supposed to use the audiobook's quality profile to decide whether the incoming file is an
upgrade. That decision ran through `ImportQualityEvaluator.IsAcceptable`
(`listenarr.application/Downloads/Import/ImportQualityEvaluator.cs`), which was broken in two
ways: it accepted a `QualityProfile` argument but **never read it** (the profile was used only
as a null-check), and it decided purely by regex-extracting a numeric bitrate from each side.

> NOTE: The issue text and my initial write-up referred to
> `DownloadImportService.IsQualityBetter`. That method no longer exists — the import path was
> refactored before I started, and the broken logic now lives in `ImportQualityEvaluator`.
> The diagnosis held; only the location had moved.

### Expected Behavior
On import, the existing audiobook quality and the download quality should be compared according
to the configured quality profile's priorities, and the import should be skipped when the
incoming file is not an upgrade.

### Current Behavior (before fix)
For any non-numeric quality (e.g. an existing "flac"), the bitrate regex fails to parse, the
numeric comparison is skipped, and the method falls through to `return true`. The lossy MP3 is
imported anyway, leaving the audiobook with both files — the worse one sitting on top of the
better.

### Affected Components
- `listenarr.application/Downloads/Import/ImportQualityEvaluator.cs` — `IsAcceptable` (root cause)
- `listenarr.application/Downloads/Import/DownloadImportService.cs` — best-existing loop + the gate
- `listenarr.domain/Common/QualityMatcher.cs` — the correct comparator that already existed
- Models: `QualityProfile.cs`, `AudiobookFile.cs`, `AudioMetadata.cs`
- Related: issue #549 (quality determination algorithm)

---

## Reproduction Process

### Environment Setup
- .NET SDK 10.0.201, Node v24.16.0 (the repo README still says .NET 8 / Node 20; CONTRIBUTING
  correctly says .NET 10 / Node 24 — the README is stale)
- Upgraded Node and .NET accordingly
- App runs with `npm run dev` from the root (concurrently starts the API on :4545, waits on the
  readiness probe, then Vite on :5173); `dotnet watch run` + `cd fe && npm run dev` in separate
  terminals is better for backend iteration
- The flawed logic sits in the import path, which already has a unit-test suite, so reproducing
  via tests was far faster than driving the real app

### Steps to Reproduce
1. Create a quality profile that ranks lossless FLAC above lossy MP3, and assign an audiobook to it.
2. Add an existing `AudiobookFile` with `Format = "flac"`.
3. Import a completed download that is a lossy MP3.
4. Observed: the MP3 is imported instead of skipped; the audiobook ends up with 2 files instead of 1.

### Reproduction Evidence
- Branch: https://github.com/jaredvincent414/Listenarr/tree/fix-issue-582
- Failing test: `QualityGating_LosslessExisting_SkipsLossyDownload`
  → `Assert.Single() Failure: The collection contained 2 items`

---

## Solution Approach

### Analysis
Two root causes, not one:
1. **The profile was ignored.** `IsAcceptable` took a `QualityProfile` and never read it.
2. **Vocabulary mismatch.** The decision compared free-form strings ("flac", "MP3 320kbps",
   "128kbps") that three subsystems each spell differently, via a regex that only understands
   digits. Any lossless format has no digits, so it fell through to "acceptable".

A correct, profile-aware, lossless-aware comparator **already existed** in the domain:
`QualityMatcher` (`Match`, `MatchLabel`, `IsLabelBetter`). The automatic-search path already used
it via `AutomaticSearchQualityEvaluator`. **Import was the only path still on the broken regex
evaluator.** So the fix was not to build a comparator — it was to route import onto the one that
already existed.

### Proposed Solution
Make the import-time comparison profile-aware by projecting both sides into `AudioQualityInput`
(the shared codec/container/format/bitrate shape `QualityMatcher` consumes) and ranking them by
the profile's `QualityDefinition.Priority`, instead of comparing fragile free-form strings.

### Implementation Plan (UMPIRE, adapted)

**Understand:** When a completed download is imported for an audiobook that already has files,
Listenarr should check the quality profile to decide whether the incoming file is an upgrade.
The decision was a no-op for most real cases — a lossy MP3 lands on top of an existing lossless
FLAC, leaving two files.

**Match:** `QualityMatcher` in `listenarr.domain/Common/` is the single source of truth for what
a profile's priority ordering means (lower number = higher quality) and how an on-disk file maps
onto a profile rung. `AutomaticSearchQualityEvaluator.IsQualityBetter` already delegates to it.
Existing tests to mirror: `QualityProfileScoringTests`, `QualityScoringTests`.

**Plan:**
1. Expose `QualityMatcher.IsLossless(AudioQualityInput)` (the logic existed but was private).
2. Add projections `FromFile(AudiobookFile)` / `FromMetadata(AudioMetadata?, path)` so the
   library file and the incoming download converge on one vocabulary.
3. Rewrite `IsAcceptable` to rank both sides by profile priority, with a documented fallback for
   unknown quality / no profile (don't block).
4. Fix the best-existing loop so it stops hardcoding MP3 labels and clobbering `Format`.
5. Fix and extend the tests.

**Implement:** https://github.com/jaredvincent414/Listenarr/tree/fix-issue-582
- `c0b0e4ef test(downloads): show lossy MP3 imported over existing FLAC (#582)`
- `133a0400 fix(downloads): make import quality comparison profile-aware (#582)`

**Review:**

### Correctness (did I fix the real bug?)
- [x] The import gate now **reads the `QualityProfile`** (no longer ignores it).
- [x] Quality is **normalized before comparison** — candidate, existing, and profile qualities
      converge on `AudioQualityInput` / profile rungs; not a string-equality patch.
- [x] The best-existing loop no longer overwrites `Format` with MP3 labels or mis-buckets bitrate.
- [x] Lossless-over-lossy is handled (FLAC vs MP3), not just numeric MP3 bitrates.
- [x] Unknown quality / no profile still **allows** import (no silent drops) — intentional and
      commented.
- [x] The two contracts are reconciled: **import = "not worse" (allow-equal)**; **search =
      "strictly better"** (`QualityMatcher.IsLabelBetter`). No inverted-boolean bug at any call site.

## Tests
- [x] Repro `QualityGating_LosslessExisting_SkipsLossyDownload` now **passes**.
- [x] Regression `QualityGating_SkipsLowerQualityImport` still passes.
- [x] Matrix added (numeric upgrade, cross-format upgrade, equal-format reimport, unknown/no-profile).
- [x] `dotnet test --filter "FullyQualifiedName~DownloadImportServiceTests"` green.
- [x] Full `dotnet test` green (**1020 passed, 0 failed**); `cd fe && npm run test:unit`
      (389 passed) + `npm run type-check` green.

> **Honest note on the matrix:** only the repro test distinguishes the fixed code from the broken
> code. The four matrix cases would also pass against the old logic (its regex bitrate parse fell
> through to "acceptable" for any non-numeric label, so it imported in all four). Their job is to
> guard against **over-correction**, not to prove the fix. The load-bearing one is
> `QualityGating_EqualQuality_IsImported`: it fails if the gate is ever tightened to the
> strictly-better rule that automatic search uses, which would skip every part of a multi-file
> audiobook whose quality merely matches what is already on disk.

## Code style & architecture
- [x] C# 4-space indentation; meaningful names; comparison logic commented.
- [x] Logic lives in `application` + `domain` — no infrastructure/EF leakage (layering hook passes).
- [x] **0 warnings, 0 errors** on `dotnet build`; pre-commit hook (format + layering + async-void) passed.
- [x] *nix line endings.

## Scope discipline
- [x] One bug-fix per PR — "delete existing files when better" is **deferred**, not bundled. It is
      blocked on #542 (multi-file size), #736 (recycling bin), #737 (stored codec/bitrate).
- [x] Diff contains only the 4 relevant files (+89 / −37); no stray/IDE/config noise.

## Commits
- [x] Fix is its own commit, Conventional-Commit style with scope + issue ref.
- [x] History is meaningful: failing test, then fix (red → green).

## PR readiness (enforced by repo)
- [x] Rebased on latest `canary`.
- [x] Pushed to fork (`jaredvincent414/Listenarr:fix-issue-582`, force-with-lease; the remote had
      held the stale pre-rebase branch). Pre-push hook passed: backend format, FE type-check, FE tests.
- [x] PR targets `canary`.
- [ ] Exactly one **`patch`** label. Not yet applied — labeling requires repo-admin rights I don't
      have as an outside contributor; a maintainer will need to add it during triage.
- [x] PR body fills the template; says "Fixes #582"; references #549.

## Final pass
- [ ] Re-read the diff line by line as a reviewer.
- [x] Behavior confirmed via the test suite (skip path now taken).

**Evaluate:** Repro flipped green; regression stayed green; full `dotnet test` (**1020 passed**),
`npm run test:unit` (389 passed), and `npm run type-check` all pass; build is warning-free.

---

## Testing Strategy

### Unit Tests
- [x] Lossless vs lossy: existing FLAC + lossy MP3 download → MP3 skipped (**the repro; now green**).
- [x] Numeric downgrade (regression): existing MP3 320 + MP3 128 → skipped (**still green**).
- [x] Numeric upgrade: existing MP3 128 + MP3 320 → imported.
- [x] Cross-format upgrade: existing MP3 + FLAC download, profile ranks FLAC higher → imported.
- [x] Equal-format reimport: same quality both sides → allowed (multi-file parts must still import).
- [x] Unknown quality / no profile: import allowed (explicit fallback, no silent drop).

All six live in `DownloadImportServiceTests` as `QualityGating_*` and run against the real service
and repositories, so they are closer to integration tests than isolated unit tests.

### Integration Tests
- [x] Full `ImportDownloadFilesAsync` flow against a DB-backed audiobook with an existing file +
      assigned profile → final `AudiobookFile` count and kept file's format asserted (this is what
      the `QualityGating_*` tests actually exercise — real service + real repositories).
- [ ] Multi-file batch where some files are upgrades and some aren't → only upgrades imported.
      (Deferred: the current gate compares each candidate against the best existing file, so this
      is implied but not yet asserted end-to-end.)

### Manual Testing
Not required for this fix — the gate is fully exercised through the service-level test, which is a
far faster loop than driving the real app. The skip is observable at runtime via the log line in
`DownloadImportService` ("Skipping import of file ... because candidate quality '...' is not better
than existing '...'"), which now fires where it previously never did.

---

## Implementation Notes

### Week 3 Progress

**Built:**
- `QualityMatcher.IsLossless(AudioQualityInput)` — made the existing private lossless check public.
- `ImportQualityEvaluator`: added `FromFile` / `FromMetadata` projections and `Describe` (readable
  log label); replaced the string+regex `IsAcceptable` with a profile-aware one; deleted
  `TryParseBitrate`.
- `DownloadImportService`: removed the hardcoded MP3 bitrate buckets; best-existing file is now
  chosen by reusing `IsAcceptable` as the comparator; the gate compares projected inputs.
- Fixed the repro test's profile priorities.

**Challenges / decisions:**

1. **The issue's stated location was stale.** `DownloadImportService.IsQualityBetter` doesn't exist
   anymore. I had to trace the current import path to find that the logic had moved to
   `ImportQualityEvaluator.IsAcceptable`. Lesson: verify the issue's line references against the
   current code before planning.

2. **My own repro test was subtly wrong.** I had written the profile as
   `flac Priority = 2, mp3 Priority = 1` with a comment saying it "ranks lossless FLAC above lossy
   MP3" — but `QualityDefinition.Priority` is documented **lower number = higher quality**, so my
   profile ranked MP3 *above* FLAC. The test was failing for the wrong reason, and a *correct* fix
   would have made it fail too. Fixed to `flac = 1, mp3 = 2`. Lesson: a red test isn't automatically
   a valid red test — verify it fails for the reason you think.

3. **The empty-profile trap (the key design insight).** The existing passing test
   `QualityGating_SkipsLowerQualityImport` builds its profile with `new QualityProfileBuilder().Build()`,
   which produces an **empty** `Qualities` list. It only passed because the old regex path ignored the
   profile entirely. If I had routed *everything* through `QualityMatcher`, that profile would match no
   rung → the gate would allow the 128kbps downgrade → a currently-**passing** test would break. So
   `IsAcceptable` needed two paths: profile-priority when the profile can rank both sides, and a
   codec/bitrate fallback (lossless-beats-lossy, then numeric bitrate) when it can't. The fallback is a
   bonus: lossy-over-lossless is now blocked **even for users with no quality profile configured**.

4. **Import vs. search semantics.** Search asks "is the candidate *strictly better*?"
   (`IsLabelBetter`). Import must ask "is the candidate *not worse*?" — allow-equal — because a
   multi-file audiobook imports chapter parts whose quality *equals* what's already on disk, and a
   strict-better rule would skip every one of them. Getting this backwards would have been a silent,
   nasty regression.

### Code Changes
- **Files modified (4):**
  - `listenarr.domain/Common/QualityMatcher.cs` — expose `IsLossless(AudioQualityInput)`
  - `listenarr.application/Downloads/Import/ImportQualityEvaluator.cs` — projections + the new
    profile-aware `IsAcceptable`; deleted the regex `TryParseBitrate`
  - `listenarr.application/Downloads/Import/DownloadImportService.cs` — removed the hardcoded MP3
    bitrate buckets; best-existing selection and the gate now use the new comparator
  - `tests/Features/Application/Downloads/Import/DownloadImportServiceTests.cs` — fixed the repro
    profile; added the four matrix cases
  - Fix commit: **+89 / −37**; tests commit: **+148 / −1**
- **Key commits** (branch `fix-issue-582`, pushed to the fork):
  - `c0b0e4ef` — `test(downloads): show lossy MP3 imported over existing FLAC (#582)` (failing repro)
  - `133a0400` — `fix(downloads): make import quality comparison profile-aware (#582)`
  - `5abae8a1` — `test(downloads): cover import quality gating matrix (#582)`
- **Approach decisions:** reuse the existing `QualityMatcher` rather than write a new comparator;
  keep the import-specific *policy* (allow-equal, allow-unknown) in the application layer and the
  *ranking* in the domain layer, preserving the project's layering rules. Kept three commits rather
  than squashing, so the red → green story stays legible to a reviewer.

---

## Pull Request

**PR Link:** https://github.com/Listenarrs/Listenarr/pull/752

**Summary:** Import gating ran through `ImportQualityEvaluator.IsAcceptable`, which accepted a
`QualityProfile` but never read it, so any non-numeric quality (an existing FLAC) fell through to
"acceptable" and a lossy MP3 could import on top of a lossless file. The PR routes import onto the
same profile-aware `QualityMatcher` the automatic-search path already uses, comparing "not worse"
(allow-equal) rather than the strictly-better rule search uses, so multi-file audiobook parts that
match on-disk quality still import. The delete/replace-on-upgrade half of #582 is deliberately out
of scope, deferred behind #542, #736, and #737.

**Status:** Open, awaiting maintainer review. Opened 2026-08-11. `@Paelsmoessan` was tagged via PR
comment. `patch` label requested but not yet applied (needs maintainer/admin rights).

---

## Maintainer Feedback Log

| Date | Feedback | My Response | Commit Ref |
|------|----------|--------------|------------|
| _(none yet)_ | PR #752 opened 2026-08-11; no maintainer feedback received yet. | — | — |

_This table will be updated as review comments come in._

---

## Learnings & Reflections

### Technical Skills Gained
- Reading a layered .NET codebase (domain / application / infrastructure / api) and respecting its
  boundaries — the repo enforces "no infrastructure references from application" via a git hook.
- C# pattern matching (`is int x and > 0`), nullable value types, `readonly record struct`, and
  object-initializer construction.
- xUnit + builder-pattern test fixtures, and service-level tests that run against real repositories.
- Git: diagnosing divergent branches, understanding that a rebase rewrites SHAs, and why
  `--force-with-lease` is the safe way to publish a rebased branch.

### Challenges Overcome
- **Stale issue references** — learned to trust the code over the issue text.
- **A test that lied** — my own repro had inverted priority numbers; it failed, but for the wrong
  reason. Catching this before "fixing" the code saved me from writing a fix that satisfied a
  broken test.
- **Not breaking a passing test** — discovering the empty-profile case is what forced the
  two-path design, which turned out to be a *better* fix than the plan I started with.
- **Environment drift** — after rebasing onto a newer `canary`, all 75 frontend suites failed with
  `Tsconfig not found`. Root cause was a stale `fe/node_modules` (missing `@tsconfig/node24`, vite
  behind the lockfile), not my code. `cd fe && npm ci` fixed it. Lesson: after a rebase that touches
  `package.json`, reinstall before trusting test failures.

- **An architecture mismatch that broke the pre-push hook.** The push failed with
  `Cannot find module '@rolldown/binding-darwin-x64'`, even though the frontend tests passed when I
  ran them by hand. Cause: my `/usr/local/bin/git` is an **x86_64-only** binary, so on Apple Silicon
  it runs under Rosetta and every process it spawns — including the husky hook — inherits x86_64.
  `node` is a universal binary, so the hook's node ran its **x64 slice** and looked for an x64
  rolldown binding, while `npm ci` (running under arm64 node) had installed only the **arm64** one.
  Pushing through `/usr/bin/git` (Apple's, which has a native arm64 slice) made the hook pass. The
  real lesson: when a hook fails but the same command succeeds in your shell, suspect the
  *environment the hook runs in*, not the command.

### What I'd Do Differently Next Time
- Verify the issue's cited file/line references against `HEAD` **before** writing the plan — half my
  original plan targeted a method that no longer existed.
- Search for an existing utility before proposing a new one. I planned to "extract a shared
  comparator"; `QualityMatcher` already was one. The real fix was much smaller than my plan assumed.
- Write the repro test's *data* as carefully as its assertions — and sanity-check that it fails for
  the intended reason.

---

## Resources Used
- `BACKEND_ARCHITECTURE.md` — the layering contract and worker/state-ownership rules
- `CONTRIBUTING.md` — branching model (canary → beta → main), commit and PR rules
- Issue #582 discussion — especially @Paelsmoessan's comment establishing that the
  delete/replace strategy is blocked on #542, #736, #737
- Related issues: #549 (quality determination), #542 (multi-file size aggregation)
