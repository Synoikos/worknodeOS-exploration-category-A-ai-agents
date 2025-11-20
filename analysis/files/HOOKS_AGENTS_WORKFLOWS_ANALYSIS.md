# HOOKS_AGENTS_WORKFLOWS.MD - Analysis

**File**: `source-docs/HOOKS_AGENTS_WORKFLOWS.MD`
**Category**: Agent Validation & Workflow Automation
**Priority**: P0 (v1.0 CRITICAL for quality assurance)

---

## 1. Executive Summary

This document explores **automated validation loops** for AI agents using git pre-commit hooks and Claude Code hooks to prevent code violations **before** they reach the repository. The core insight: **Agents need real-time feedback** to self-correct, and hooks provide this without manual intervention.

Three validation layers proposed:
1. **Git pre-commit hooks**: Prevent bad commits (NASA compliance, complexity, compilation)
2. **Claude Code hooks**: Block bad code before writing to disk (PreToolUse/PostToolUse)
3. **Circuit breakers**: Prevent context exhaustion from validation loops

Key mechanisms:
- **12 validation checks**: malloc detection, unbounded loops, recursion, complexity >10, compilation, etc.
- **PreToolUse hooks**: Validate code BEFORE agent writes it
- **PostToolUse hooks**: Validate code AFTER agent writes it
- **Iteration limits**: Stop after 5 failed attempts to preserve context

**Value Proposition**: **Zero-manual-intervention quality enforcement** - agents self-validate and self-correct automatically.

---

## 2. Architectural Alignment

**Fits Worknode Abstraction**: ✅ **YES** (Application-Layer Pattern)

- Hooks are **external to Worknode** - run in development environment
- Validation scripts call Worknode RPC for audit logging
- Event emissions for validation results

**Integration Points**:
```c
// Worknode records validation events
Event validation_event = {
    .type = EVENT_CODE_VALIDATION,
    .source_id = agent_id,
    .data = {
        .result = VALIDATION_FAILED,
        .violations = ["malloc detected", "complexity 12 > 10"]
    }
};
worknode_emit_event(&agent_node, validation_event);
```

**Impact on Capability Security**: **MODERATE** (Enhancement)
- Hooks enforce **code-level capabilities** (what code patterns allowed)
- Complements Worknode's **file-level capabilities** (what files can be modified)
- Example: Agent has capability to write `src/network/`, but hook blocks if code violates NASA rules

**Impact on Consistency Model**: **NONE** (Development-Time Only)
- Hooks run before code reaches Worknode
- Don't affect runtime consistency
- Validation is synchronous (blocking)

---

## 3. Criterion 1: NASA Compliance Status

**Assessment**: ✅ **SAFE** (Enforcement Mechanism)

Hooks **enforce** NASA Power of Ten compliance:

1. **No Dynamic Allocation** (Rule 2)
   ```bash
   grep -n "malloc\|free\|realloc" $FILE
   # Blocks commit if found
   ```

2. **No Unbounded Loops** (Rule 3)
   ```bash
   ast-grep --pattern 'while (true) { $$$ }' $FILE
   # Blocks commit if found
   ```

3. **No Recursion** (Rule 1)
   ```bash
   # Detect function calling itself
   grep -n "$func_name(" $FILE
   ```

4. **Complexity ≤10** (Rule 10)
   ```bash
   pmccabe $FILE | awk '$1 > 10'
   # Blocks commit if complexity >10
   ```

**Why This is Critical for v1.0**:
- Agents might generate non-compliant code
- Automated enforcement prevents human error
- **Guarantees**: No non-compliant code reaches repository

**Compatibility with Worknode**:
- Hooks validate **before** code enters Worknode
- Worknode assumes all code is compliant
- Clean separation of concerns

---

## 4. Criterion 2: v1.0 vs v2.0 Timing

**Assessment**: 🔴 **v1.0 CRITICAL**

**Why Absolutely Required for v1.0**:
1. **Wave 4 uses AI agents** to implement RPC layer
2. **Without hooks**: Agents could commit non-compliant code
3. **With hooks**: Agents self-validate and fix violations automatically

**Integration Timeline**:
- **Before Wave 4 starts**: Setup hooks (2-3 days)
- **During Wave 4**: Hooks prevent violations as agents work
- **After Wave 4**: Validation logs show agent behavior

**Implementation Effort**:

