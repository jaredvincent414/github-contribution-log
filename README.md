# Contribution [810
[Quality filters: Flaws in the current logic
]

**Contribution Number:** [582]  
**Student:** [Your Name]  
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

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

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
