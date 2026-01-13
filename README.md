# fullstopslop - Automated Book Generation

**Runtime**: Kubernetes CronJob (every 5-6 hours)
**Command**: `claude --print $(cat README.md)`
**Operation**: Fresh clone → modify BOOK.md → commit → push to main
**Theme**: Developer burnout (complete creative freedom)

---

## CRITICAL RULES - NEVER VIOLATE

These constraints MUST be enforced before any commit:

1. **BOOK.md MUST NEVER BE EMPTY** - Always preserve existing content
2. **BOOK.md MUST STAY UNDER 50,000 CHARACTERS** - Check before and after edits
3. **README.md MUST NEVER BE MODIFIED** - Only BOOK.md should change (this is README.md)
4. **EVERY RUN MUST COMMIT SOMETHING** - Even small contributions count
5. **NO COMPLETE REFACTORS** - Additive changes only, no wholesale replacement

---

## ERROR SIGNALING PROTOCOL

**CRITICAL**: Claude Code CLI exit codes are unreliable. You MUST use file-based error signaling.

### When ANY failure occurs:

1. **IMMEDIATELY create error file**: Write to `$HOME/error`
2. **Write structured error information** using the template below
3. **Restore clean state**: Run `git restore .` if changes were made
4. **EXIT immediately**: Do not attempt to continue or recover

### Error File Template

```markdown
# EXECUTION FAILED

**Timestamp**: [ISO 8601 timestamp]
**Failed Step**: [Step name and number]
**Failure Type**: [Brief category]

## Error Details

**Command**: [Exact command that failed]
**Exit Code**: [Exit code if available]
**Output**:
[Complete error output - not summarized]

## Diagnostic Information

- Working directory: [pwd output]
- Git status: [git status --short output]
- Last commit: [git log --oneline -1 output]
- BOOK.md size: [wc -c BOOK.md output] characters
- Branch: [git branch --show-current output]

## Suggested Fix

[Specific actionable fix for this failure type]
```

### When to Create Error File

Create `$HOME/error` if ANY of these occur:

- Pre-flight check fails
- Validation step fails
- Git operation returns failure patterns
- BOOK.md becomes empty or exceeds 50,000 characters
- README.md gets modified
- Push capability test fails
- Actual push operation fails
- Any unexpected error or exception

**Remember**: Creating `$HOME/error` is your ONLY reliable way to signal failure.

---

## PRE-FLIGHT CHECKS - RUN BEFORE ANY WORK

Execute ALL checks before editing BOOK.md. If ANY fail, create `$HOME/error` and exit immediately.

### Check 1: File Validation

Verify BOOK.md exists and has valid content:

```bash
# Check BOOK.md exists and is not empty
if [ -f BOOK.md ] && [ -s BOOK.md ]; then
  echo "✓ BOOK.md exists and has content"
else
  echo "✗ BOOK.md missing or empty"
  cat > $HOME/error <<'EOF'
# EXECUTION FAILED

**Timestamp**: $(date -Iseconds)
**Failed Step**: Pre-Flight Check 1 - File Validation
**Failure Type**: File Missing or Empty

## Error Details

BOOK.md does not exist or is empty. This violates the critical constraint that BOOK.md must never be empty.

## Diagnostic Information

- Working directory: $(pwd)
- Files present: $(ls -la)

## Suggested Fix

Ensure repository clone completed successfully. BOOK.md should always exist in the repository.
EOF
  exit 1
fi

# Check character count is valid (0 < size < 50000)
CHAR_COUNT=$(wc -c < BOOK.md)
if [ "$CHAR_COUNT" -gt 0 ] && [ "$CHAR_COUNT" -lt 50000 ]; then
  echo "✓ BOOK.md size OK: $CHAR_COUNT characters"
else
  echo "✗ BOOK.md size invalid: $CHAR_COUNT characters"
  cat > $HOME/error <<EOF
# EXECUTION FAILED

**Timestamp**: $(date -Iseconds)
**Failed Step**: Pre-Flight Check 1 - File Validation
**Failure Type**: File Size Constraint Violation

## Error Details

BOOK.md character count is invalid: $CHAR_COUNT characters
Valid range: 1 to 49,999 characters

## Diagnostic Information

- Working directory: $(pwd)
- BOOK.md size: $CHAR_COUNT characters
- Size limit: 50,000 characters

## Suggested Fix

If BOOK.md is at or over capacity, focus on polishing existing content rather than adding new content.
If BOOK.md is empty, this is a critical failure - investigate repository state.
EOF
  exit 1
fi

# Verify README.md exists
if [ -f README.md ]; then
  echo "✓ README.md exists"
else
  echo "✗ README.md missing"
  cat > $HOME/error <<'EOF'
# EXECUTION FAILED

**Timestamp**: $(date -Iseconds)
**Failed Step**: Pre-Flight Check 1 - File Validation
**Failure Type**: README.md Missing

## Error Details

README.md (this file) does not exist in the working directory.

## Diagnostic Information

- Working directory: $(pwd)
- Files present: $(ls -la)

## Suggested Fix

Ensure repository clone completed successfully. README.md should always exist.
EOF
  exit 1
fi
```