**Week 1**: Git Pre-Commit Hooks (3 days)
- Create `.git/hooks/pre-commit` script
- Integrate: ast-grep, pmccabe, cppcheck, gcc
- Test on existing codebase

**Week 1**: Claude Code Hooks (2 days)
- Configure `~/.claude/settings.json`
- Create `scripts/validate_code_before_write.py`
- Create `scripts/check_nasa_compliance.sh`
- Test with sample Claude Code session

**Week 2**: Circuit Breakers (2 days)
- Add iteration counters to hooks
- Add tiered validation (strict → medium → minimal)
- Add context budget awareness

**Total**: 7 days (1.5 weeks)

**Blocking Risk**:
- If skipped: Agents could commit non-NASA-compliant code
- Fix cost: Refactor after the fact (weeks of work)
- Prevention cost: 1.5 weeks upfront

**Verdict**: Must be complete **before** Wave 4 agent work begins.

---

## 5. Criterion 3: Integration Complexity

**Score**: **2/10** (LOW)

**Justification**:
- No Worknode core changes required
- Scripts run externally via git hooks
- Uses existing tools (ast-grep, pmccabe, gcc, cppcheck)

**What Needs to Be Done**:

1. **Git Hook Script** (1 day)
   ```bash
   # .git/hooks/pre-commit
   #!/bin/bash
   STAGED_C_FILES=$(git diff --cached --name-only | grep '\.c$')

   for file in $STAGED_C_FILES; do
       # Check 1: No malloc
       if grep -q "malloc" "$file"; then
           echo "❌ malloc detected in $file"
           exit 1
       fi

       # Check 2: Complexity
       HIGH_COMPLEXITY=$(pmccabe "$file" | awk '$1 > 10')
       if [ -n "$HIGH_COMPLEXITY" ]; then
           echo "❌ High complexity in $file"
           exit 1
       fi
   done
   ```

2. **Claude Code Hook Config** (1 day)
   ```json
   {
     "hooks": {
       "PostToolUse": {
         "match": { "tool": ["Edit", "Write"], "path": "**/*.c" },
         "command": "scripts/check_nasa_compliance.sh",
         "blocking": true
       }
     }
   }
   ```

3. **Validation Scripts** (1 day)
   - `scripts/validate_code_before_write.py`
   - `scripts/check_nasa_compliance.sh`
   - `scripts/validate_agent_handoff.sh`

**No Core System Changes**: All external tooling

**Multi-Phase Implementation**: No - single phase (pre-Wave-4)

---

## 6. Criterion 4: Mathematical/Theoretical Rigor

**Assessment**: ⚠️ **RIGOROUS** (with limitations)

**Strong Foundation**:
- Static analysis (AST-based) is **decidable** for bounded programs
- Cyclomatic complexity is **computable** (proven metric)
- Pattern matching (regex, ast-grep) is **formally defined**

**Limitations**:
- **Halting problem**: Can't prove termination in general case
- **Heuristics**: Some checks are pattern-based (not proof-based)
- **False negatives**: Might miss subtle violations

**Examples**:

**Decidable Checks** ✅:
- malloc/free detection (lexical pattern)
- Complexity calculation (graph traversal)
- Compilation success (compiler verdict)

**Heuristic Checks** ⚠️:
- Recursion detection (may miss indirect recursion via function pointers)
- NULL check validation (may have false positives/negatives)
- Unbounded loop detection (while(1) caught, but while(f()) might not be)

**Formal Verification Gap**:
- Hooks provide **necessary but not sufficient** guarantees
- For full correctness: Add SPIN/Frama-C formal verification (v2.0+)

**Rigor Level**: **RIGOROUS** for development-time validation, **not proof-level correctness**

---

## 7. Criterion 5: Security/Safety Implications

**Assessment**: ✅ **SAFETY-CRITICAL** (Positive)

**Safety Enhancements**:
1. **Prevent Agent Exploitation**
   - Agent can't inject malloc into codebase
   - Agent can't create unbounded loops
   - Agent can't violate complexity limits

2. **Audit Trail**
   - All validation attempts logged
   - Pattern: Agent tries violation → Hook blocks → Agent fixes
   - Creates evidence of agent behavior

3. **Defense in Depth**
   - Layer 1: Agent self-validation (in prompt)
   - Layer 2: PreToolUse hook (before write)
   - Layer 3: PostToolUse hook (after write)
   - Layer 4: Git pre-commit hook (before commit)

