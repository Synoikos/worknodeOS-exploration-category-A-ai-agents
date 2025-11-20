> how would 3. Git pre-commit hooks - Prevent violations mechanisms work exactly, and could I
tie any verification tool/mechanism to it for different kinds of violations? could it also
work for documentation, e.g. in case document strays from our allowed documentation structure?


● Excellent question! Let me break down how Git pre-commit hooks work and their full
  potential:

  🔧 How Git Pre-Commit Hooks Work

  Mechanism Overview

  Developer runs: git commit -m "message"
            ↓
  Git executes: .git/hooks/pre-commit (if exists)
            ↓
  Hook runs checks (exit code 0 = pass, non-zero = fail)
            ↓
  If PASS (exit 0):   Commit proceeds normally
  If FAIL (exit ≠0):  Commit BLOCKED, files unchanged

  Key Features:
  - Runs BEFORE commit is created (preventative, not reactive)
  - Has access to ALL staged files (git diff --cached)
  - Can run ANY executable (bash, python, ast-grep, valgrind, etc.)
  - Can modify files (auto-format code, generate docs)
  - Can be bypassed with git commit --no-verify (but you can warn users)

  ---
  📋 Full Pre-Commit Hook for DISTRIBUTED_SYSTEMS

  Complete Implementation

  Create .git/hooks/pre-commit:

  #!/bin/bash
  # DISTRIBUTED_SYSTEMS - NASA/JPL Compliance Pre-Commit Hook
  # Prevents ANY commit that violates Power of Ten rules or documentation standards

  set -e  # Exit on first error

  echo ""
  echo "═══════════════════════════════════════════════════════════"
  echo "🛡️  NASA/JPL COMPLIANCE PRE-COMMIT VERIFICATION"
  echo "═══════════════════════════════════════════════════════════"
  echo ""

  # Get list of staged C files
  STAGED_C_FILES=$(git diff --cached --name-only --diff-filter=ACM | grep '\.c$' || true)
  STAGED_H_FILES=$(git diff --cached --name-only --diff-filter=ACM | grep '\.h$' || true)
  STAGED_MD_FILES=$(git diff --cached --name-only --diff-filter=ACM | grep '\.md$' || true)

  VIOLATIONS=0

  # ═══════════════════════════════════════════════════════════
  # 1. CODE COMPLIANCE CHECKS (C files only)
  # ═══════════════════════════════════════════════════════════

  if [ -n "$STAGED_C_FILES" ]; then
    echo "📁 Checking staged C files:"
    echo "$STAGED_C_FILES" | sed 's/^/  - /'
    echo ""

    # ───────────────────────────────────────────────────────────
    # CHECK 1: No dynamic allocation (Power of Ten Rule 2)
    # ───────────────────────────────────────────────────────────
    echo "🔍 [1/8] Checking for malloc/free/realloc..."

    MALLOC_VIOLATIONS=$(echo "$STAGED_C_FILES" | xargs grep -n "malloc\|free\|realloc\|calloc"
   2>/dev/null || true)

    if [ -n "$MALLOC_VIOLATIONS" ]; then
      echo "❌ VIOLATION: Dynamic allocation detected (Power of Ten Rule 2)"
      echo "$MALLOC_VIOLATIONS" | sed 's/^/     /'
      VIOLATIONS=$((VIOLATIONS + 1))
    else
      echo "✅ No dynamic allocation"
    fi
    echo ""

    # ───────────────────────────────────────────────────────────
    # CHECK 2: No unbounded loops (Power of Ten Rule 3)
    # ───────────────────────────────────────────────────────────
    echo "🔍 [2/8] Checking for unbounded loops (ast-grep)..."

    UNBOUNDED_WHILE=$(~/bin/ast-grep --pattern 'while (true) { $$$ }' $STAGED_C_FILES
  2>/dev/null || true)
    UNBOUNDED_WHILE_1=$(~/bin/ast-grep --pattern 'while (1) { $$$ }' $STAGED_C_FILES
  2>/dev/null || true)
    UNBOUNDED_FOR=$(~/bin/ast-grep --pattern 'for (;;) { $$$ }' $STAGED_C_FILES 2>/dev/null ||
   true)

    if [ -n "$UNBOUNDED_WHILE" ] || [ -n "$UNBOUNDED_WHILE_1" ] || [ -n "$UNBOUNDED_FOR" ];
  then
      echo "❌ VIOLATION: Unbounded loops detected (Power of Ten Rule 3)"
      [ -n "$UNBOUNDED_WHILE" ] && echo "$UNBOUNDED_WHILE" | sed 's/^/     /'
      [ -n "$UNBOUNDED_WHILE_1" ] && echo "$UNBOUNDED_WHILE_1" | sed 's/^/     /'
      [ -n "$UNBOUNDED_FOR" ] && echo "$UNBOUNDED_FOR" | sed 's/^/     /'
      VIOLATIONS=$((VIOLATIONS + 1))
    else
      echo "✅ All loops bounded"
    fi
    echo ""

    # ───────────────────────────────────────────────────────────
    # CHECK 3: No recursion (Power of Ten Rule 1)
    # ───────────────────────────────────────────────────────────
    echo "🔍 [3/8] Checking for recursion (ast-grep)..."

    # Extract function names from staged files
    FUNCTIONS=$(grep -h "^[a-zA-Z_][a-zA-Z0-9_]* [a-zA-Z_][a-zA-Z0-9_]*(" $STAGED_C_FILES |
  awk '{print $2}' | cut -d'(' -f1 || true)

    RECURSION_FOUND=false
    for func in $FUNCTIONS; do
      # Check if function calls itself
      RECURSIVE=$(grep -n "$func(" $STAGED_C_FILES | grep -v "^[[:space:]]*//.*$func(" | tail
  -n +2 || true)
      if [ -n "$RECURSIVE" ]; then
        echo "❌ VIOLATION: Recursion detected in $func()"
        echo "$RECURSIVE" | sed 's/^/     /'
        RECURSION_FOUND=true
      fi
    done

    if [ "$RECURSION_FOUND" = true ]; then
      VIOLATIONS=$((VIOLATIONS + 1))
    else
      echo "✅ No recursion detected"
    fi
    echo ""

    # ───────────────────────────────────────────────────────────
    # CHECK 4: All functions return Result type (convention)
    # ───────────────────────────────────────────────────────────
    echo "🔍 [4/8] Checking function return types..."

    NON_RESULT=$(grep -n "^void [a-zA-Z_][a-zA-Z0-9_]*(" $STAGED_C_FILES | grep -v "static" ||
   true)

    if [ -n "$NON_RESULT" ]; then
      echo "⚠️  WARNING: Non-Result return types detected (NASA best practice)"
      echo "$NON_RESULT" | sed 's/^/     /'
      # Not a blocker, just warning
    else
      echo "✅ All public functions return Result"
    fi
    echo ""

    # ───────────────────────────────────────────────────────────
    # CHECK 5: Complexity analysis (pmccabe)
    # ───────────────────────────────────────────────────────────
    echo "🔍 [5/8] Checking cyclomatic complexity..."

    HIGH_COMPLEXITY=$(pmccabe $STAGED_C_FILES 2>/dev/null | awk '$1 > 10 {print $5 ":" $1 "
  (complexity)"}' || true)

    if [ -n "$HIGH_COMPLEXITY" ]; then
      echo "❌ VIOLATION: Complexity >10 detected (NASA A+ requirement)"
      echo "$HIGH_COMPLEXITY" | sed 's/^/     /'
      VIOLATIONS=$((VIOLATIONS + 1))
    else
      echo "✅ All functions ≤10 complexity"
    fi
    echo ""

    # ───────────────────────────────────────────────────────────
    # CHECK 6: Static analysis (cppcheck)
    # ───────────────────────────────────────────────────────────
    echo "🔍 [6/8] Running static analysis (cppcheck)..."

    CPPCHECK_ERRORS=$(cppcheck --quiet --enable=all --error-exitcode=1 \
      --suppress=missingIncludeSystem \
      $STAGED_C_FILES 2>&1 || true)

    if [ -n "$CPPCHECK_ERRORS" ]; then
      echo "⚠️  WARNING: Static analysis issues detected"
      echo "$CPPCHECK_ERRORS" | sed 's/^/     /'
      # Not blocking, but informational
    else
      echo "✅ No static analysis issues"
    fi
    echo ""

    # ───────────────────────────────────────────────────────────
    # CHECK 7: NULL pointer checks (ast-grep)
    # ───────────────────────────────────────────────────────────
    echo "🔍 [7/8] Checking for NULL checks on pointers..."

    # Find pointer dereferences without NULL checks (simplified heuristic)
    MISSING_NULL_CHECKS=$(grep -n "->.*=" $STAGED_C_FILES | grep -v "if.*NULL" | head -5 ||
  true)

    if [ -n "$MISSING_NULL_CHECKS" ]; then
      echo "⚠️  WARNING: Potential missing NULL checks (first 5 shown)"
      echo "$MISSING_NULL_CHECKS" | sed 's/^/     /'
    else
      echo "✅ Pointer usage looks safe"
    fi
    echo ""

    # ───────────────────────────────────────────────────────────
    # CHECK 8: Compilation test (gcc)
    # ───────────────────────────────────────────────────────────
    echo "🔍 [8/8] Compilation test (gcc -fsyntax-only)..."

    COMPILE_ERRORS=""
    for file in $STAGED_C_FILES; do
      ERRORS=$(wsl -d Ubuntu-22.04 bash -c "cd ~/workspace/DISTRIBUTED_SYSTEMS && gcc -std=c11
   -Wall -Wextra -Werror -fsyntax-only -Iinclude -D_POSIX_C_SOURCE=200809L $file 2>&1" ||
  true)
      if [ -n "$ERRORS" ]; then
        COMPILE_ERRORS="$COMPILE_ERRORS\n$file:\n$ERRORS"
      fi
    done

    if [ -n "$COMPILE_ERRORS" ]; then
      echo "❌ VIOLATION: Compilation errors detected"
      echo -e "$COMPILE_ERRORS" | sed 's/^/     /'
      VIOLATIONS=$((VIOLATIONS + 1))
    else
      echo "✅ All files compile"
    fi
    echo ""
  fi

  # ═══════════════════════════════════════════════════════════
  # 2. DOCUMENTATION COMPLIANCE CHECKS (Markdown files)
  # ═══════════════════════════════════════════════════════════

  if [ -n "$STAGED_MD_FILES" ]; then
    echo "📁 Checking staged Markdown files:"
    echo "$STAGED_MD_FILES" | sed 's/^/  - /'
    echo ""

    # ───────────────────────────────────────────────────────────
    # CHECK 9: Documentation structure validation
    # ───────────────────────────────────────────────────────────
    echo "🔍 [9/12] Validating documentation structure..."

    for md_file in $STAGED_MD_FILES; do
      case "$md_file" in
        # Agent handoff files MUST have required sections
        .agent-handoffs/*-COMPLETE.md)
          echo "  Checking agent handoff: $md_file"

          MISSING_SECTIONS=""
          grep -q "## What Was Implemented" "$md_file" ||
  MISSING_SECTIONS="$MISSING_SECTIONS\n  - Missing: ## What Was Implemented"
          grep -q "## Critical Decisions" "$md_file" || MISSING_SECTIONS="$MISSING_SECTIONS\n
   - Missing: ## Critical Decisions"
          grep -q "## Dependencies" "$md_file" || MISSING_SECTIONS="$MISSING_SECTIONS\n  -
  Missing: ## Dependencies"
          grep -q "## Blockers" "$md_file" || MISSING_SECTIONS="$MISSING_SECTIONS\n  -
  Missing: ## Blockers"

          if [ -n "$MISSING_SECTIONS" ]; then
            echo "❌ VIOLATION: $md_file missing required sections"
            echo -e "$MISSING_SECTIONS"
            VIOLATIONS=$((VIOLATIONS + 1))
          else
            echo "  ✅ All required sections present"
          fi
          ;;

        # Session handoff files
        SESSION_BOOTSTRAP.md)
          echo "  Checking SESSION_BOOTSTRAP.md structure"

          grep -q "## 🚨 CURRENT SESSION STATE" "$md_file" || {
            echo "❌ VIOLATION: Missing ## 🚨 CURRENT SESSION STATE"
            VIOLATIONS=$((VIOLATIONS + 1))
          }
          grep -q "## 📍 CURRENT STATE" "$md_file" || {
            echo "❌ VIOLATION: Missing ## 📍 CURRENT STATE"
            VIOLATIONS=$((VIOLATIONS + 1))
          }
          ;;

        # Implementation log
        IMPLEMENTATION_LOG.md)
          echo "  Checking IMPLEMENTATION_LOG.md structure"

          # Check for session entries with date format
          grep -q "## Session [0-9]\{4\}-[0-9]\{2\}-[0-9]\{2\}" "$md_file" || {
            echo "⚠️  WARNING: No dated session entries found"
          }
          ;;

        # NASA compliance status
        NASA_COMPLIANCE_STATUS.md)
          echo "  Checking NASA_COMPLIANCE_STATUS.md"

          grep -q "Current compliance:" "$md_file" || {
            echo "❌ VIOLATION: Missing compliance grade"
            VIOLATIONS=$((VIOLATIONS + 1))
          }
          ;;
      esac
    done
    echo ""

    # ───────────────────────────────────────────────────────────
    # CHECK 10: No time estimates (your constraint)
    # ───────────────────────────────────────────────────────────
    echo "🔍 [10/12] Checking for forbidden time estimates..."

    TIME_ESTIMATES=$(grep -n "Week [0-9]\|Phase [0-9] (.*hours)\|[0-9]+ hours\|[0-9]+ days"
  $STAGED_MD_FILES || true)

    if [ -n "$TIME_ESTIMATES" ]; then
      echo "❌ VIOLATION: Time estimates detected (forbidden per user constraint)"
      echo "$TIME_ESTIMATES" | sed 's/^/     /'
      VIOLATIONS=$((VIOLATIONS + 1))
    else
      echo "✅ No time estimates"
    fi
    echo ""

    # ───────────────────────────────────────────────────────────
    # CHECK 11: Markdown formatting (markdownlint - optional)
    # ───────────────────────────────────────────────────────────
    echo "🔍 [11/12] Checking Markdown formatting..."

    # Check for common issues without external tool
    FORMATTING_ISSUES=""

    for md_file in $STAGED_MD_FILES; do
      # Check for trailing whitespace
      TRAILING=$(grep -n " $" "$md_file" || true)
      if [ -n "$TRAILING" ]; then
        FORMATTING_ISSUES="$FORMATTING_ISSUES\n  $md_file: trailing whitespace"
      fi

      # Check for inconsistent heading levels (should not skip levels)
      # (simplified check)
    done

    if [ -n "$FORMATTING_ISSUES" ]; then
      echo "⚠️  WARNING: Formatting issues detected"
      echo -e "$FORMATTING_ISSUES"
    else
      echo "✅ Markdown formatting OK"
    fi
    echo ""

    # ───────────────────────────────────────────────────────────
    # CHECK 12: File references exist
    # ───────────────────────────────────────────────────────────
    echo "🔍 [12/12] Validating file references in documentation..."

    BROKEN_REFS=""

    for md_file in $STAGED_MD_FILES; do
      # Extract file references like `src/core/file.c`
      REFS=$(grep -o '`[^`]*\.[ch]`' "$md_file" | tr -d '`' || true)

      for ref in $REFS; do
        if [ ! -f "$ref" ]; then
          BROKEN_REFS="$BROKEN_REFS\n  $md_file references missing: $ref"
        fi
      done
    done

    if [ -n "$BROKEN_REFS" ]; then
      echo "⚠️  WARNING: Broken file references detected"
      echo -e "$BROKEN_REFS"
    else
      echo "✅ All file references valid"
    fi
    echo ""
  fi

  # ═══════════════════════════════════════════════════════════
  # 3. FINAL VERDICT
  # ═══════════════════════════════════════════════════════════

  echo "═══════════════════════════════════════════════════════════"
  if [ $VIOLATIONS -eq 0 ]; then
    echo "✅ ALL CHECKS PASSED - Commit allowed"
    echo "═══════════════════════════════════════════════════════════"
    echo ""
    exit 0
  else
    echo "❌ $VIOLATIONS VIOLATION(S) DETECTED - Commit BLOCKED"
    echo "═══════════════════════════════════════════════════════════"
    echo ""
    echo "Fix violations and try again, or bypass with: git commit --no-verify"
    echo "(Bypassing is NOT recommended for NASA-compliant code)"
    echo ""
    exit 1
  fi

  ---
  🔌 Extensibility - Tying Additional Tools

  Yes! You can tie ANY verification tool. Examples:

  1. Add Valgrind Memory Check

  # In pre-commit hook, add:
  echo "🔍 Running Valgrind on test suite..."
  valgrind --leak-check=full --error-exitcode=1 ./build/bin/test_phase7_pm 2>&1 > /dev/null
  if [ $? -ne 0 ]; then
    echo "❌ VIOLATION: Memory errors detected"
    VIOLATIONS=$((VIOLATIONS + 1))
  fi

  2. Add Custom Python Validator

  # Check JSON files for valid schema
  if [ -n "$STAGED_JSON_FILES" ]; then
    python3 scripts/validate_json_schema.py $STAGED_JSON_FILES || {
      echo "❌ VIOLATION: Invalid JSON schema"
      VIOLATIONS=$((VIOLATIONS + 1))
    }
  fi

  3. Add Custom Documentation Rules

  # Enforce specific terminology
  FORBIDDEN_TERMS=$(grep -in "just\|simply\|obviously\|clearly" $STAGED_MD_FILES || true)
  if [ -n "$FORBIDDEN_TERMS" ]; then
    echo "⚠️  WARNING: Avoid weak qualifiers (just/simply/obviously)"
    echo "$FORBIDDEN_TERMS"
  fi

  4. Add SPIN Model Checker

  # Run SPIN verification on Promela models
  if [ -n "$STAGED_PML_FILES" ]; then
    for pml in $STAGED_PML_FILES; do
      spin -a $pml && gcc -o pan pan.c && ./pan || {
        echo "❌ VIOLATION: SPIN model verification failed for $pml"
        VIOLATIONS=$((VIOLATIONS + 1))
      }
    done
  fi

  5. Add License Header Check

  # Ensure all C files have NASA-approved license header
  for file in $STAGED_C_FILES; do
    head -n 5 "$file" | grep -q "NASA/JPL" || {
      echo "⚠️  WARNING: Missing NASA license header in $file"
    }
  done

  ---
  📚 Documentation Structure Enforcement - Full Example

  Enforcing Agent Handoff Structure

  Create scripts/validate_agent_handoff.sh:

  #!/bin/bash
  # Validates agent handoff document structure

  FILE=$1

  # Required sections for agent handoffs
  REQUIRED_SECTIONS=(
    "## What Was Implemented"
    "## Critical Decisions Made"
    "## Dependencies for Next Agent"
    "## Blockers/Issues"
    "## Files Created/Modified"
    "## Tests Written"
    "## Integration Points"
  )

  VIOLATIONS=0

  echo "Validating agent handoff: $FILE"

  for section in "${REQUIRED_SECTIONS[@]}"; do
    if ! grep -q "^$section" "$FILE"; then
      echo "❌ Missing section: $section"
      VIOLATIONS=$((VIOLATIONS + 1))
    fi
  done

  # Check for specific content
  if ! grep -q "```" "$FILE"; then
    echo "⚠️  WARNING: No code blocks found (expected for implementations)"
  fi

  # Check for file references
  if ! grep -q '`.*\.[ch]`' "$FILE"; then
    echo "⚠️  WARNING: No file references found"
  fi

  # Check for test results
  if ! grep -qE "[0-9]+/[0-9]+ (tests )?pass" "$FILE"; then
    echo "⚠️  WARNING: No test results found (X/Y pass format)"
  fi

  exit $VIOLATIONS

  In pre-commit hook:

  # Validate agent handoffs
  for handoff in $STAGED_MD_FILES; do
    if [[ "$handoff" == .agent-handoffs/*-COMPLETE.md ]]; then
      ./scripts/validate_agent_handoff.sh "$handoff" || VIOLATIONS=$((VIOLATIONS + 1))
    fi
  done

  ---
  🎨 Advanced: Custom Documentation Linting

  Example: Enforce Consistent Terminology

  Create scripts/doc_style_check.sh:

  #!/bin/bash
  # Enforce consistent terminology in documentation

  FILE=$1

  # Forbidden terms (use preferred alternatives)
  STYLE_VIOLATIONS=""

  # Check for inconsistent terminology
  grep -in "work node" "$FILE" && STYLE_VIOLATIONS="$STYLE_VIOLATIONS\n  Use 'Worknode' not
  'work node'"
  grep -in "power of 10" "$FILE" && STYLE_VIOLATIONS="$STYLE_VIOLATIONS\n  Use 'Power of Ten'
  not 'power of 10'"
  grep -in "nasa jpl" "$FILE" && STYLE_VIOLATIONS="$STYLE_VIOLATIONS\n  Use 'NASA/JPL' not
  'nasa jpl'"

  # Check for required capitalization
  grep -E "(phase [0-9])" "$FILE" && STYLE_VIOLATIONS="$STYLE_VIOLATIONS\n  Use 'Phase N' not
  'phase n'"

  # Check heading format
  grep "^##[^ ]" "$FILE" && STYLE_VIOLATIONS="$STYLE_VIOLATIONS\n  Headings must have space
  after ##"

  if [ -n "$STYLE_VIOLATIONS" ]; then
    echo "⚠️  Style violations in $FILE:"
    echo -e "$STYLE_VIOLATIONS"
    return 1
  fi

  ---
  ⚡ Performance Optimization

  Problem: Pre-commit hooks slow down commits

  Solutions:

  1. Check Only Staged Files (already doing this)

  # Good: Only check changed files
  STAGED_FILES=$(git diff --cached --name-only)

  # Bad: Check entire codebase every commit
  # SLOW!

  2. Parallel Execution

  # Run independent checks in parallel
  (check_malloc &)
  (check_complexity &)
  (check_docs &)
  wait  # Wait for all background jobs

  3. Caching Results

  # Cache expensive checks (e.g., cppcheck results)
  HASH=$(md5sum $FILE)
  if [ -f ".cache/$HASH.result" ]; then
    cat ".cache/$HASH.result"
  else
    cppcheck $FILE > ".cache/$HASH.result"
  fi

  4. Skip on WIP Commits

  # Skip checks for WIP commits
  if git log -1 --pretty=%B | grep -q "WIP:"; then
    echo "⚠️  WIP commit - skipping checks (use 'git commit --amend' later)"
    exit 0
  fi

  ---
  🛠️ Installation & Usage

  Quick Install:

  # Copy pre-commit hook
  cp scripts/pre-commit.sh .git/hooks/pre-commit
  chmod +x .git/hooks/pre-commit

  # Test it
  git add src/core/allocator.c
  git commit -m "test"  # Will run all checks

  Bypass When Needed:

  # Emergency bypass (not recommended)
  git commit --no-verify -m "hotfix"

  Team Enforcement:

  # Prevent bypass (check in commit-msg hook)
  if git log -1 --pretty=%B | grep -q "EMERGENCY"; then
    echo "✅ Emergency bypass approved"
  else
    # Run full checks even if --no-verify used
  fi

  ---
  📊 Summary

  Can you tie ANY verification tool? → YES!

  Examples integrated:
  - ✅ ast-grep (syntax-aware)
  - ✅ cppcheck (static analysis)
  - ✅ pmccabe (complexity)
  - ✅ gcc (compilation)
  - ✅ Custom scripts (documentation structure)
  - ✅ grep (pattern matching)
  - ⚡ Valgrind (add easily)
  - ⚡ SPIN (add easily)
  - ⚡ Custom Python validators (add easily)

  Documentation structure enforcement? → Absolutely!
  - Required sections check
  - Terminology consistency
  - Markdown formatting
  - File reference validation
  - Anti-patterns detection

  Want me to create the full pre-commit hook + validation scripts now?
  ⎿  Stop hook succeeded:

> but wait, if we agent refactors files and then commits, the files are already changed, no?
how would the pre-hook and commit then work? would it mean committing to git, so that even if
the files were changed, the new version in git would also be changed?

● EXCELLENT catch! You've identified a critical workflow issue. Let me clarify:

  🔄 Git Workflow: When Pre-Commit Hooks Actually Run

  The Git Commit Flow

  1. Agent modifies files on disk
     ↓
  2. Files are in "working directory" (uncommitted)
     ↓
  3. You run: git add <files>
     ↓
  4. Files are now "staged" (ready to commit)
     ↓
  5. You run: git commit -m "message"
     ↓
  6. **PRE-COMMIT HOOK RUNS HERE** ← Checks staged files
     ↓
  7a. If hook PASSES (exit 0):
      → Commit is created in Git history
      → Files are now in repository

  7b. If hook FAILS (exit ≠0):
      → Commit is BLOCKED
      → Files remain staged but NOT committed
      → Working directory files are UNCHANGED

  ---
  🎯 The Key Point: Pre-Commit Hooks Don't Prevent File Changes

  What pre-commit hooks DO:
  - ✅ Prevent BAD code from entering Git history
  - ✅ Block commits with violations
  - ✅ Force you to fix issues before committing

  What pre-commit hooks DON'T do:
  - ❌ Prevent agents from writing bad code to disk
  - ❌ Revert file changes if hook fails
  - ❌ Stop agents from modifying files

  ---
  🚨 The Real Problem: Agent Already Wrote Bad Code

  Scenario 1: Agent Violates Rules

  # Agent writes code with malloc (violation)
  # Agent creates: src/core/bad_code.c (on disk)

  git add src/core/bad_code.c
  git commit -m "Agent work"

  # Pre-commit hook runs:
  ❌ VIOLATION: malloc detected in src/core/bad_code.c
  Commit BLOCKED

  # Current state:
  - bad_code.c still exists on disk (UNCHANGED)
  - bad_code.c is staged (ready to commit)
  - No commit created in Git history
  - You must fix bad_code.c manually, then re-commit

  Problem: You now have to manually fix what the agent broke!

  ---
  ✅ Solution 1: Pre-Flight Validation (BEFORE Agent Writes)

  Strategy: Validate agent's PROPOSED changes before they write to disk

  Implementation: Agent Review Workflow

  # Agent workflow with validation:

  1. Agent writes code to TEMP location first
     → /tmp/agent_workspace/src/core/allocator.c

  2. Run pre-commit checks on TEMP files
     → scripts/validate_code.sh /tmp/agent_workspace/

  3a. If validation PASSES:
      → Copy files to actual project
      → git add + git commit (will pass hook)

  3b. If validation FAILS:
      → Reject agent output
      → DO NOT copy to project
      → Ask agent to fix issues

  Script: scripts/validate_agent_output.sh

  #!/bin/bash
  # Validate agent output BEFORE accepting it

  AGENT_WORKSPACE=$1  # e.g., /tmp/agent_workspace/

  echo "🔍 Validating agent output (pre-flight check)..."

  # Copy all validation checks from pre-commit hook
  # but run on AGENT_WORKSPACE instead of repo

  VIOLATIONS=0

  # Check 1: No malloc
  if grep -r "malloc\|free\|realloc" "$AGENT_WORKSPACE"/*.c; then
    echo "❌ Agent used malloc - REJECTING output"
    VIOLATIONS=$((VIOLATIONS + 1))
  fi

  # Check 2: No unbounded loops
  if ~/bin/ast-grep --pattern 'while (true) { $$$ }' "$AGENT_WORKSPACE"/*.c; then
    echo "❌ Agent used unbounded loops - REJECTING output"
    VIOLATIONS=$((VIOLATIONS + 1))
  fi

  # Check 3: Complexity
  HIGH_COMPLEXITY=$(pmccabe "$AGENT_WORKSPACE"/*.c | awk '$1 > 10')
  if [ -n "$HIGH_COMPLEXITY" ]; then
    echo "❌ Agent created high-complexity functions - REJECTING output"
    VIOLATIONS=$((VIOLATIONS + 1))
  fi

  if [ $VIOLATIONS -eq 0 ]; then
    echo "✅ Agent output validated - SAFE to accept"
    exit 0
  else
    echo "❌ Agent output has $VIOLATIONS violations - DO NOT ACCEPT"
    exit 1
  fi

  Usage in Agent Instructions:

  Task("Implement allocator.c",
       "READ: SESSION_BOOTSTRAP.md, NASA_COMPLIANCE.md

       WORKFLOW:
       1. Write code to /tmp/agent_workspace/
       2. Run: scripts/validate_agent_output.sh /tmp/agent_workspace/
       3. If validation passes:
          - Copy files to src/core/
          - Run tests
          - Create handoff
       4. If validation fails:
          - FIX issues in /tmp workspace
          - Re-run validation
          - DO NOT copy to project until passes

       DELIVERABLE:
       - Pre-validated code (passed validation script)
       - .agent-handoffs/allocator-COMPLETE.md",
       "general-purpose")

  ---
  ✅ Solution 2: Post-Write Validation + Auto-Rollback

  Strategy: Agent writes directly, but we validate and rollback if bad

  Implementation: Git Stash + Validate

  # Workflow:
  1. Create git checkpoint BEFORE agent starts
     → git commit -m "Pre-agent checkpoint"

  2. Agent writes code directly to project
     → src/core/allocator.c modified

  3. Validate changes
     → scripts/validate_changes.sh

  4a. If valid:
      → git add + git commit (will pass pre-commit hook)

  4b. If invalid:
      → git reset --hard HEAD (rollback to checkpoint)
      → Agent must try again

  Script: scripts/validate_and_commit_agent_work.sh

  #!/bin/bash
  # Validate agent changes and commit if clean, rollback if bad

  AGENT_NAME=$1
  COMMIT_MSG=$2

  echo "🔍 Validating agent work: $AGENT_NAME"

  # Get list of modified files
  MODIFIED_FILES=$(git diff --name-only)

  if [ -z "$MODIFIED_FILES" ]; then
    echo "⚠️  No changes detected"
    exit 1
  fi

  echo "📁 Files modified:"
  echo "$MODIFIED_FILES"
  echo ""

  # Run validation (same as pre-commit checks)
  VIOLATIONS=0

  # Check for violations
  C_FILES=$(echo "$MODIFIED_FILES" | grep '\.c$')
  if [ -n "$C_FILES" ]; then
    # Run all NASA compliance checks
    if grep -r "malloc\|free" $C_FILES; then
      echo "❌ malloc detected"
      VIOLATIONS=$((VIOLATIONS + 1))
    fi

    # ... (all other checks from pre-commit hook)
  fi

  if [ $VIOLATIONS -eq 0 ]; then
    echo "✅ Validation passed - committing agent work"
    git add -A
    git commit -m "$COMMIT_MSG"
    echo "✅ Committed: $COMMIT_MSG"
    exit 0
  else
    echo "❌ Validation failed - rolling back changes"
    git reset --hard HEAD
    git clean -fd
    echo "❌ Changes rolled back - agent must fix violations"
    exit 1
  fi

  Usage:

  # Before agent starts
  git commit -m "Pre-agent checkpoint"

  # Agent writes code (directly to project)
  # ... agent modifies files ...

  # Validate and commit (or rollback)
  ./scripts/validate_and_commit_agent_work.sh "Agent A" "Implement allocator.c"

  # Result:
  # - If valid: Changes committed
  # - If invalid: Changes DELETED, back to checkpoint

  ---
  ✅ Solution 3: Hybrid Approach (Recommended)

  Best of both worlds: Agent self-validates BEFORE we accept

  Agent Instructions with Self-Validation:

  Task("Implement allocator.c - NASA Compliant",
       `
       ═══════════════════════════════════════════════════════════
       🚨 MANDATORY SELF-VALIDATION BEFORE COMPLETION
       ═══════════════════════════════════════════════════════════

       You MUST run these checks on YOUR code before marking complete:

       1. Run: grep -r "malloc\|free\|realloc" [your files]
          Expected: 0 matches
          If FAIL: Fix before proceeding

       2. Run: ~/bin/ast-grep --pattern 'while (true) { $$$ }' [your files]
          Expected: 0 matches
          If FAIL: Fix before proceeding

       3. Run: pmccabe [your files] | awk '$1 > 10'
          Expected: 0 matches
          If FAIL: Refactor before proceeding

       4. Run: gcc -std=c11 -Wall -Wextra -Werror -fsyntax-only [your files]
          Expected: No errors
          If FAIL: Fix before proceeding

       ═══════════════════════════════════════════════════════════
       🎯 IMPLEMENTATION TASK
       ═══════════════════════════════════════════════════════════

       Create src/core/allocator.c (pool-based allocator)

       ═══════════════════════════════════════════════════════════
       ✅ COMPLETION CRITERIA
       ═══════════════════════════════════════════════════════════

       DO NOT mark complete until:
       □ All 4 self-validation checks pass
       □ Tests written and passing
       □ Handoff document created

       Report validation results in handoff document.
       `,
       "general-purpose")

  In Coordinator (You) After Agent Completes:

  # Agent claims completion
  # You verify:

  # 1. Read agent handoff
  cat .agent-handoffs/allocator-COMPLETE.md

  # 2. Check if agent reported validation results
  grep "Self-Validation Results" .agent-handoffs/allocator-COMPLETE.md

  # 3. Re-run validation yourself (trust but verify)
  ./scripts/validate_agent_output.sh src/core/

  # 4a. If passes:
  git add src/core/allocator.c
  git commit -m "Agent A: Implement pool allocator (validated)"

  # 4b. If fails:
  echo "❌ Agent validation was incomplete - rejecting work"
  git restore src/core/allocator.c
  # Spawn new agent with stricter instructions

  ---
  📊 Comparison: When Each Solution Works Best

  | Solution                    | When to Use                | Pros                       |
  Cons                             |
  |-----------------------------|----------------------------|----------------------------|---
  -------------------------------|
  | Pre-flight (temp workspace) | Don't trust agent          | No bad code touches repo   |
  Complex workflow, extra copying  |
  | Post-write + rollback       | Trust agent mostly         | Simple workflow            |
  Agent wasted time if rejected    |
  | Self-validation             | Agent follows instructions | Agent catches own mistakes |
  Requires compliant agent         |
  | Pre-commit hook only        | Manual commits             | Prevents bad commits       |
  Doesn't prevent bad code on disk |

  ---
  🎯 Recommended Workflow for DISTRIBUTED_SYSTEMS

  Hybrid: Self-Validation + Pre-Commit Hook + Manual Review

  1. Agent writes code (directly to project)
     ↓
  2. Agent runs self-validation (in task instructions)
     ↓
  3. Agent reports results in handoff document
     ↓
  4. You (coordinator) review handoff
     ↓
  5. You re-run validation (trust but verify)
     ↓
  6a. If validation passes:
      → git add files
      → git commit (pre-commit hook runs as final gate)
      → Commit succeeds

  6b. If validation fails:
      → git restore files (undo agent changes)
      → Spawn new agent with stricter instructions
      → Document what went wrong

  Why This Works:

  1. Agent self-validation: Catches 80% of issues immediately
  2. Manual re-validation: Catches agent laziness/mistakes
  3. Pre-commit hook: Final safety net (prevents human error too)
  4. Git history stays clean: Only validated code gets committed

  ---
  🛠️ Quick Setup

  Want me to create:

  1. ✅ Pre-commit hook (final safety net)
  2. ✅ Agent self-validation script (agent runs this)
  3. ✅ Manual validation script (you run after agent)
  4. ✅ Updated agent instruction template (includes self-validation)

  This gives you three layers of defense:
  - Layer 1: Agent self-validates
  - Layer 2: You manually re-validate
  - Layer 3: Git pre-commit hook blocks bad commits

  Should I implement this now?
  ⎿  Stop hook succeeded:

> ok but could we have git prehook compliance verification trigger-like mechanism as feedback
loop for agents? But without me having to trigger or commit anything while agent works? is
this distinct from claude hooks btw? what are claude hooks and could we use em somehow?

● BRILLIANT question! You want automated feedback loops for agents WITHOUT manual
  intervention. Let me explain both mechanisms:

  🔄 Git Pre-Commit Hooks vs Claude Code Hooks

  Git Pre-Commit Hooks

  - Trigger: When YOU run git commit
  - Scope: Repository-level validation
  - Problem: Requires manual git commit to trigger
  - ❌ Can't auto-trigger during agent work

  Claude Code Hooks

  - Trigger: When Claude uses tools (Edit, Write, Bash, etc.)
  - Scope: Claude-level validation (any tool use)
  - Configured: ~/.claude/settings.json
  - ✅ Can auto-trigger during agent work!

  ---
  🎯 Claude Code Hooks - The Missing Piece!

  What Are Claude Code Hooks?

  Event interceptors that run shell commands when Claude performs actions:

  // ~/.claude/settings.json
  {
    "hooks": {
      "PreToolUse": {
        "match": { "tool": ["Edit", "Write"] },
        "command": "scripts/validate_file_before_write.sh",
        "description": "Validate code before Claude writes it"
      },
      "PostToolUse": {
        "match": { "tool": ["Edit", "Write"] },
        "command": "scripts/check_nasa_compliance.sh",
        "description": "Check NASA compliance after file modification"
      }
    }
  }

  Available Hook Events:

  | Hook Event       | When It Fires                        | Use Case
          |
  |------------------|--------------------------------------|---------------------------------
  --------|
  | UserPromptSubmit | Before Claude processes user message | Inject context, load memory
          |
  | PreToolUse       | Before ANY tool executes             | Validate inputs, check
  preconditions    |
  | PostToolUse      | After tool completes                 | Validate outputs, run compliance
   checks |
  | Stop             | When Claude finishes response        | Cleanup, session logging
          |
  | SubagentStop     | When subagent (Task tool) finishes   | Validate agent deliverables
          |

  ---
  🚀 Solution: Claude Hooks as Agent Feedback Loop

  Automatic Compliance Validation During Agent Work

  // ~/.claude/settings.json
  {
    "hooks": {
      // ════════════════════════════════════════════════════════
      // HOOK 1: Validate BEFORE agent writes code
      // ════════════════════════════════════════════════════════
      "PreToolUse": {
        "match": {
          "tool": ["Write", "Edit"],
          "path": "**/*.c"  // Only C files
        },
        "command": "bash -c 'echo \"$CLAUDE_TOOL_INPUT\" | python3
  scripts/validate_code_before_write.py'",
        "description": "Pre-validate C code before writing"
      },

      // ════════════════════════════════════════════════════════
      // HOOK 2: Validate AFTER agent writes code
      // ════════════════════════════════════════════════════════
      "PostToolUse": {
        "match": {
          "tool": ["Write", "Edit"],
          "path": "**/*.c"
        },
        "command": "scripts/check_nasa_compliance.sh \"$CLAUDE_TOOL_ARGS_file_path\"",
        "description": "Check NASA compliance after file write",
        "blocking": true  // BLOCK Claude if validation fails
      },

      // ════════════════════════════════════════════════════════
      // HOOK 3: Validate agent deliverables when subagent stops
      // ════════════════════════════════════════════════════════
      "SubagentStop": {
        "match": { "type": "all" },
        "command": "scripts/validate_agent_handoff.sh",
        "description": "Validate agent completed all requirements"
      },

      // ════════════════════════════════════════════════════════
      // HOOK 4: Auto-inject NASA compliance context
      // ════════════════════════════════════════════════════════
      "UserPromptSubmit": {
        "match": { "type": "all" },
        "command": "scripts/inject_nasa_context.sh",
        "description": "Auto-add NASA compliance reminders"
      }
    }
  }

  ---
  🛠️ Implementation: Automatic Validation Loop

  Script 1: scripts/check_nasa_compliance.sh

  This runs AUTOMATICALLY after EVERY Edit or Write tool use:

  #!/bin/bash
  # Auto-runs after Claude writes/edits C files
  # Blocks Claude if violations detected

  FILE_PATH=$1

  echo "🔍 Auto-validating: $FILE_PATH" >&2

  # Only check C files
  if [[ ! "$FILE_PATH" =~ \.c$ ]]; then
    exit 0
  fi

  VIOLATIONS=0

  # ═══════════════════════════════════════════════════════════
  # CHECK 1: No malloc/free (Power of Ten Rule 2)
  # ═══════════════════════════════════════════════════════════
  if grep -q "malloc\|free\|realloc" "$FILE_PATH"; then
    echo "❌ HOOK VIOLATION: malloc detected in $FILE_PATH" >&2
    echo "   Claude must rewrite without dynamic allocation" >&2
    VIOLATIONS=$((VIOLATIONS + 1))
  fi

  # ═══════════════════════════════════════════════════════════
  # CHECK 2: No unbounded loops (Power of Ten Rule 3)
  # ═══════════════════════════════════════════════════════════
  if ~/bin/ast-grep --pattern 'while (true) { $$$ }' "$FILE_PATH" 2>/dev/null | grep -q "";
  then
    echo "❌ HOOK VIOLATION: Unbounded loop in $FILE_PATH" >&2
    VIOLATIONS=$((VIOLATIONS + 1))
  fi

  # ═══════════════════════════════════════════════════════════
  # CHECK 3: Complexity check (Power of Ten Rule 10)
  # ═══════════════════════════════════════════════════════════
  HIGH_COMPLEXITY=$(pmccabe "$FILE_PATH" 2>/dev/null | awk '$1 > 10' || true)
  if [ -n "$HIGH_COMPLEXITY" ]; then
    echo "❌ HOOK VIOLATION: Complexity >10 in $FILE_PATH" >&2
    echo "$HIGH_COMPLEXITY" >&2
    VIOLATIONS=$((VIOLATIONS + 1))
  fi

  # ═══════════════════════════════════════════════════════════
  # CHECK 4: Compilation test
  # ═══════════════════════════════════════════════════════════
  COMPILE_ERRORS=$(gcc -std=c11 -Wall -Wextra -Werror -fsyntax-only \
    -I/home/logmios/workspace/DISTRIBUTED_SYSTEMS/include \
    -D_POSIX_C_SOURCE=200809L \
    "$FILE_PATH" 2>&1 || true)

  if [ -n "$COMPILE_ERRORS" ]; then
    echo "❌ HOOK VIOLATION: Compilation errors in $FILE_PATH" >&2
    echo "$COMPILE_ERRORS" >&2
    VIOLATIONS=$((VIOLATIONS + 1))
  fi

  # ═══════════════════════════════════════════════════════════
  # RESULT
  # ═══════════════════════════════════════════════════════════
  if [ $VIOLATIONS -eq 0 ]; then
    echo "✅ NASA compliance verified: $FILE_PATH" >&2
    exit 0
  else
    echo "" >&2
    echo "═══════════════════════════════════════════════════════" >&2
    echo "⛔ BLOCKED: Fix $VIOLATIONS violation(s) before proceeding" >&2
    echo "═══════════════════════════════════════════════════════" >&2
    exit 1  # Non-zero exit BLOCKS Claude from continuing
  fi

  How This Works in Practice:

  1. Agent (subagent) writes code using Edit tool
     → Edit src/core/allocator.c

  2. PostToolUse hook triggers AUTOMATICALLY
     → scripts/check_nasa_compliance.sh src/core/allocator.c

  3a. If validation PASSES (exit 0):
      → Claude sees: "✅ NASA compliance verified"
      → Agent continues normally

  3b. If validation FAILS (exit 1):
      → Claude sees: "❌ BLOCKED: malloc detected"
      → Claude CANNOT proceed until fixed
      → Agent must Edit file again (loop until valid)

  NO MANUAL INTERVENTION NEEDED!

  ---
  🎯 Advanced: Pre-Tool Validation (Block Bad Code BEFORE Writing)

  Script 2: scripts/validate_code_before_write.py

  This analyzes Claude's INTENDED changes BEFORE they touch disk:

  #!/usr/bin/env python3
  # Validates code BEFORE Claude writes it (PreToolUse hook)

  import sys
  import json
  import re

  # Claude passes tool input via stdin
  tool_input = json.loads(sys.stdin.read())

  # For Edit/Write tools, check the new_string/content
  if 'new_string' in tool_input:
      code = tool_input['new_string']
  elif 'content' in tool_input:
      code = tool_input['content']
  else:
      sys.exit(0)  # Not a code write, skip

  violations = []

  # Check 1: No malloc
  if re.search(r'\bmalloc\b|\bfree\b|\brealloc\b', code):
      violations.append("❌ Code contains malloc/free (not allowed)")

  # Check 2: No unbounded loops
  if re.search(r'while\s*\(\s*(true|1)\s*\)', code):
      violations.append("❌ Code contains unbounded loop")

  # Check 3: No recursion (simplified check)
  # Extract function name being defined
  func_match = re.search(r'Result (\w+)\([^)]*\)', code)
  if func_match:
      func_name = func_match.group(1)
      # Check if function calls itself
      if re.search(rf'\b{func_name}\s*\(', code.split('{', 1)[1] if '{' in code else ''):
          violations.append(f"❌ Function {func_name} appears recursive")

  if violations:
      print("\n".join(violations), file=sys.stderr)
      print("\n⛔ BLOCKED: Fix violations before writing", file=sys.stderr)
      sys.exit(1)  # Block Claude
  else:
      print("✅ Pre-flight validation passed", file=sys.stderr)
      sys.exit(0)

  Usage in hooks:
  "PreToolUse": {
    "match": { "tool": ["Edit", "Write"] },
    "command": "python3 scripts/validate_code_before_write.py",
    "blocking": true
  }

  Effect: Claude literally CANNOT write code with violations!

  ---
  🔄 Complete Automatic Feedback Loop

  The Flow (Zero Manual Intervention):

  1. You spawn agent:
     Task("Implement allocator", "...", "general-purpose")

  2. Agent attempts to write code:
     Edit src/core/allocator.c (with malloc)

  3. PreToolUse hook fires:
     → validate_code_before_write.py
     → Detects malloc
     → Returns exit 1 (BLOCK)

  4. Claude receives hook result:
     "❌ BLOCKED: Code contains malloc"

  5. Agent sees error, rewrites code:
     Edit src/core/allocator.c (using pool allocator)

  6. PreToolUse hook fires again:
     → validate_code_before_write.py
     → No violations
     → Returns exit 0 (ALLOW)

  7. Edit succeeds, file written to disk

  8. PostToolUse hook fires:
     → check_nasa_compliance.sh
     → Validates file on disk
     → Returns exit 0 (OK)

  9. Agent continues to next task

  Entire loop happens automatically during agent execution!

  ---
  📊 Hook Types Comparison

  | Hook             | When              | Can Block? | Use Case                         |
  |------------------|-------------------|------------|----------------------------------|
  | PreToolUse       | Before tool runs  | ✅ Yes      | Block bad code BEFORE writing    |
  | PostToolUse      | After tool runs   | ✅ Yes      | Validate result, rollback if bad |
  | SubagentStop     | When agent done   | ✅ Yes      | Validate deliverables complete   |
  | UserPromptSubmit | Before processing | ❌ No       | Inject context (read-only)       |

  ---
  🎯 Recommended Setup for DISTRIBUTED_SYSTEMS

  Three-Layer Automatic Validation:

  // ~/.claude/settings.json
  {
    "hooks": {
      // Layer 1: Block bad code before writing (fastest feedback)
      "PreToolUse": {
        "match": { "tool": ["Edit", "Write"], "path": "**/*.c" },
        "command": "python3
  /home/logmios/workspace/DISTRIBUTED_SYSTEMS/scripts/validate_code_before_write.py",
        "blocking": true
      },

      // Layer 2: Validate file after writing (comprehensive check)
      "PostToolUse": {
        "match": { "tool": ["Edit", "Write"], "path": "**/*.c" },
        "command":
  "/home/logmios/workspace/DISTRIBUTED_SYSTEMS/scripts/check_nasa_compliance.sh
  \"$CLAUDE_TOOL_ARGS_file_path\"",
        "blocking": true
      },

      // Layer 3: Validate agent handoff when subagent completes
      "SubagentStop": {
        "match": { "type": "all" },
        "command":
  "/home/logmios/workspace/DISTRIBUTED_SYSTEMS/scripts/validate_agent_handoff.sh",
        "blocking": false  // Don't block, just warn
      }
    }
  }

  Benefits:

  1. ✅ Zero manual intervention during agent work
  2. ✅ Instant feedback (agent sees violations immediately)
  3. ✅ Self-correcting loop (agent retries until valid)
  4. ✅ Works with subagents (Task tool spawned agents)
  5. ✅ Prevents bad commits (only valid code reaches disk)

  ---
  🛠️ Quick Install

  Want me to:

  1. ✅ Create ~/.claude/settings.json with hooks config
  2. ✅ Create scripts/validate_code_before_write.py
  3. ✅ Create scripts/check_nasa_compliance.sh
  4. ✅ Create scripts/validate_agent_handoff.sh
  5. ✅ Test the hooks with a sample violation

  This gives you AUTOMATIC agent feedback loops with ZERO manual steps!

  Should I implement this now?
  ⎿  Stop hook succeeded:

