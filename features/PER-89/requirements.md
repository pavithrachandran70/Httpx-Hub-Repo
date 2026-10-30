# Requirements — PER-89

**Ticket:** Fix typo "equivelent" → "equivalent" in SSL docs  
**Type:** Bug  
**Complexity:** Trivial

## Context

A single-word typo exists in SSL-related documentation within the `Https-test` repository. The misspelling "equivelent" must be corrected to "equivalent" with no other changes.

## Acceptance Criteria

### AC-1: Locate and correct the typo
**Given** the file `README.md` contains the misspelled word "equivelent"  
**When** the typo is corrected  
**Then** the word "equivelent" is replaced with "equivalent" at its exact location

### AC-2: No other changes
**Given** this is a documentation-only typo fix  
**When** the file is saved  
**Then** only the single word on the identified line is modified, with no other text, formatting, whitespace, or files changed

### AC-3: Verification by diff
**Given** the correction has been made  
**When** `git diff` is run  
**Then** it shows only the one-word change from "equivelent" to "equivalent" on a single line in `README.md`

## Verification

- Manual inspection of `README.md` confirms "equivelent" no longer appears
- Manual inspection confirms "equivalent" appears in its place
- `git diff` shows exactly one line changed in `README.md`, with only the word replacement
- `git status` shows no modifications to any other files

## Out of Scope

- Searching for or fixing other typos in the repository
- Modifying code, tests, or configuration files
- Adding spell-check tooling or CI checks
- Changing documentation structure, formatting, or content beyond the single word