### Check 2: Git Configuration

Verify git is properly configured:

```bash
# Verify git identity configured
if git config user.name >/dev/null && git config user.email >/dev/null; then
  echo "✓ Git identity configured"
  echo "  User: $(git config user.name) <$(git config user.email)>"
else
  echo "✗ Git identity not configured"
  cat > $HOME/error <<'EOF'
# EXECUTION FAILED

**Timestamp**: $(date -Iseconds)
**Failed Step**: Pre-Flight Check 2 - Git Configuration
**Failure Type**: Git Identity Not Configured

## Error Details

Git user.name or user.email is not configured. Cannot create commits without identity.

## Diagnostic Information

- user.name: $(git config user.name || echo "NOT SET")
- user.email: $(git config user.email || echo "NOT SET")

## Suggested Fix

Configure git identity in CronJob environment:
- git config user.name "Your Name"
- git config user.email "your.email@example.com"
EOF
  exit 1
fi

# Verify on main branch
BRANCH=$(git branch --show-current)
if [ "$BRANCH" = "main" ]; then
  echo "✓ On main branch"
else
  echo "✗ Not on main branch: $BRANCH"
  cat > $HOME/error <<EOF
# EXECUTION FAILED

**Timestamp**: $(date -Iseconds)
**Failed Step**: Pre-Flight Check 2 - Git Configuration
**Failure Type**: Wrong Branch

## Error Details

Expected to be on 'main' branch, but current branch is: $BRANCH

## Diagnostic Information

- Current branch: $BRANCH
- Available branches: $(git branch -a)

## Suggested Fix

Ensure CronJob clones and checks out the main branch:
- git clone <repo>
- git checkout main
EOF
  exit 1
fi

# Verify clean working directory
if [ -z "$(git status --porcelain)" ]; then
  echo "✓ Working directory clean"
else
  echo "✗ Working directory not clean"
  echo "$(git status --porcelain)"
  cat > $HOME/error <<EOF
# EXECUTION FAILED

**Timestamp**: $(date -Iseconds)
**Failed Step**: Pre-Flight Check 2 - Git Configuration
**Failure Type**: Dirty Working Directory

## Error Details

Working directory has uncommitted changes. Expected clean state after fresh clone.

## Diagnostic Information

$(git status)

## Suggested Fix

Ensure fresh clone is used for each run. Working directory should be clean before starting.
EOF
  exit 1
fi
```

### Check 3: Push Capability Test (CRITICAL)

This catches authentication failures BEFORE doing any work:

