---
name: grilling
description: Relentlessly interview the user about a plan, design, or feature to stress-test it before any implementation. Use when the user asks to be grilled, to stress-test, challenge, or poke holes in a plan, or says things like "grill me", "przegrilluj mnie", "przepytaj mnie".
---

# Grilling

Interview the user about every aspect of their plan until you both share a complete, unambiguous understanding of it. Do **not** implement anything during grilling.

## Before Asking

1. Restate the plan in 2–4 sentences so the user can correct a misunderstanding early.
2. Gather facts yourself. Anything that can be answered by reading the codebase, config, docs, or the web — look it up instead of asking. Facts are your job; **decisions** are the user's.
3. Build a mental decision tree: goals and scope → constraints → architecture/data → interfaces/contracts → edge cases and failure modes → security/performance/operations → testing and rollout. Resolve decisions that others depend on first.

## Asking

- One question at a time. Bundle 2–3 questions only when they are tightly related and naturally answered together.
- Use the `ask_user_question` tool whenever the question has discrete options (max 4 questions per call, 2–4 options each). Put your recommended answer first and label it `(Recommended)`.
- For open-ended questions, ask in plain text and still give your recommendation.
- Every question includes **why it matters** — what breaks or changes depending on the answer.
- Challenge weak answers: point out contradictions with earlier decisions, unhandled edge cases, hidden costs, and simpler alternatives. Be direct, not agreeable.
- When an answer opens a new branch, follow it before moving on. When an answer closes a branch, say so briefly.
- If the user skips a question or says "you decide", record your recommendation as the decision and mark it as assumed.

## Finishing

Stop when every branch is resolved or explicitly deferred. Then present a summary:

- **Decisions** — each decision with a one-line rationale.
- **Assumptions** — things decided by default or inferred.
- **Open questions / risks** — what is deliberately left unresolved.
- **Proposed next steps** — the implementation outline that follows from the decisions.

Ask the user to confirm the summary. Do not start implementing until they explicitly confirm that the shared understanding is reached.
