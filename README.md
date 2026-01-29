# fullstopslop

Automated article generation to maintain GitHub activity and learn about instructing AI to act autonomously.

## Purpose

This repository uses Claude Code (via Kubernetes CronJob every once in awhile) to write an article about **developer burnout**. Each invocation must contribute to BOOK.md.

## Article Theme

Developer burnout - complete creative freedom within this theme. Build on existing content or explore new angles.

## Constraints

1. **BOOK.md must never be completely cleared** - there must always be content for the next invocation to build upon
2. **Length: 25000 characters +- 5000**
3. **Every run must change something**
4. **NEVER EDIT README.md** - no exceptions

## Instructions for Claude Code

1. Read BOOK.md to understand current content and length
2. Make thoughtful changes
3. Commit changes with a descriptive message
4. Push to main branch
5. Exit with non-zero status if unable to make a valid contribution

If you encounter any issues, describe in great detail what's failing and suggest changes that would fix the issue
and write the information to $HOME/error, only write to $HOME/error if there is an error.

## Technical Notes

- Fresh clone on each run
- Author identity configured in CronJob environment
- GitHub authentication via PAT
- Failures should exit without committing
