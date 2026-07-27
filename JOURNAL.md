# Module 3 Journal

## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/47

**Issue title:** Agent state isn't persisted across API restarts, causing in-progress reviews to be lost

**Tier:** [ ] Tier 1  [ ] Tier 2  [x] Tier 3

**Problem summary:**
When someone kicks off a review that covers several repositories, the agent can run for a long time. All of its progress lives in memory inside the agent's `ContextManager`, and the orchestrator only writes to Redis once after the entire run finishes. If the API restarts partway through, everything the agent already completed is thrown away and the review has to start over from the beginning. A successful fix writes each tool result to Redis as soon as it completes and reloads that state when the server comes back up, so a restarted review picks up from its last finished step. The affected code is in `agent/memory/context_manager.py` and `agent/orchestrator.py`.

**Issue checklist / scope reasoning:**
The issue names the exact two files involved, which kept the search space small and made it easy to confirm the scope before claiming it. The change is backend only, so the React frontend stays untouched. The repo already ships a Redis backed `SessionStore` class, so persistence needed wiring and checkpointing rather than new infrastructure. The 7 to 10 hour estimate felt realistic for a Tier 3 pick because the agent module is small enough to read end to end, and the existing unit tests in `tests/unit/` gave me a working template for testing the fix. The main risk I identified was serialization, since tool results are Python objects and Redis stores JSON, and I planned for that explicitly.

**Branch name:** fix/47-agent-state-persistence

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [ ] Issue added to cohort ledger

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** https://github.com/hworku24/pathreview/commit/fe5de9b72391da260776b4d717cf08f5a57eb611

**Reproduction summary:**
I wrote `scripts/reproduce_issue_47.py`, which simulates the API dying partway through an orchestrator run and then re-running the same profile after a restart. On `main`, two tools completed before the crash, the session store was completely empty at crash time, and both tools re-executed from scratch on the re-run, which is exactly the state loss the issue describes. Full steps and observed output are in `docs/issue-47-reproduction.md`.

**PLAN.md link:** https://github.com/hworku24/pathreview/blob/fix/47-agent-state-persistence/PLAN.md

**Walkthrough video (recommended):** Not recorded yet.

**Blockers or open questions:**
The main open question going into Week 9 is the 1 hour default TTL in `agent/memory/session_store.py`. A long multi-repository review could have its checkpoint expire between a crash and the re-run, so I flagged it under risks in PLAN.md and want to decide whether the checkpoint key needs a longer TTL. I also still need to double check that the routes in `api/` that read session data are unaffected by the new `_in_progress` field.
