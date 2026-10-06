# Rust correctness

**Practical Rust guidance for AI coding agents.**

A self-contained Agent Skill for writing, fixing, and reviewing Rust. It helps agents preserve error causes, enforce type invariants, resolve ownership problems, and apply Clippy suggestions without changing the intended behavior.

The guiding rule is simple: make the smallest coherent change that satisfies the contract, then verify the relevant behavior.

**[Read the skill →](SKILL.md)**

## What it covers

| Area | Focus |
| --- | --- |
| Error handling | Preserve typed failure categories, useful context, and the `Error::source()` chain. |
| Type invariants | Validate construction, deserialization, conversions, and mutation through consistent rules. |
| Ownership and borrowing | Diagnose moves, overlapping borrows, short-lived temporaries, and lifetime mismatches. |
| Clippy | Review the semantic impact of suggested changes, including ownership, evaluation order, allocation, and public API behavior. |
| Async and shared state | Control guard lifetimes, reject stale updates, and distinguish cancellation from completed side effects. |
| Verification | Connect acceptance criteria to concrete evidence while respecting MSRV and supported configurations. |

## Quick start

Clone the repository:

```sh
git clone https://github.com/zsoXi/rust-correctness.git
```

Place the `rust-correctness` directory in the skills location configured for your agent, keeping `SKILL.md` at the directory root. If your client provides a skill installer, use its repository installation flow instead.

Only `SKILL.md` is needed by the agent. The skill has no supporting reference files, scripts, or runtime installation requirements. Rust examples use project dependencies where indicated; they do not require installing a separate skill runtime.

Once the skill is available, ask your agent to use `rust-correctness` for a relevant task. Activation and installation details depend on the agent client.

## Example requests

> Use rust-correctness to fix this E0502 error while preserving the function's behavior and avoiding unnecessary clones.

> Use rust-correctness to review whether this newtype protects its invariant through Serde deserialization and every public mutation path.

> Use rust-correctness to preserve the underlying error causes while adding context to this file-loading operation.

> Use rust-correctness to review this async cache refresh for guards held across await, stale writes, and cancellation behavior.

## Inside the skill

The document combines a decision table, a borrow checker diagnostic map, focused Clippy guidance, and four concrete Rust examples:

1. A validated `PageSize` newtype that reuses its conversion rule during deserialization.
2. A contextual error enum that preserves both I/O and parsing causes.
3. A copy-then-mutate-then-borrow pattern for updating a map entry.
4. An async refresh that releases its guard before awaiting and rejects stale results before committing.

It also defines an evidence-based completion report using `met`, `unmet`, and `unverified`, so missing verification remains visible.

## Design principles

- Keep required behavior and compatibility constraints explicit.
- Explain the technical reason for a rule and the conditions that change a decision.
- Preserve meaningful failures instead of converting them into apparent success.
- Keep the instructions self-contained and independent of a particular agent vendor.
- Match verification to the actual claim, configuration, and failure path.

The skill provides engineering guidance. Project instructions, the actual codebase, compiler diagnostics, and task requirements determine how it should be applied.

## Contributing

Keep changes focused and explain the problem they address. For changes to a Rust example, include the relevant compiler or behavioral evidence in the pull request. Preserve the self-contained structure of `SKILL.md` and keep client-specific setup details outside its instructions.

## Author

Created and maintained by **[zsoXi](https://github.com/zsoXi)**.
