# Story — Fix typo "equivelent" → "equivalent" in SSL docs

**ID:** (assigned by parent epic)  
**Type:** Story  
**Track:** Quick  
**Complexity:** Trivial

## Description

In the SSL documentation, the word "equivelent" is misspelled. Change it to "equivalent". No behavior change. Add or update nothing else.

## Technical Specification

### Location

**File:** `README.md` (SSL-related documentation in repository root)  
**Search pattern:** The exact string `equivelent`  
**Correction:** Replace with `equivalent`

### Implementation

1. Open `README.md`
2. Locate the line containing `equivelent`
3. Replace `equivelent` with `equivalent`
4. Save the file

No other files, lines, or content should be modified.

## Acceptance Criteria

### AC-1: Locate and correct the typo
**Given** the SSL documentation contains the misspelled word "equivelent"  
**When** the file is edited  
**Then** the word is changed to "equivalent" at the exact location of the typo

### AC-2: No other changes
**Given** this is a documentation-only typo fix  
**When** the correction is made  
**Then** no other text, formatting, or files are modified

### AC-3: File identified
**Given** the typo exists in SSL-related documentation  
**When** searching the repository  
**Then** the exact file path `README.md` and line number containing "equivelent" is identified

## Verification

- [ ] Manual inspection confirms "equivelent" no longer appears in `README.md`
- [ ] Manual inspection confirms "equivalent" appears in its place at the corrected location
- [ ] `git diff` shows only the single-word change on one line in `README.md`
- [ ] No other files show modifications in `git status` or `git diff`

## Notes

- This is a documentation-only change with no code, test, or configuration impact
- The base branch is `master` per repo context
- No automated tests are needed; verification is by diff inspection
