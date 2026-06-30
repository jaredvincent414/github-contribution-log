# Contribution [810
[Quality filters: Flaws in the current logic
]

**Contribution Number:** 582  
**Student:** Vincent Jared  
**Issue:** [https://github.com/Listenarrs/Listenarr/issues/582]  
**Status:** [Phase I | Complete]

---

## Why I Chose This Issue

[I am eager to contribute to Listenarr mostly because I have been learning and sharpening my C# and .NET skills and working in a real-world codebase for a tool I's actually use would really do me good in strengthening my skills. Quality filtering is core to the user experience, it's the logic that decides what actually lands in your library.]

## Understanding the Issue

### Problem Description
[When a completed download is imported for an audiobook that already has files, Listenarr is suppose to use the audiobook quality profile to decide whether the incoming file is an upgrade. This logic is implemented in the DownloadImportService.IsQualityBetter, which is currently broken: it accepts a QualityProfile argument but it never actually uses it, and it only compares qualities when both can be regex-parsed into a numeric bitrate. For any non-numeric bitrate format like FLAC or MB4, the comparison falls through to return true, treating the incoming files as always better.]

### Expected Behavior

[When importing, the existing audiobook quality and the download quality should be compared according to the configured quality profiles priorities, and the import should be skipped when the incoming file is not an upgrade]

### Current Behavior

[The quality profile is ignored since "Flac" contains no digits, the bitrate parse fails, the numeric comparison branch is skipped and the method returns true i.e the the MP3 file is imported anyway, leaving the the audiobook with both files, the worse and the better quality one over it.]

### Affected Components

- DownloadImportService.CS - IsQualityBetter
- Models: QualityProfile.CS, AudiobookFile.cs
- There is a related issue # 549 on quality determnination algorithm

---

## Reproduction Process

### Environment Setup

- Tools: NET SDK 10.0.201, Node v24.16.0 (The repo README still says .NET8/ Node 20)
- upgraded Node and .NET
- I run app the full app (npm run dev - runs the full program in the same local port), could do it in different terminal for both frontend and backend
- The flowed logic lived in DownloadImportService, which already has unit-test suite. Reproduction using the tests was way faster.

### Steps to Reproduce

1. Create a quality profile that is above a flossy MP3, and an audiobook assigned to that profile.
2. Add an existing AudiobookFile with format = "flac"
3. Import a completed download that is a low quality MP3
4. Observed result: The MP3 is imported instead of skipped, the audiobook ends up with 2 files instead of 1.

### Reproduction Evidence

- https://github.com/jaredvincent414/Listenarr/tree/fix-issue-582
---

## Solution Approach

### Analysis

[The import path is broken. First, DownloadImportService.IsQualityBetter accepts a QualityProfile but never reads it, it only regex-extracts a numeric bitrate from each side and, for any non-numeric format falls through returning True. !isQualityBetter is never true for the non-numeric formats and the lower quality file imports]

### Proposed Solution

[Make the import-time comparison profile-aware by ranking both the existing quality and incoming quality through profile.Qualities priority, and resolve qualities to a profile QualityDefinition by Codec + Bitrate+IsLossless instead of fragile free-form string quality]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [When a completed download is importedfor an audiobook that already has files, Listenarr should check the audiobook's quality profile to decide whether the incoming file is an upgrade. Currently the decision is a no-op for most real cases, for example a lossy mp3 gets imported on top of an existing lossless FLAC leaving two files]

**Match:** [AutomaticSearchService.IsQualityBetter already compares qualities via profile.Qualities priority. There are also existing tests to mirror like QualityProfileScoringTests, QualityScoringTests ]

**Plan:** [Step-by-step implementation plan]
1. [Quality normalization mapper-AudioMetadata/AudiobookFile → a profile QualityDefinition via codec+bitrate+lossless) so all three vocabularies converge.]
2. [Rewrite DownloadImportService.IsQualityBetter to rank both sides by profile priority, with a documented fallback for unknown quality / no profile (don't block)]
3. [Fix the best existing quality loop so that it doesn't overwrite Format and handles lossless/non-MP3 correctly]
4. [Extract one shared comparator used by both import and search to kill the duplication.]
5. [update and extend the test]

**Implement:** [https://github.com/jaredvincent414/Listenarr/tree/fix-issue-582]

**Review:** 
### Correctness (did I fix the real bug?)

- [ ] `IsQualityBetter` in `listenarr.application/Downloads/DownloadImportService.cs:490` now
      **reads the `QualityProfile`** (no longer ignores it).
- [ ] Quality strings are **normalized before comparison** — candidate, existing, and profile
      qualities use one vocabulary (root cause #2); the fix isn't just a string-equality patch
      that breaks on real metadata.
- [ ] The "best existing quality" loop (`DownloadImportService.cs:107-114`) no longer overwrites
      `Format` with MP3 labels or mis-divides bitrate.
- [ ] Lossless-over-lossy is handled (FLAC vs MP3), not just numeric MP3 bitrates.
- [ ] Unknown quality / no profile still **allows** import (no silent drops) — and that's
      intentional/commented, not accidental.
- [ ] The two `IsQualityBetter` contracts (import = "allow" vs search = "strictly better") are
      reconciled — no inverted-boolean bug at any call site (`DownloadImportService.cs:186`).

## Tests

- [ ] Repro `QualityGating_LosslessExisting_SkipsLossyDownload` now **passes**.
- [ ] Regression `QualityGating_SkipsLowerQualityImport` still passes.
- [ ] Added the matrix: numeric upgrade, cross-format upgrade, equal-format reimport,
      unknown/no-profile.
- [ ] `dotnet test --filter "FullyQualifiedName~DownloadImportServiceTests"` green.
- [ ] Full `dotnet test` green; `cd fe && npm run test:unit` + `npm run type-check` green.

## Code style & architecture

- [ ] C# **4-space** indentation; meaningful names; comments on the comparison logic.
- [ ] Helper lives in `application`/`domain` — **no infrastructure/EF leakage** (layering hook
      passes).
- [ ] No new compiler warnings; pre-commit hook (format + layering + async-void) passes.
- [ ] *nix line endings.

## Scope discipline

- [ ] One bug-fix per PR — the "delete existing files when better" idea is **deferred**, not
      bundled.
- [ ] Diff contains only files relevant to #582 (no stray/IDE/config noise).

## Commits

- [ ] Fix is its own commit, Conventional-Commit style with scope + issue ref, e.g.
      `fix(downloads): make import quality comparison profile-aware (#582)`.
- [ ] History is meaningful or squashed (no "wip"/"fix typo" noise).

## PR readiness (enforced by repo)

- [ ] Rebased on **latest `canary`**; PR targets **`canary`** (not `main`/`beta`).
- [ ] Exactly **one** version label: **`patch`** (`.github/workflows/validate-pr-labels.yml` fails
      on zero or multiple).
- [ ] PR body fills `.github/PULL_REQUEST_TEMPLATE.md`: Summary / Changes→Fixed / Testing / Notes.
- [ ] Body says **"Fixes #582"**; references related **#549**.
- [ ] CONTRIBUTING PR checklist all ticked (self-review, comments, tests added, tests pass,
      no warnings, docs if needed, rebased).

## Final pass

- [ ] Re-read my own diff line by line as if reviewing someone else.
- [ ] Confirmed the behavior change manually or via the skip log line
      (`DownloadImportService.cs:189`) if running the app.


**Evaluate:** [ Repro test QualityGating_LosslessExisting_SkipsLossyDownload flips green; QualityGating_SkipsLowerQualityImport stays green; full dotnet test + cd fe && npm run test:unit + npm run type-check all pass.]

---

## Testing Strategy

### Unit Tests

- [ ] Lossless vs lossy: existing FLAC + lossy MP3 download → MP3 skipped (the repro; currently failing).
- [ ] Numeric downgrade (regression): existing MP3 320 + MP3 128 → skipped (already passing — must stay green).
- [ ] Numeric upgrade: existing MP3 128 + MP3 320 → imported.
- [ ] Cross-format upgrade: existing MP3 + FLAC download, profile ranks FLAC higher → imported.
- [ ] Equal-format reimport: same quality on both sides → handled per policy (not treated as a spurious upgrade).
- [ ] Unknown quality / no profile: import allowed (explicit fallback, no silent drop).


### Integration Tests

- [ ]  Full ImportDownloadFilesAsync flow against a DB-backed audiobook with an existing file + assigned profile → verify final AudiobookFile count and the kept file's format.
- [ ]  Multi-file batch where some files are upgrades and some aren't → only upgrades imported

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
