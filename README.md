# fullstopslop

Automated book generation to gamify AI spending and maintain GitHub activity.

## Purpose

This repository uses Claude Code (via Kubernetes CronJob every 5-6 hours) to write a book about **developer burnout**. Each invocation must contribute to BOOK.md.

## Book Theme

Developer burnout - complete creative freedom within this theme. Build on existing content or explore new angles.

## Constraints

1. **BOOK.md must never be completely cleared** - there must always be content for the next invocation to build upon
2. **Maximum size: 50,000 characters** - if approaching this limit, focus on editing/refining existing sections rather than adding new content
3. **No complete refactors** - additive changes, refinements, and expansions are encouraged; wholesale replacement of large sections is not
4. **Every run must commit something** - even small contributions count (new paragraph, refinement, expansion)
4. **NEVER EDIT README.md** - no exceptions

## Instructions for Claude Code

1. Read BOOK.md to understand current content and length
2. If under 50k chars: Add new content, expand existing sections, or explore new burnout-related themes
3. If near 50k chars: Focus on polish, clarification, or minor additions
4. Commit changes with a descriptive message
5. Push to main branch
6. Exit with non-zero status if unable to make a valid contribution

## Technical Notes

- Fresh clone on each run
- Author identity configured in CronJob environment
- GitHub authentication via PAT
- Failures should exit without committing
