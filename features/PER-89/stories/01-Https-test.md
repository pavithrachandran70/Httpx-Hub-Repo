# Story — Fix typo "equivelent" → "equivalent" in SSL docs

**ID:** (assigned by parent epic)  
**Type:** Story  
**Track:** Quick  
**Complexity:** Trivial

## Description

In the SSL documentation file `README.md`, the word "equivelent" is misspelled. Change it to "equivalent". No behavior change. Add or update nothing else.

## Technical Specification

### Location

**File:** `README.md` (repository root: `/home/pavc/Documents/Https-Demo/README.md`)  
**Search pattern:** The exact string `equivelent`  
**Correction:** Replace with `equivalent`

### Implementation

1. Open `README.md` in the repository root
2. Search for the string `equivelent`
3. Replace the single occurrence with `equivalent`
4. Save the file

No other files, lines, or content should be modified.

### Files Modified

- `README.md` — one word changed on one line

## Acceptance Criteria

### AC-1: Locate and correct the typo
**Given** `README.md` contains the misspelled word "equivelent"  
**When** the file is edited  
**Then** the word is changed to "equivalent" at the exact location where "equivelent" appears

### AC-2: No other changes
**Given** this is a documentation-only typo fix  
**When** the correction is saved  
**Then** no other text, whitespace, formatting, or files are modified

### AC-3: Verification by diff
**Given** the correction has been made  
**When** `git diff` is run  
**Then** it shows exactly one line changed in `README.md`, replacing "equivelent" with "equivalent", and no other modifications

## Verification

- [ ] Manual inspection of `README.md` confirms "equivelent" no longer appears
- [ ] Manual inspection confirms "equivalent" appears in the corrected location
- [ ] `git diff` shows only the single-word change on one line in `README.md`
- [ ] `git status` shows only `README.md` as modified, no other files changed

## Notes

- This is a documentation-only change with no code, test, or configuration impact
- The base branch is `master` per repo context (`Https-test`)
- No automated tests are needed; verification is by diff inspection and manual review
- The file path is confirmed to exist in the repository tree