> but if main agent instructing runs out of context, the hook loop ends, rigth?

● YES! Critical limitation identified.

  🚨 The Context Limit Problem

  What Happens When Main Agent Runs Out of Context:

  1. You spawn subagent:
     Task("Implement allocator", "...", "general-purpose")

  2. Subagent writes code (PostToolUse hook validates)
     → Violation detected
     → Hook returns error to subagent

  3. Subagent sees error, tries to fix
     → Edit file again
     → Hook validates again
     → Still has violation

  4. Loop continues: write → validate → error → fix → write...

  5. Subagent uses 50K tokens on validation loop

  6. Subagent hits 200K context limit
     → Subagent TERMINATES
     → Returns incomplete work to you (main coordinator)
     → Hook loop ENDS

  7. You see subagent failed, spawned new subagent
     → New subagent has fresh 200K context
     → Loop starts over...

  ---
  ⚠️ Scenarios Where Context Exhaustion Happens

  Scenario 1: Persistent Bug

  Agent writes code with malloc
    → Hook: "❌ malloc detected"
    → Agent: "Let me fix that" (uses 5K tokens)
    → Agent writes code with different malloc pattern
    → Hook: "❌ malloc detected"
    → Agent: "Let me try different approach" (5K tokens)
    → [Repeat 10 times]
    → 50K tokens wasted on same bug

  Scenario 2: Complex Refactoring

  Agent tries to reduce complexity from 28 → 10
    → Hook: "❌ Complexity still 22"
    → Agent refactors (10K tokens)
    → Hook: "❌ Complexity now 18"
    → Agent refactors more (10K tokens)
    → [Repeat 8 times]
    → 80K tokens on single function
    → Context exhausted before success

  Scenario 3: Compilation Errors

  Agent writes code
    → Hook: "❌ Missing header"
    → Agent adds header
    → Hook: "❌ Type mismatch"
    → Agent fixes type
    → Hook: "❌ Undefined reference"
    → [20 iterations]
    → Context exhausted

  ---
  🔄 Hook Behavior After Context Exhaustion

  What Actually Happens:

  When subagent hits 200K token limit:

  1. Claude Code SDK terminates subagent
  2. SubagentStop hook fires ONE LAST TIME
  3. Hook can validate final state, but CANNOT fix
  4. Main coordinator receives subagent report (may be incomplete)
  5. Hooks DO NOT carry over to next subagent (fresh start)

  Key Point: Hooks work per-session. New subagent = new hook lifecycle.

  ---
  ✅ Solutions to Context Exhaustion Problem

  Solution 1: Iteration Limits in Hooks

  Prevent infinite loops by counting validation attempts:

  #!/bin/bash
  # scripts/check_nasa_compliance.sh (with iteration limit)

  FILE_PATH=$1
  COUNTER_FILE="/tmp/claude_hook_counter_$$"

  # Increment counter
  if [ -f "$COUNTER_FILE" ]; then
    COUNT=$(cat "$COUNTER_FILE")
    COUNT=$((COUNT + 1))
  else
    COUNT=1
  fi
  echo $COUNT > "$COUNTER_FILE"

  echo "🔍 Validation attempt #$COUNT for $FILE_PATH" >&2

  # ═══════════════════════════════════════════════════════════
  # CIRCUIT BREAKER: Abort after 5 failed attempts
  # ═══════════════════════════════════════════════════════════
  if [ $COUNT -gt 5 ]; then
    echo "" >&2
    echo "⛔ CIRCUIT BREAKER TRIGGERED" >&2
    echo "   Validation failed 5+ times - stopping to prevent context exhaustion" >&2
    echo "   Agent should document blocker and return control to coordinator" >&2
    echo "" >&2

    # Clear counter
    rm -f "$COUNTER_FILE"

    # Return SUCCESS (allow agent to complete with partial work)
    # Agent will document the issue in handoff
    exit 0
  fi

  # Run normal validation...
  VIOLATIONS=0

  if grep -q "malloc" "$FILE_PATH"; then
    echo "❌ Attempt #$COUNT: malloc still detected" >&2
    VIOLATIONS=$((VIOLATIONS + 1))
  fi

  if [ $VIOLATIONS -eq 0 ]; then
    echo "✅ Validation passed on attempt #$COUNT" >&2
    rm -f "$COUNTER_FILE"  # Clear counter on success
    exit 0
  else
    exit 1  # Block and retry
  fi

  Effect: After 5 failed attempts, hook stops blocking and lets agent complete with
  documentation of blocker.

  ---
  Solution 2: Fast-Fail Validation (Prevent Wasted Context)

  Detect unwinnable situations early:

  #!/usr/bin/env python3
  # scripts/validate_code_before_write.py (with fast-fail)

  import sys, json, re

  tool_input = json.loads(sys.stdin.read())
  code = tool_input.get('new_string') or tool_input.get('content', '')

  violations = []

  # Check for violations
  if re.search(r'\bmalloc\b', code):
      violations.append("malloc detected")

  if re.search(r'while\s*\(\s*(true|1)\s*\)', code):
      violations.append("unbounded loop")

  # ═══════════════════════════════════════════════════════════
  # FAST-FAIL: Detect if agent is stuck in bad pattern
  # ═══════════════════════════════════════════════════════════
  HISTORY_FILE = "/tmp/claude_hook_history"

  # Load previous violations
  history = []
  if os.path.exists(HISTORY_FILE):
      with open(HISTORY_FILE, 'r') as f:
          history = [line.strip() for line in f.readlines()[-5:]]  # Last 5 attempts

  # Check if same violations repeating
  current_violations = "|".join(sorted(violations))
  history.append(current_violations)

  # Save history
  with open(HISTORY_FILE, 'w') as f:
      f.write("\n".join(history))

  # If last 3 attempts had SAME violations → agent is stuck
  if len(history) >= 3 and history[-1] == history[-2] == history[-3]:
      print("⚠️  WARNING: Same violations detected 3 times in a row", file=sys.stderr)
      print("   Agent may be stuck - recommend documenting blocker", file=sys.stderr)
      print("   Allowing this attempt to proceed...", file=sys.stderr)
      sys.exit(0)  # Allow (stop wasting context)

  if violations:
      print(f"❌ Violations: {', '.join(violations)}", file=sys.stderr)
      sys.exit(1)
  else:
      sys.exit(0)

  Effect: If agent tries same broken approach 3x, hook allows it through to prevent context
  waste.

  ---
  Solution 3: Context Budget Awareness

  Track token usage in hooks:

  #!/bin/bash
  # Estimate tokens used (rough heuristic)

  SESSION_START_FILE="/tmp/claude_session_start_$$"
  ESTIMATED_TOKENS_FILE="/tmp/claude_tokens_$$"

  # On first run, record start time
  if [ ! -f "$SESSION_START_FILE" ]; then
    date +%s > "$SESSION_START_FILE"
    echo "0" > "$ESTIMATED_TOKENS_FILE"
  fi

  # Estimate tokens used (very rough: 1K tokens per minute)
  START_TIME=$(cat "$SESSION_START_FILE")
  CURRENT_TIME=$(date +%s)
  ELAPSED_MINUTES=$(( (CURRENT_TIME - START_TIME) / 60 ))
  ESTIMATED_TOKENS=$((ELAPSED_MINUTES * 1000))

  echo "Estimated tokens used: ~${ESTIMATED_TOKENS}K" >&2

  # ═══════════════════════════════════════════════════════════
  # CONTEXT BUDGET: If >150K tokens used, stop blocking
  # ═══════════════════════════════════════════════════════════
  if [ $ESTIMATED_TOKENS -gt 150000 ]; then
    echo "⚠️  WARNING: Estimated token usage >150K" >&2
    echo "   Stopping validation to preserve context for completion" >&2
    exit 0  # Allow everything through
  fi

  # Normal validation if under budget...

  Effect: Hook becomes permissive when context is running low to ensure agent can complete.

  ---
  Solution 4: Tiered Validation (Progressive Relaxation)

  Relax validation criteria over time:

  # Attempt 1-2: STRICT (all checks)
  # Attempt 3-4: MEDIUM (skip complexity, allow warnings)
  # Attempt 5+: MINIMAL (only blocking errors)

  ATTEMPT=$(cat /tmp/hook_counter 2>/dev/null || echo 1)

  if [ $ATTEMPT -le 2 ]; then
    # STRICT: All checks
    run_all_nasa_checks
  elif [ $ATTEMPT -le 4 ]; then
    # MEDIUM: Essential checks only
    check_malloc
    check_unbounded_loops
    skip_complexity_check  # Documented technical debt
  else
    # MINIMAL: Compilation only
    gcc -fsyntax-only  # Just make sure it compiles
  fi

  Effect: Balances rigor with context efficiency.

  ---
  🎯 Recommended Hybrid Approach

  Smart Hooks with Circuit Breakers:

  // ~/.claude/settings.json
  {
    "hooks": {
      "PreToolUse": {
        "match": { "tool": ["Edit", "Write"], "path": "**/*.c" },
        "command": "scripts/smart_validate.sh pre \"$CLAUDE_TOOL_INPUT\"",
        "blocking": true
      },
      "PostToolUse": {
        "match": { "tool": ["Edit", "Write"], "path": "**/*.c" },
        "command": "scripts/smart_validate.sh post \"$CLAUDE_TOOL_ARGS_file_path\"",
        "blocking": true
      }
    }
  }

  scripts/smart_validate.sh:

  #!/bin/bash
  # Smart validation with circuit breakers

  MODE=$1  # "pre" or "post"
  FILE=$2

  COUNTER_FILE="/tmp/claude_validation_counter_$$"
  COUNT=$(cat "$COUNTER_FILE" 2>/dev/null || echo 0)
  COUNT=$((COUNT + 1))
  echo $COUNT > "$COUNTER_FILE"

  echo "🔍 Validation #$COUNT ($MODE mode)" >&2

  # ═══════════════════════════════════════════════════════════
  # CIRCUIT BREAKER 1: Max 5 attempts
  # ═══════════════════════════════════════════════════════════
  if [ $COUNT -gt 5 ]; then
    echo "⛔ Circuit breaker: 5 attempts reached - allowing to proceed" >&2
    rm -f "$COUNTER_FILE"
    exit 0
  fi

  # ═══════════════════════════════════════════════════════════
  # CIRCUIT BREAKER 2: Tiered validation
  # ═══════════════════════════════════════════════════════════
  if [ $COUNT -le 2 ]; then
    # STRICT: All NASA checks
    run_full_validation "$FILE"
  elif [ $COUNT -le 4 ]; then
    # MEDIUM: Core checks only
    check_malloc "$FILE"
    check_unbounded_loops "$FILE"
  else
    # MINIMAL: Compilation only
    gcc -fsyntax-only "$FILE"
  fi

  RESULT=$?

  if [ $RESULT -eq 0 ]; then
    echo "✅ Validation passed (attempt $COUNT)" >&2
    rm -f "$COUNTER_FILE"
  else
    echo "❌ Validation failed (attempt $COUNT)" >&2
  fi

  exit $RESULT

  ---
  📊 Expected Behavior With Smart Hooks

  Scenario: Agent Struggles with Complex Bug

  Attempt 1: Full validation → malloc detected → BLOCK
    Context used: 5K tokens

  Attempt 2: Full validation → still malloc → BLOCK
    Context used: 10K tokens

  Attempt 3: Core checks only → malloc → BLOCK (complexity ignored)
    Context used: 15K tokens

  Attempt 4: Core checks → malloc → BLOCK
    Context used: 20K tokens

  Attempt 5: Compilation only → compiles OK → ALLOW
    Context used: 25K tokens

  Agent completes with note: "Technical debt: malloc used (pool allocator refactor needed)"

  Total context waste: 25K tokens (vs 100K+ without circuit breakers)

  ---
  ✅ Final Recommendation

  Use Hooks with These Safety Features:

  1. ✅ Iteration limits (5 max attempts)
  2. ✅ Tiered validation (strict → medium → minimal)
  3. ✅ Fast-fail detection (same error 3x → stop)
  4. ✅ Context budget awareness (>150K tokens → permissive)
  5. ✅ Agent handoff requirements (must document blockers)

  Agent Instructions Include:

  If validation fails 3+ times:
  □ Document blocker in handoff
  □ Note: "Technical debt - validation blocked after N attempts"
  □ Suggest fix for next agent/session
  □ Complete with partial work (better than context exhaustion)

  This balances:
  - ✅ Validation rigor (most code gets validated)
  - ✅ Context efficiency (no infinite loops)
  - ✅ Agent completion (always produces deliverables)
  - ✅ Transparency (blockers documented)