**Security Considerations**:
- **Hook Bypass**: Agent could use `git commit --no-verify`
  - Mitigation: Document this as forbidden
  - Alternative: Server-side hooks (GitHub Actions)

- **Script Tampering**: Agent could modify validation scripts
  - Mitigation: Capability security - agent can't write to `scripts/` directory
  - Verify script integrity with checksums

**Circuit Breaker Safety**:
- Prevents infinite validation loops
- Saves context (allows agent to complete with partial work)
- Documented technical debt rather than hidden failures

---

## 8. Criterion 6: Resource/Cost Impact

**Assessment**: **LOW-COST** (<1% overhead)

**Development-Time Costs**:
- Hook execution: ~0.5-2 seconds per file modified
- Negligible compared to agent thinking time (5-30 seconds)
- Human perspective: Instant feedback

**Runtime Costs**:
- **Zero** - hooks run during development only
- No production overhead

**Tooling Costs**:
- ast-grep: Free, open-source
- pmccabe: Free, open-source
- cppcheck: Free, open-source
- gcc: Free, open-source

**Implementation Costs**:
- 1.5 weeks developer time (one-time)
- Maintenance: ~1 hour/month (update rules as needed)

**ROI Calculation**:
- Cost: 1.5 weeks upfront
- Benefit: Prevents weeks of refactoring non-compliant code
- ROI: 10-20x

---

## 9. Criterion 7: Production Deployment Viability

**Assessment**: ✅ **PRODUCTION-READY** (Proven Pattern)

**Maturity**:
- Git hooks: Used by millions of projects
- Pre-commit framework: Industry standard
- Static analysis tools: Battle-tested

**Deployment Complexity**: **LOW**
```bash
# Install hooks
cp scripts/pre-commit .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit

# Configure Claude Code
cp scripts/claude_settings.json ~/.claude/settings.json

# Test
git add test_file.c
git commit -m "test"  # Hook runs automatically
```

**Team Adoption**:
- Works for human developers too (not just agents)
- Enforces consistency across team
- Can't forget to run checks (automatic)

**CI/CD Integration**:
```yaml
# GitHub Actions
name: Validate Code
on: [push, pull_request]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run validation
        run: ./scripts/check_nasa_compliance.sh
```

**Known Challenges**:
- Hook bypass with `--no-verify` (mitigation: server-side checks)
- Cross-platform compatibility (WSL2 vs native Linux)
- Tool installation (ast-grep, pmccabe) - document in README

---

## 10. Criterion 8: Esoteric Theory Integration

**Relates to Existing Theories**: ⚠️ **MINIMAL** (Pragmatic Engineering)

**Weak Theoretical Connection**:
- Static analysis rooted in abstract interpretation
- Complexity metrics from graph theory
- But hooks are **practical application**, not research

**Potential Synergies**:

1. **Operational Semantics** (COMP-1.11)
   - Hook execution as small-step evaluation
   - Configuration: `⟨Code, Hook, Context⟩ → ⟨Code', Hook', Context'⟩`
   - Could formalize hook correctness

2. **Category Theory** (COMP-1.9)
   - Hooks as endofunctors: `Hook: Code → Code`
   - Composition: `Hook₃ ∘ Hook₂ ∘ Hook₁`
   - Hook pipeline guarantees

3. **Topos Theory** (COMP-1.10)
   - Validation as local consistency check
   - File-level compliance → project-level compliance
   - Sheaf-like aggregation

**Novel Research Opportunities**:
1. **Formal Verification of Hooks**: Prove hooks guarantee NASA compliance
2. **Hook Composition Theory**: Which hook orderings preserve correctness?
3. **Agent-Hook Game Theory**: Adversarial agent vs defensive hooks

---

## 11. Key Decisions Required

**Decision A-HOOK-001**: When to enforce hooks?
- **Option 1**: Pre-commit only (allow bad code temporarily)
- **Option 2**: Pre-Tool + Post-Tool + Pre-commit (strict)
- **Option 3**: Tiered (strict in CI, permissive locally)
- **Recommendation**: Option 2 - prevents bad code from ever reaching disk

**Decision A-HOOK-002**: Circuit breaker thresholds?
- **Current Proposal**: 5 attempts max
- **Alternative**: 3 attempts (more aggressive) or 10 attempts (more patient)
- **Recommendation**: 5 attempts - balances quality and context preservation

