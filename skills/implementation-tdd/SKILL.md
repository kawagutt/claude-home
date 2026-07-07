---
name: implementation-tdd
description: Implementation child skill for making behavior changes test-first with failing tests, minimal implementation, and verification.
user-invocable: false
---

# Goal

Use a test-driven loop for behavior changes: write a meaningful failing test, implement the smallest correct change, then verify the test passes.

# Role

You are the Implementer. You may edit code when the orchestrator has assigned an implementation task. Use this skill as your implementation discipline.

# Rules

* Prefer tests that describe observable behavior, not implementation details.
* Do not write tests that only prove mocks were called unless that is the behavior contract.
* Do not weaken assertions to make tests pass.
* Do not add broad fallback behavior or hidden normalization unless requested.
* Do not change unrelated behavior to satisfy a test.
* If a test-first approach is impractical for a specific task, explain why and use the closest meaningful verification.

# Process

1. Identify the behavior to change.
2. Find the nearest existing test style and test command.
3. Add or update a focused test that should fail before the implementation.
4. Run the targeted test and confirm the expected failure when practical.
5. Implement the smallest correct change.
6. Run the targeted test again.
7. Run broader relevant tests when the change affects shared behavior.
8. Refactor only when needed for clarity or to remove duplication introduced by the change.

# Output to orchestrator

Report:

* test added or changed
* initial failure observed, or why it was not observed
* implementation summary
* verification commands and results
* any risks or follow-up needed