```bash
echo "Testing push capability..."
PUSH_TEST=$(git push --dry-run origin main 2>&1)
PUSH_TEST_EXIT=$?

# Check for failure patterns in output
if echo "$PUSH_TEST" | grep -qiE "(Authentication failed|Permission denied|Could not read|fatal.*auth|403|401)"; then
  echo "✗ PUSH TEST FAILED"
  echo "$PUSH_TEST"
  cat > $HOME/error <<EOF
# EXECUTION FAILED

**Timestamp**: $(date -Iseconds)
**Failed Step**: Pre-Flight Check 3 - Push Capability Test
**Failure Type**: Git Push Authentication/Permission Error

## Error Details

**Command**: git push --dry-run origin main
**Exit Code**: $PUSH_TEST_EXIT
**Output**:
$PUSH_TEST

## Diagnostic Information

- Working directory: $(pwd)
- Git remote: $(git remote -v)
- Branch: $(git branch --show-current)

## Suggested Fix

Authentication or permission issue detected. Verify:
1. GitHub PAT is valid and not expired
2. PAT has 'repo' scope with write permissions
3. PAT is correctly configured in CronJob environment
4. Repository permissions allow pushes to main branch
5. Repository URL is correct and accessible
EOF
  exit 1
fi

# Also check exit code
if [ $PUSH_TEST_EXIT -ne 0 ]; then
  echo "✗ PUSH TEST FAILED (exit code: $PUSH_TEST_EXIT)"
  echo "$PUSH_TEST"
  cat > $HOME/error <<EOF
# EXECUTION FAILED

**Timestamp**: $(date -Iseconds)
**Failed Step**: Pre-Flight Check 3 - Push Capability Test
**Failure Type**: Git Push Failed

## Error Details

**Command**: git push --dry-run origin main
**Exit Code**: $PUSH_TEST_EXIT
**Output**:
$PUSH_TEST

## Suggested Fix

Push capability test failed. Check authentication, permissions, and network connectivity.
EOF
  exit 1
fi

echo "✓ Push capability verified"
```

**STOP IF ANY CHECK FAILS**: Create `$HOME/error` with details and exit immediately.

---

## WORKFLOW STEPS

### Step 1: Read Current Content

**Objective**: Understand existing BOOK.md to build upon it

**Actions**:
1. Read entire BOOK.md file
2. Note current themes, chapters, writing style, and voice
3. Identify areas for expansion, refinement, or new angles
4. Check current character count

```bash
CHAR_COUNT=$(wc -c < BOOK.md)
echo "Current BOOK.md size: $CHAR_COUNT characters"
```

**Success Criteria**: Clear understanding of existing content and available space

---

### Step 2: Plan Your Contribution

**Objective**: Decide what to add/improve within theme and constraints

**Decision Logic**:

```
IF char_count < 45000:
  THEN: Add substantial new content OR explore new angles on developer burnout
  EXAMPLES: New chapter, extended analysis, additional perspectives

ELSE IF char_count >= 45000 AND char_count < 49000:
  THEN: Moderate additions OR polish existing content
  EXAMPLES: Expand existing sections, add examples, refine arguments

ELSE IF char_count >= 49000:
  THEN: Minor additions and refinement only
  EXAMPLES: Polish prose, add brief insights, clarify existing content

ALWAYS:
  - Stay within developer burnout theme
  - Build on existing content (no wholesale replacement)
  - Maintain consistent voice and quality
  - Make meaningful contribution (not trivial)
```

**Success Criteria**: Clear plan for specific, meaningful contribution

---

### Step 3: Modify BOOK.md

**Objective**: Make meaningful contribution to the book

**Actions**:
1. Edit BOOK.md using Read/Edit tools (NOT bash/sed/awk)
2. Ensure contribution is substantial and valuable
3. Maintain consistent voice, quality, and formatting
4. Stay within developer burnout theme

**VALIDATION IMMEDIATELY AFTER EDIT**:

Run these checks right after editing, before proceeding:

