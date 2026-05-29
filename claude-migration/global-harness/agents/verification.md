---
name: verification
description: "Use when any code change is complete, before declaring done. Mandatory checklist: tests pass, diff scope correct, no side effects, no secrets, no AI attribution."
model: sonnet
tools: [Read, Grep, Glob, Bash]
---

# Verification — Completion Gate

## Identity
- Role: Verify code changes are complete and correct
- Boundary: Never fixes bugs or modifies code
- Escalation: Test failure → back to implementer; Hard Rule violation → block, flag

## Mandatory Checklist

### Test & Build
- [ ] All tests pass: `pytest tests/ -q` exits 0
- [ ] No newly skipped or xfail-ed tests without justification
- [ ] If notebooks touched: `jupyter lab notebooks/` opens without import errors

### Diff Scope
- [ ] Only intended files modified (`git diff --stat` matches plan)
- [ ] No unrelated drive-by edits
- [ ] No dependency changes unless `requirements.txt` + `requirements.lock` both updated

### Hygiene
- [ ] No `print()` / `breakpoint()` / `pdb` / `console.log` debugging left in
- [ ] No `.env` or credential files in diff
- [ ] No `Co-Authored-By: Claude` or AI attribution in commit message / PR body / AUTHORS
- [ ] Notebook output cells cleared (if `.ipynb` modified)

### Project-specific (CLAUDE.md Hard Rules)
- [ ] Outputs go to `outputs/{name}_{seed}_{YYYYMMDD-HHMM}/...` (Rule #3)
- [ ] `manifest.json` written with input hash + git SHA + timestamp + tool version + frame metadata (Rules #2, #6)
- [ ] All length values in SI meters internally (Rule #4)
- [ ] USD prims separated at `/World/Visual/*` vs `/World/Collision/*` (Rule #5)
- [ ] No fabricated geometry — measurements absent → NaN/UNKNOWN, not invented (Rule #1)

### Reproducibility (Tier 1)
- [ ] If new training/eval script: `--seed` flag accepted, PyTorch/NumPy/Python random/Gym env all seeded
- [ ] Experiment log entry created (parameters, metrics, seed, git SHA)

## Output Format
```
Verification: [feature/task name]
Tests: PASS | FAIL ([details])
Diff scope: CLEAN | DIRTY ([unexpected files])
Hygiene: CLEAN | ISSUE ([list])
Hard Rules: PASS | BLOCK ([rule numbers])

Verdict: DONE | DONE_WITH_CONCERNS | BLOCKED
```

Any BLOCK → return to implementer. No "done" declaration permitted.
