---
name: code-structure
description: Language- and framework-agnostic guidance on code organization — file and function size, module boundaries, separation of concerns, side effects, naming, refactoring scope, and testability. Use when creating, editing, reviewing, refactoring, or designing code, modules, components, or project structure.
---

# Code Structure

This skill is about **how code is organized**, independent of language and framework. For TypeScript language conventions (types, imports, async, error syntax, modern built-ins), also apply the `typescript-style` skill. Established repo conventions take precedence over both.

## Readability First

**Optimize for the next person reading the code, not for the fewest lines.** Among correct solutions, readability wins over brevity, deduplication, clever syntax, and keeping everything inline.

- Existing architecture, naming, and tooling are constraints; existing unreadable code is not a pattern to copy. Improve the code in scope without restyling unrelated files.
- Write one logical step at a time. Separate guards, input reading, validation, payload construction, and side effects with blank lines.
- Use explicit branches for workflows. Do not hide validation, alternative operations, or side effects inside ternaries, boolean expressions, arguments, or rendering markup.
- Name meaningful intermediate values by their role: `stockInputResult`, `dialogTitle`, `validationMessage`. Avoid vague names such as `data`, `parsed`, or `result` when several stages or values coexist.
- Keep rendering declarative: markup describes the view; named handlers describe interactions; feature-local functions own nontrivial parsing and payload construction. A reader should not have to execute a callback mentally to understand a component's layout.
- Extract a helper when its name summarizes a cohesive operation and hides details the caller does not need. Start locally; a helper does not automatically need a new file, hook, class, or generic abstraction.
- Do not turn every expression into a helper or variable. A direct value or trivial callback is fine; extracting a one-line wrapper that adds no meaning only makes the code harder to follow.
- A little straightforward duplication is better than a shared abstraction that obscures different workflows. Do not add flags, generic parsers, or callback pipelines just to combine two small branches.

## Size Signals

These are warning lights, not hard limits. When one triggers, look for a better split — do not split mechanically.

| Unit                                            | Signal                                                                             |
| ----------------------------------------------- | ---------------------------------------------------------------------------------- |
| Any file                                        | > 300 lines                                                                        |
| UI component, route/controller, service, worker | > 200 lines — extract before adding more behaviour                                 |
| UI component specifically                       | > 150 lines — look for child components, extracted state/logic, formatting helpers |
| Function                                        | > 50 lines, or nesting deeper than 3 levels                                        |
| Test file                                       | > 400 lines — extract fixtures, builders, or split by behaviour                    |

A larger file with one clear responsibility beats several pass-through files. Split by **reason to change**, not by line count.

Never meet a size signal by removing whitespace, packing statements onto one line, or moving workflow logic into inline expressions. Readability is the goal; a shorter file is not evidence of better structure.

## Boundaries

Keep modules deep: a small public API hiding real complexity.

- Keep domain rules, parsing, formatting, and data normalization out of entry points — UI components, HTTP routes, CLI commands, event/message handlers, scheduled jobs. Entry points translate input, call a feature module, and translate output.
- Push side effects (I/O, time, randomness, global state) to the edges. Keep core logic pure or nearly pure, so it can be tested without mocks.
- Wrap external systems (network, filesystem, database, queues, browser APIs, third-party SDKs) behind small adapters owned by your code. The rest of the code depends on the adapter, not on the SDK.
- Define shared contracts (schemas, message/event types, DTOs) in one place when data crosses feature boundaries.
- Organize by feature/domain first, by technical layer second, unless the repo already does it differently.
- Avoid shallow modules: a helper that only forwards to another function adds indirection without hiding anything.
- Avoid circular dependencies between modules. If two modules need each other, extract what they share.
- Colocate tests with the module they cover (or follow the repo's test layout).

## Before Editing, Ask

- What is this file/module responsible for? Can I say it in one sentence?
- Which concerns are mixed here (I/O, business rules, presentation, mapping)?
- Is this duplication real knowledge duplication, or two similar-looking cases that will change for different reasons? Only the first should be merged.
- Would an extraction make the _next_ change or the test easier? If not, leave it.
- Could a better name remove the need for a comment?

## Implementation Defaults

- Narrow scopes, immutable data flow, no hidden global or shared mutable state.
- Guard clauses and early returns instead of deep nesting.
- Named option objects instead of long positional parameter lists (roughly > 3 parameters, or several of the same type).
- Model variants and states explicitly (tagged/discriminated unions, sum types) instead of combinations of booleans and optional fields.
- Use the standard library before writing utilities; use existing project utilities before adding new ones; add a dependency only when it clearly pays for itself.
- Handle errors explicitly at the right boundary; never swallow them.
- Names reveal intent and domain meaning: `calculateInvoiceTotal`, not `processData`; `isExpired`, not `check`.
- Comments explain why — intent, trade-offs, constraints, external quirks. Delete narration and stale comments.
- Clear, boring code over clever, compact code.

## Forms And Other Entry Points

A form submission or event handler should read top to bottom as a short sequence:

1. Stop the default action and reject an action that cannot run yet.
2. Read input.
3. Call a feature-local parser or operation that builds and validates the input.
4. Handle failure with an explicit block and early return.
5. Apply the side effect and its success/error behaviour.

Keep alternative input shapes and schema selection inside that cohesive parser, using explicit branches. Do not embed two schema calls and two payload shapes in a ternary inside the event handler. Use existing contracts rather than inventing a generic validation framework.

Keep parsing independent of UI state setters and network mutations. Keep the handler responsible for displaying validation errors and triggering the operation. During a readability refactor, preserve validation order, defaults, pending guards, error handling, and success callbacks.

## Refactoring Discipline

- Keep edits scoped to the requested behaviour and its ownership boundary.
- Fix nearby structural problems only when they are obvious, small, and directly related to the change. Otherwise, mention them instead of doing them.
- Preserve names, file layout, and patterns that are part of the local architecture unless they cause the problem being solved.
- Do not mix large refactors with behaviour changes in the same step; refactor first (behaviour-preserving), then change behaviour.
- Add or update tests when behaviour, contracts, parsing, or cross-module boundaries change. Test behaviour through public APIs, not implementation details.
- Run the project's formatter, linter, type checker, and relevant tests, proportionate to the change.

## Final Pass

Before finishing, scan the changed files for:

- compressed control flow, multiple statements on one line, and branches hidden inside expressions,
- inline handlers that mix interaction rules with markup, and nontrivial display values computed inside markup,
- vague names and workflows without visible steps; read the main path top to bottom without expanding nested expressions,
- swollen functions or files that crossed a size signal,
- mixed responsibilities and logic leaking into entry points,
- dead code, unused exports, leftover debug output, stale comments,
- duplication that encodes the same knowledge,
- abstractions with a single trivial use.

Report any worthwhile larger cleanup you noticed but intentionally left out of scope.