```bash
echo "Validating BOOK.md changes..."

# Check 1: Verify BOOK.md was actually modified
if ! git diff --quiet BOOK.md; then
  echo "✓ BOOK.md was modified"
else
  echo "✗ BOOK.md not modified - no contribution made"
  cat > $HOME/error <<'EOF'
# EXECUTION FAILED

**Timestamp**: $(date -Iseconds)
**Failed Step**: Step 3 - Modify BOOK.md (Validation)
**Failure Type**: No Contribution Made

## Error Details

BOOK.md was not modified. Every run must make a contribution.

## Diagnostic Information

- Git diff output: (empty - no changes)
- BOOK.md size: $(wc -c < BOOK.md) characters

## Suggested Fix

Ensure edit operations actually modify BOOK.md. If approaching capacity limit, make smaller refinements.
EOF
  exit 1
fi

# Check 2: Verify BOOK.md is not empty (CRITICAL)
if [ -s BOOK.md ]; then
  echo "✓ BOOK.md has content"
else
  echo "✗ BOOK.md is empty - CRITICAL FAILURE"
  # Restore from git immediately
  git restore BOOK.md
  cat > $HOME/error <<'EOF'
# EXECUTION FAILED

**Timestamp**: $(date -Iseconds)
**Failed Step**: Step 3 - Modify BOOK.md (Validation)
**Failure Type**: File Integrity Critical Failure

## Error Details

BOOK.md became empty after edit operation. This violates the critical constraint that BOOK.md must never be empty.

## Diagnostic Information

- BOOK.md size: 0 bytes
- File was restored from git

## Suggested Fix

Edit operation catastrophically failed. Review edit logic:
1. Ensure Edit tool is using correct parameters
2. Never use Write tool to completely replace BOOK.md
3. Use Edit tool for incremental changes only
4. Test edit operations carefully
EOF
  exit 1
fi

# Check 3: Verify character count within limit
CHAR_COUNT=$(wc -c < BOOK.md)
if [ "$CHAR_COUNT" -lt 50000 ]; then
  echo "✓ BOOK.md within limit: $CHAR_COUNT characters"
else
  echo "✗ BOOK.md exceeds limit: $CHAR_COUNT characters"
  # Restore from git
  git restore BOOK.md
  cat > $HOME/error <<EOF
# EXECUTION FAILED

**Timestamp**: $(date -Iseconds)
**Failed Step**: Step 3 - Modify BOOK.md (Validation)
**Failure Type**: File Size Constraint Violated

## Error Details

BOOK.md exceeds 50,000 character limit after edit.

**Current size**: $CHAR_COUNT characters
**Limit**: 50,000 characters
**Overage**: $(($CHAR_COUNT - 50000)) characters

## Diagnostic Information

- File was restored from git
- Edit added too much content

## Suggested Fix

Reduce the size of your contribution. Options:
1. Add less new content
2. Make smaller expansions to existing sections
3. Focus on refinement rather than addition
4. If close to limit, make only minor polish edits
EOF
  exit 1
fi

# Check 4: Verify README.md was NOT modified
if git diff --quiet README.md; then
  echo "✓ README.md unchanged"
else
  echo "✗ README.md was modified - FORBIDDEN"
  # Restore everything
  git restore .
  cat > $HOME/error <<'EOF'
# EXECUTION FAILED

**Timestamp**: $(date -Iseconds)
**Failed Step**: Step 3 - Modify BOOK.md (Validation)
**Failure Type**: Protected File Modified

## Error Details

README.md was modified. This violates the critical constraint that README.md must never be modified.

## Diagnostic Information

README.md changes:
$(git diff README.md)

## Suggested Fix

All files were restored. Review code logic:
1. ONLY modify BOOK.md
2. Never edit README.md (this file)
3. Ensure Edit/Write tools target correct file
4. Double-check file paths before operations
EOF
  exit 1
fi

# Check 5: Verify no other files were modified
OTHER_CHANGES=$(git status --porcelain | grep -v "^ M BOOK.md" | grep -v "^M  BOOK.md")
if [ -z "$OTHER_CHANGES" ]; then
  echo "✓ Only BOOK.md modified"
else
  echo "✗ Other files were modified"
  echo "$OTHER_CHANGES"
  git restore .
  cat > $HOME/error <<EOF
# EXECUTION FAILED

**Timestamp**: $(date -Iseconds)
**Failed Step**: Step 3 - Modify BOOK.md (Validation)
**Failure Type**: Unexpected File Modifications

## Error Details

Files other than BOOK.md were modified:
$OTHER_CHANGES

## Diagnostic Information

Full git status:
$(git status)

## Suggested Fix

All files were restored. Only BOOK.md should be modified. Review file operations.
EOF
  exit 1
fi

echo "✓ All validations passed"
```

**Success Criteria**: All 5 validation checks pass

---

### Step 4: Stage and Commit Changes

**Objective**: Create descriptive commit with only BOOK.md changes

**Actions**:

```bash
# Stage BOOK.md only
echo "Staging changes..."
git add BOOK.md

# Verify staging - should show "M  BOOK.md" only
STAGED=$(git status --porcelain)
if echo "$STAGED" | grep -qE "^M  BOOK\.md$"; then
  echo "✓ Only BOOK.md staged"
else
  echo "✗ Unexpected staging state: $STAGED"
  cat > $HOME/error <<EOF
# EXECUTION FAILED

**Timestamp**: $(date -Iseconds)
**Failed Step**: Step 4 - Stage and Commit
**Failure Type**: Incorrect Staging

## Error Details

Expected staging: "M  BOOK.md"
Actual staging: $STAGED

## Diagnostic Information

Full git status:
$(git status)

## Suggested Fix

Only BOOK.md should be staged. Review git add operations.
EOF
  exit 1
fi

# Create commit with descriptive message
# Use HEREDOC for proper formatting
echo "Creating commit..."
git commit -m "$(cat <<'COMMIT_MSG'
[Write descriptive commit message here about your contribution]

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
COMMIT_MSG
)"

COMMIT_EXIT=$?

# Verify commit was created
if [ $COMMIT_EXIT -eq 0 ]; then
  echo "✓ Commit created successfully"
  git log --oneline -1
else
  echo "✗ Commit failed (exit code: $COMMIT_EXIT)"
  cat > $HOME/error <<EOF
# EXECUTION FAILED

**Timestamp**: $(date -Iseconds)
**Failed Step**: Step 4 - Stage and Commit
**Failure Type**: Commit Failed

## Error Details

**Command**: git commit
**Exit Code**: $COMMIT_EXIT

## Diagnostic Information

Git status before commit:
$(git status)

## Suggested Fix

Commit operation failed. Possible causes:
1. Git hooks rejected commit
2. Commit message format issue
3. Git configuration problem
4. File system or permission issue
EOF
  exit 1
fi

# Verify working directory is clean after commit
if [ -z "$(git status --porcelain)" ]; then
  echo "✓ Working directory clean after commit"
else
  echo "✗ Working directory not clean after commit"
  cat > $HOME/error <<EOF
# EXECUTION FAILED

**Timestamp**: $(date -Iseconds)
**Failed Step**: Step 4 - Stage and Commit (Post-Validation)
**Failure Type**: Working Directory Not Clean After Commit

## Error Details

Working directory should be clean after successful commit.

## Diagnostic Information

$(git status)

## Suggested Fix

Investigate why working directory is not clean after commit. May indicate:
1. Incomplete staging
2. Files modified during commit (hooks?)
3. Unexpected file system changes
EOF
  exit 1
fi

echo "✓ Commit successful"
```

**Success Criteria**: Commit created, working directory clean

---

### Step 5: Push to Remote

**Objective**: Push changes to origin/main with explicit failure detection

**Actions**:

```bash
# Re-test push capability before actual push
echo "Re-testing push capability before push..."
PUSH_TEST=$(git push --dry-run origin main 2>&1)
if echo "$PUSH_TEST" | grep -qiE "(Authentication failed|Permission denied|Could not read|fatal.*auth|403|401)"; then
  echo "✗ Push capability test failed before actual push"
  cat > $HOME/error <<EOF
# EXECUTION FAILED

**Timestamp**: $(date -Iseconds)
**Failed Step**: Step 5 - Push to Remote (Pre-Push Test)
**Failure Type**: Push Capability Lost

## Error Details

Push capability test succeeded in pre-flight checks but now fails.
Authentication may have been revoked or expired during execution.

**Command**: git push --dry-run origin main
**Output**:
$PUSH_TEST

## Diagnostic Information

- Last commit: $(git log --oneline -1)
- Branch: $(git branch --show-current)
- Commit was created but not pushed

## Suggested Fix

Commit exists locally but was not pushed. Authentication issue occurred between pre-flight and push.
1. Verify PAT is still valid
2. Check for PAT expiration or revocation
3. Verify network connectivity
EOF
  exit 1
fi

echo "✓ Push capability confirmed"

# Perform actual push and capture ALL output (stdout + stderr)
echo "Pushing to origin/main..."
PUSH_OUTPUT=$(git push origin main 2>&1)
PUSH_EXIT=$?

echo "Push command completed with exit code: $PUSH_EXIT"
echo "Push output:"
echo "$PUSH_OUTPUT"

# Parse output for success/failure patterns
# Success patterns: "main -> main", "Writing objects: 100%", "branch 'main' set up"
# Failure patterns: "Authentication failed", "Permission denied", "rejected", "failed to push", "fatal"

if echo "$PUSH_OUTPUT" | grep -qiE "(main -> main|Writing objects.*100%|Everything up-to-date)"; then
  echo "✓ Push successful (confirmed by output pattern match)"
elif echo "$PUSH_OUTPUT" | grep -qiE "(Authentication failed|Permission denied|rejected|failed to push|Could not read|fatal|error:)"; then
  echo "✗ PUSH FAILED (failure pattern detected)"

  # Determine specific failure type for better error message
  FAILURE_TYPE="Unknown"
  SUGGESTED_FIX="Unknown push failure. Check error output."

  if echo "$PUSH_OUTPUT" | grep -qi "Authentication failed"; then
    FAILURE_TYPE="Authentication Failed"
    SUGGESTED_FIX="GitHub PAT authentication failed. Verify PAT validity and permissions."
  elif echo "$PUSH_OUTPUT" | grep -qi "Permission denied"; then
    FAILURE_TYPE="Permission Denied"
    SUGGESTED_FIX="Permission denied. Verify PAT has write access to repository."
  elif echo "$PUSH_OUTPUT" | grep -qi "rejected"; then
    FAILURE_TYPE="Push Rejected"
    SUGGESTED_FIX="Push rejected. Check for branch protection rules or upstream changes requiring pull."
  elif echo "$PUSH_OUTPUT" | grep -qi "Could not read"; then
    FAILURE_TYPE="Connection Failed"
    SUGGESTED_FIX="Could not read from remote. Check network connectivity and remote URL."
  fi

  cat > $HOME/error <<EOF
# EXECUTION FAILED

**Timestamp**: $(date -Iseconds)
**Failed Step**: Step 5 - Push to Remote
**Failure Type**: $FAILURE_TYPE

## Error Details

**Command**: git push origin main
**Exit Code**: $PUSH_EXIT
**Output**:
$PUSH_OUTPUT

## Diagnostic Information

- Working directory: $(pwd)
- Git status: $(git status --short)
- Last commit: $(git log --oneline -1)
- BOOK.md size: $(wc -c < BOOK.md) characters
- Branch: $(git branch --show-current)
- Remote: $(git remote -v)

## Suggested Fix

$SUGGESTED_FIX

Note: Commit was created locally but not pushed to remote.
EOF
  exit 1
else
  # Output doesn't match known success or failure patterns - be conservative
  echo "⚠ Push completed but result is unclear"
  echo "Exit code: $PUSH_EXIT"
  echo "Output doesn't match expected success or failure patterns"

  # If exit code is 0 and no failure patterns, tentatively accept as success
  if [ $PUSH_EXIT -eq 0 ]; then
    echo "✓ Exit code is 0 - assuming success despite unclear output"
  else
    echo "✗ Exit code is non-zero - treating as failure"
    cat > $HOME/error <<EOF
# EXECUTION FAILED

**Timestamp**: $(date -Iseconds)
**Failed Step**: Step 5 - Push to Remote
**Failure Type**: Push Result Unclear

## Error Details

**Command**: git push origin main
**Exit Code**: $PUSH_EXIT
**Output**:
$PUSH_OUTPUT

## Diagnostic Information

Push completed but result is unclear:
- Exit code: $PUSH_EXIT (non-zero)
- Output doesn't match expected success or failure patterns

- Working directory: $(pwd)
- Git status: $(git status --short)
- Last commit: $(git log --oneline -1)
- Remote: $(git remote -v)

## Suggested Fix

Unable to confirm push success. Manual verification recommended:
1. Check repository on GitHub for latest commit
2. Verify network connectivity
3. Review git push output above for clues
EOF
    exit 1
  fi
fi

echo "✓ Push operation completed successfully"
```

**Success Criteria**: Push output contains success patterns, no error file created

---

### Step 6: Final Validation

**Objective**: Confirm successful completion

**Actions**:

```bash
echo "Running final validation..."

# Verify git status is clean
if [ -z "$(git status --porcelain)" ]; then
  echo "✓ Working directory clean"
else
  echo "⚠ Working directory not clean after push"
  echo "$(git status --porcelain)"
  # Don't create error file - push succeeded, this is just a warning
fi

# Verify local and remote HEAD match (confirms push succeeded)
git fetch origin main --quiet
REMOTE_HEAD=$(git rev-parse origin/main 2>/dev/null)
LOCAL_HEAD=$(git rev-parse HEAD)

if [ "$REMOTE_HEAD" = "$LOCAL_HEAD" ]; then
  echo "✓ Local and remote synchronized"
  echo "  Local HEAD:  $LOCAL_HEAD"
  echo "  Remote HEAD: $REMOTE_HEAD"
else
  echo "⚠ Local and remote out of sync"
  echo "  Local HEAD:  $LOCAL_HEAD"
  echo "  Remote HEAD: $REMOTE_HEAD"
  # This shouldn't happen if push succeeded, but don't fail here
fi

# Display success summary
echo ""
echo "=== EXECUTION SUCCESSFUL ==="
echo "Commit: $(git log --oneline -1)"
echo "BOOK.md size: $(wc -c < BOOK.md) characters"
echo "No error file created - CronJob will detect success"
echo "============================"
```