**Decision A-HOOK-003**: Validation tool choice?
- **ast-grep**: Syntax-aware pattern matching (recommended)
- **Alternative**: clang-tidy, cppcheck-only, custom parser
- **Recommendation**: ast-grep + pmccabe + gcc (multi-tool redundancy)

**Decision A-HOOK-004**: Hook bypass policy?
- **Allow `--no-verify`**: Trusts developers
- **Block `--no-verify`**: Strict enforcement
- **Recommendation**: Allow but log bypass events (audit trail)

**Decision A-HOOK-005**: Context exhaustion handling?
- **Option 1**: Stop validation after N attempts (allow partial work)
- **Option 2**: Terminate agent (force human intervention)
- **Recommendation**: Option 1 - preserve work, document technical debt

---

## 12. Dependencies

**Depends On**:
- **None** (standalone validation infrastructure)

**Blocks**:
- **Wave 4 agent work** - must be in place before agents start coding
- **Agent handoff quality** - ensures handoffs meet standards

**Enables**:
- **Safe agent autonomy** - agents can self-correct without human supervision
- **NASA compliance guarantee** - no non-compliant code reaches repository
- **Audit trail** - complete record of agent validation loops

**Required Tools**:
```bash
# Install validation toolchain
apt-get install pmccabe cppcheck gcc
cargo install ast-grep
```

**Other Files Dependency**:
- AI_AGENTS_WORKNODE.MD (agent integration context)
- Claude_flow_mechanisms.md (hook patterns)

---

## 13. Priority Ranking

**Overall**: 🔴 **P0** (v1.0 BLOCKING)

**Rationale**:
1. **Wave 4 uses agents** - must validate agent output
2. **NASA compliance critical** - violations unacceptable
3. **Low implementation cost** - 1.5 weeks
4. **High ROI** - prevents weeks of refactoring

**Implementation Schedule**:
- **Before Wave 4 starts**: Complete (mandatory)
- **During Wave 4**: Monitor hook effectiveness
- **After Wave 4**: Refine thresholds based on data

**Risk if Skipped**:
- HIGH - Agents could commit non-compliant code
- Cleanup cost: 3-6 weeks of refactoring
- Compliance risk: Lose NASA A+ grade

**Breakdown**:
- **Git pre-commit hooks**: P0 (mandatory)
- **Claude Code hooks**: P0 (mandatory)
- **Circuit breakers**: P0 (prevents context exhaustion)
- **Advanced validation** (Frama-C, SPIN): P2 (v2.0+)

---

## Summary: Implementation Roadmap

**Week 1: Git Hooks** (3 days)
```bash
# Day 1: Core checks
- malloc/free detection
- Unbounded loop detection (ast-grep)
- Recursion detection

# Day 2: Advanced checks
- Complexity (pmccabe)
- Static analysis (cppcheck)
- Compilation test (gcc)

# Day 3: Documentation checks
- Agent handoff structure validation
- Time estimate detection (forbidden)
- File reference validation
```

**Week 1: Claude Code Hooks** (2 days)
```bash
# Day 4: PreToolUse hook
- Validate code before writing
- Pattern matching (malloc, loops, etc.)
- Fast-fail for repeated violations

# Day 5: PostToolUse hook
- Validate file after writing
- Comprehensive checks (all 12 validations)
- Circuit breaker implementation
```

**Week 2: Circuit Breakers** (2 days)
```bash
# Day 6: Iteration limits
- Counter tracking (/tmp/hook_counter)
- 5-attempt threshold
- Graceful degradation

# Day 7: Tiered validation
- Attempts 1-2: Full validation (strict)
- Attempts 3-4: Core checks (medium)
- Attempts 5+: Compilation only (minimal)
```

**Validation**:
```bash
# Test suite
./tests/test_hooks.sh

# Expected results:
- ✅ Blocks malloc/free
- ✅ Blocks unbounded loops
- ✅ Blocks complexity >10
- ✅ Allows compliant code
- ✅ Circuit breaker activates after 5 attempts
```

---

**VERDICT**: Absolutely critical P0 feature that must be complete **before Wave 4 begins**. Low implementation cost (1.5 weeks) with massive ROI (prevents weeks of refactoring). Provides automatic quality assurance for AI agent work with zero manual intervention.

**Estimated Effort**: 7 days (1.5 weeks)
**Risk Level**: Low (proven pattern, mature tooling)
**Value**: High (enables safe agent autonomy)