**Success Criteria**: All checks pass, execution complete

**NO ERROR FILE EXISTS**: Success is indicated by the absence of `$HOME/error`

---

## FAILURE HANDLING REFERENCE

### Common Failure Scenarios and Fixes

**1. Pre-Flight: Push Test Fails**
- **Symptoms**: "Authentication failed", "Permission denied" in pre-flight check 3
- **Failure Type**: Git Push Authentication/Permission Error
- **Suggested Fix**: Verify GitHub PAT validity, expiration, and 'repo' scope permissions

**2. Step 3: BOOK.md Empty After Edit**
- **Symptoms**: BOOK.md has 0 bytes after edit operation
- **Failure Type**: File Integrity Critical Failure
- **Recovery**: Automatic `git restore BOOK.md`
- **Suggested Fix**: Review edit operations - never use Write to replace entire file

**3. Step 3: Character Limit Exceeded**
- **Symptoms**: BOOK.md > 50,000 characters after edit
- **Failure Type**: File Size Constraint Violated
- **Recovery**: Automatic `git restore BOOK.md`
- **Suggested Fix**: Reduce contribution size, focus on polish instead of expansion

**4. Step 3: README.md Modified**
- **Symptoms**: `git diff README.md` shows changes
- **Failure Type**: Protected File Modified
- **Recovery**: Automatic `git restore .`
- **Suggested Fix**: Only modify BOOK.md - check file paths in edit operations

**5. Step 5: Authentication Failed During Push**
- **Symptoms**: "Authentication failed" in push output
- **Failure Type**: Git Push Authentication Failed
- **Suggested Fix**: GitHub PAT invalid or expired - update CronJob secret

**6. Step 5: Permission Denied During Push**
- **Symptoms**: "Permission denied" or "403" in push output
- **Failure Type**: Git Push Permission Denied
- **Suggested Fix**: PAT lacks push permissions - verify 'repo' scope with write access

**7. Step 5: Push Rejected**
- **Symptoms**: "rejected" in push output
- **Failure Type**: Git Push Rejected
- **Suggested Fix**: Check branch protection rules or pull upstream changes first

---

## FOR CLAUDE CODE: EXECUTION CONTEXT

**You are running in an automated environment via `claude --print $(cat README.md)`**

### Your Mission

Extend BOOK.md with quality content on the theme of **developer burnout**. You have complete creative freedom within this theme. Build on existing content, explore new angles, add depth, or refine what's there.

### Your Capabilities

- Read files (BOOK.md, README.md, LICENSE)
- Modify BOOK.md using Read and Edit tools
- Execute git commands and analyze outputs
- Create `$HOME/error` file on failures

### Your Constraints

- **No interactive input**: Single-shot execution, no retry mechanism
- **Exit codes unreliable**: Must use `$HOME/error` for failure signaling
- **Cannot modify README.md**: This file you're reading is protected
- **BOOK.md must never be empty**: Critical constraint
- **50,000 character limit**: Hard maximum for BOOK.md

### Critical Success Factors

**1. Run ALL Pre-Flight Checks BEFORE editing**
- Catches authentication issues before wasting computation
- Validates environment and file states
- Tests push capability with --dry-run
- Each check has explicit error file creation on failure

**2. Validate AFTER each major step**
- Don't assume operations succeeded
- Check file states explicitly (size, emptiness, modifications)
- Parse git outputs for success/failure patterns
- Never skip validation steps - they exist because these failures have occurred

**3. Create `$HOME/error` on ANY failure**
- This is your ONLY reliable way to signal failures (exit codes don't work)
- Include detailed diagnostics using the provided templates
- Be specific about failure type and suggested fix
- Include exact commands, outputs, and diagnostic information

**4. Never skip validation steps**
- Each checkpoint prevents a specific known failure mode
- Pre-flight check 3 (push test) prevents wasted work on auth failures
- Step 3 validations prevent committing invalid states
- Step 5 validations catch push failures explicitly

**5. Report failures with extreme detail**
- Exact command that failed
- Complete error output (never summarize)
- Full diagnostic information (git status, file sizes, branch, etc.)
- Specific suggested fix for the failure type

### Workflow Summary

```
1. Pre-Flight Checks          → IF FAIL: Create $HOME/error, EXIT
   ├─ File validation
   ├─ Git configuration
   └─ Push capability test

2. Read BOOK.md               → Understand existing content

3. Plan contribution          → Decide what to add based on size

4. Edit BOOK.md              → Make changes

5. Validate edit              → IF FAIL: Restore files, create $HOME/error, EXIT
   ├─ Changes made
   ├─ Not empty
   ├─ Under limit
   ├─ README unchanged
   └─ No other files touched

6. Stage & Commit             → IF FAIL: Create $HOME/error, EXIT
   ├─ Only BOOK.md staged
   ├─ Commit succeeds
   └─ Directory clean

7. Push with validation       → IF FAIL: Create $HOME/error, EXIT
   ├─ Re-test capability
   ├─ Push to origin/main
   └─ Parse output for success

8. Final validation           → Confirm local/remote sync
```

**Remember**: The absence of `$HOME/error` is how success is detected. If ANY step fails, create the error file with comprehensive details and exit immediately. Don't try to recover or continue.

---

## TECHNICAL NOTES

### Execution Environment

- **Runtime**: Kubernetes CronJob (every 5-6 hours)
- **Command**: `claude --print $(cat README.md)`
- **Working Directory**: Fresh clone of repository
- **Git Config**: Author identity configured via CronJob environment
- **Authentication**: GitHub PAT in environment variable
- **Branch**: main (no feature branches)
- **Isolation**: Each run is independent, no state persists between runs

### Known Limitations

- **Exit Code Bug**: Claude Code CLI doesn't return non-zero exit codes reliably on failures
- **Workaround**: File-based error signaling via `$HOME/error`
- **Impact**: CronJob script must check for error file existence after claude command
- **No Retry**: Single-shot execution, must succeed on first attempt

### File Modification Rules

- **BOOK.md**: Read/Write - ONLY file that should be modified by automation
- **README.md**: Read-Only - NEVER modify (this file contains instructions)
- **LICENSE**: Read-Only - Do not modify
- **Other files**: Should not exist or be modified

### Success Indicators

1. No `$HOME/error` file exists after execution
2. New commit appears in `git log`
3. `git status` shows clean working directory
4. Push succeeded (commit visible on GitHub)
5. BOOK.md has been extended within theme

### CronJob Integration Example

```bash
#!/bin/bash
# Example CronJob script wrapper

set -e

# Clear error file from any previous run
rm -f $HOME/error

# Clone repository to temporary workspace
WORKSPACE=$(mktemp -d)
git clone git@github.com:Lillecarl/fullstopslop.git "$WORKSPACE"
cd "$WORKSPACE"

# Run Claude with README as prompt
claude --print "$(cat README.md)"

# Check for error file
if [ -f "$HOME/error" ]; then
  echo "ERROR: Claude execution failed"
  echo "======================================"
  cat "$HOME/error"
  echo "======================================"
  exit 1
fi

echo "SUCCESS: Claude execution completed"
echo "Latest commit: $(git log --oneline -1)"
exit 0
```

---

## Quick Reference: Decision Tree

```
START
  ↓
Pre-Flight Checks Pass?
  ├─ NO → Create $HOME/error → EXIT 1
  └─ YES → Continue
       ↓
Read BOOK.md & Plan
  ↓
Edit BOOK.md
  ↓
Validations Pass?
  ├─ NO → Restore files → Create $HOME/error → EXIT 1
  └─ YES → Continue
       ↓
Stage BOOK.md
  ↓
Commit
  ↓
Commit Success?
  ├─ NO → Create $HOME/error → EXIT 1
  └─ YES → Continue
       ↓
Push to origin/main
  ↓
Push Success?
  ├─ NO → Create $HOME/error → EXIT 1
  └─ YES → Continue
       ↓
Final Validation
  ↓
SUCCESS (no $HOME/error file)
```

---

**End of Instructions**
