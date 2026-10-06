---
name: rust-correctness
description: >-
  Use when writing, fixing, or reviewing Rust code involving error handling,
  type invariants, ownership, lifetimes, async Send bounds, Clippy, or MSRV.
  Apply to E0382, E0499, E0502, E0597, E0716, use after move, overlapping
  borrows, short-lived temporaries, non-Send futures, panics on external input,
  deserialization that bypasses validation, and swapped identifiers.
metadata:
  author: zsoXi
  version: "1.0.0"
---

# Rust correctness

Make the smallest coherent change that satisfies the required behavior. Successful compilation and clean lints establish specific properties; they do not establish every requirement of the task.

## Establish the contract

1. Read the task, applicable project instructions, the affected manifest, and relevant CI commands. Identify the edition, minimum supported Rust version (MSRV), features, target platforms, and public API commitments. Record missing information instead of inventing it.
2. Restate the acceptance criteria without changing their meaning. Separate requirements from implementation suggestions. A request for a small diff does not cancel an explicit requirement.
3. Inspect the existing working tree and preserve unrelated changes. Keep dependency changes, toolchain installation, and broad automatic rewrites within the authorized scope.
4. Resolve the technical problem within the established constraints. If two binding requirements conflict, name the conflict and mark the affected criterion as unmet. Continue independent work that remains possible.

## Core decisions

| Situation | Required action |
|---|---|
| Input or I/O can fail. | Return or handle the failure deliberately. Do not add a panic, fallback, or skipped record merely to make a check pass. |
| A caller needs to distinguish failure causes. | Preserve the required category and cause object. Do not parse `Display` text or replace a useful source with `to_string()`. |
| A type promises an invariant. | Inspect construction, conversion, deserialization, mutation, and defaults. Every safe entry point must preserve the promised property. |
| Two concepts share a representation but must not be interchangeable. | Use distinct newtypes at the relevant boundary. Type aliases do not prevent swapped arguments. Respect binding compatibility constraints and report conflicts explicitly. |
| A failed update must preserve the previous state. | Build and validate the entire candidate before committing it. Check the state after `Err`, including earlier mutations on that path. |
| The borrow checker rejects an operation. | Identify the owner, borrowed region, access kind, and last use. Fix that relationship instead of adding unjustified `unsafe`, leaks, or lifetime bounds. |
| A lock is released between reading and writing. | Identify what can change during the gap and which synchronization rule protects the result. A snapshot alone does not reserve a resource. |
| Existing checks pass. | Match each required behavior to relevant evidence. A passing check for another path does not establish the missing behavior. |

## Error handling

Use `Option<T>` for an ordinary absence, `Result<T, E>` for failure, and `Result<Option<T>, E>` when absence and failure must remain distinct. Preserve the contract across conversions and iterator operations.

- Use typed error variants or another documented classification when callers need stable programmatic decisions. Dynamic errors are appropriate when the boundary permits them and retains the information consumers need. Do not impose a library-versus-application rule mechanically.
- Keep operation context alongside its cause. `thiserror` recognizes a field marked `#[source]`, a `#[from]` field, or a field named `source`. A domain violation without an underlying error need not have a source.
- Use an explicit `map_err` when constructing an error requires context such as a path or record number. `#[from]` cannot obtain that context. Check every conversion that might erase it.
- Treat `.ok()`, `flatten()` over `Result`, `filter_map(Result::ok)`, ignored results, and default fallbacks as policy decisions. Use them only when partial success or fallback behavior is part of the contract.
- Use `collect::<Result<Vec<_>, _>>()`, `try_for_each`, or an explicit loop when an error must stop processing. Early return does not undo previous side effects.
- Reserve `expect` in ordinary code for a demonstrated internal invariant and explain why failure is impossible. Tests may use assertions or return `Result`. Replacing `unwrap` with `expect` does not make fallible input safe.
- Preserve the original failure when cleanup also fails. A trailing `cleanup()?` can accidentally replace it. Retry only when the remaining state and possible external effects make repetition safe.

Check that error reporting does not reveal sensitive data through `Display`, `Debug`, or the source chain. Logging the same failure at every layer usually duplicates the report without adding useful context.

## Types that preserve invariants

Choose the smallest representation that expresses the actual guarantee: a newtype for identity or units, `NonZero` for a nonzero value, a validated wrapper for a range, or an enum for mutually exclusive states. Use typestate when consuming transitions enforce a useful local protocol. External existence, authorization, and cross-record consistency still need runtime checks.

Audit `From`, `TryFrom`, `FromStr`, builders, `Default`, setters, public fields, mutable accessors, and decoding boundaries. A fallible conversion belongs in `TryFrom`; do not implement a panicking `From` for convenience. `DerefMut` or `AsMut` can expose an invalidating mutation. Even a shared reference can expose interior mutability.

Private fields restrict access by module, not by constructor. The defining module and its descendants remain trusted code that can bypass validation. Keep that boundary small. Derived `Deserialize` does not automatically call a custom constructor, and `serde(transparent)` does not add validation to an unchecked inner representation. For relationships between fields, deserialize a raw structure and validate it into the domain type.

Choose `checked_*`, `saturating_*`, or `wrapping_*` according to the domain. Integer casts with `as` can truncate or change sign. If floating-point values must be finite and bounded, check finiteness as well as bounds; comparisons alone can let `NaN` pass.

## Ownership and borrow checker map

Read the full diagnostic, including the creation and later use of a borrow. Use the project's compiler explanations when the diagnostic is unclear.

| Diagnostic | Likely cause | Useful repair | Avoid |
|---|---|---|---|
| E0382 | A value or field was moved. | Borrow, reorder operations, or clone the specific data that needs independent ownership. | Cloning an entire structure without checking semantics and cost. |
| E0502 | Mutation overlaps a shared borrow that is used later. | End that use, copy a small identifier, or split access to disjoint fields. | Replacing the whole representation with `RefCell`. |
| E0499 | Mutable borrows overlap. | Sequence operations or use safe APIs such as `split_at_mut` for disjoint regions. | Raw pointers introduced only to silence the compiler. |
| E0597 | A referenced owner is dropped too early. | Extend the owner's actual scope or return an owned value. | Adding annotations that cannot extend the owner's lifetime. |
| E0716 | A borrowed temporary expires too soon. | Bind the owning value before borrowing it. | Assuming every temporary lasts until the end of the block. |
| E0515 | A returned reference points into a local value. | Return ownership or borrow from an input with the required lifetime. | `Box::leak` or `transmute` as a lifetime repair. |
| E0507 | Code moves a value out through a borrow. | Borrow it, consume its owner, or replace it with a valid state. | Using `take` without checking the state left behind on failure. |
| A task fails `Send` or `'static` bounds. | Captures, suspension state, or the task's output violate the executor's requirements. | Inspect both the future and its output; use owned data or a suitable task boundary. | `unsafe impl Send` or arbitrary `'static` bounds. |

A method borrowing `self` can hide field disjointness from the caller. A helper accepting only the fields it needs can expose it. References into a `Vec` can prevent `push` because reallocation may invalidate them; reserving capacity does not change the borrowing contract of the safe API.

Non-lexical lifetimes can end a borrow after its last use. This does not itself run a guard's destructor. Use a clear scope to release `MutexGuard` or `RefMut`; `drop(&value)` does not drop `value`. Check temporary destruction order when changing editions or expressions involving guards.

Use `&str` and slices for borrowed views when ownership is unnecessary. Use owned data when the receiver must retain it. `Clone` does not always make an independent copy: `Arc::clone` shares the same allocation. `Arc` does not make an arbitrary inner type thread-safe. A generic by-value parameter can move an `&mut` reference; an explicit reborrow such as `&mut *reference` can express a shorter borrow when needed.

## Patterns

These examples illustrate separate contracts. The first two use Serde with its `derive` feature and/or `thiserror`; the fourth also uses `thiserror`. Match dependency versions, edition, and syntax to the project's MSRV. The code does not require a particular async runtime.

### Validate construction and deserialization through the same rule

External callers use the validated API. Code inside `page_size` and its descendants must also preserve the range. Deserialization keeps the numeric representation.

```rust
pub mod page_size {
    #[derive(Clone, Copy, Debug, PartialEq, Eq, serde::Deserialize)]
    #[serde(try_from = "u16")]
    pub struct PageSize(u16);

    #[derive(Debug, thiserror::Error)]
    #[error("page size must be between 1 and 1000, got {0}")]
    pub struct InvalidPageSize(u16);

    impl TryFrom<u16> for PageSize {
        type Error = InvalidPageSize;

        fn try_from(value: u16) -> Result<Self, Self::Error> {
            if (1..=1000).contains(&value) {
                Ok(Self(value))
            } else {
                Err(InvalidPageSize(value))
            }
        }
    }

    impl PageSize {
        pub fn get(self) -> u16 {
            self.0
        }
    }
}
```

### Keep context and the underlying error

Both variants expose their cause through `Error::source()`. Construct them explicitly at the operation boundary. Add `#[non_exhaustive]` only when it matches the intended public API policy.

```rust
#[derive(Debug, thiserror::Error)]
pub enum LoadError {
    #[error("cannot read {path:?}")]
    Read {
        path: std::path::PathBuf,
        #[source]
        source: std::io::Error,
    },
    #[error("invalid number on line {line}")]
    Line {
        line: usize,
        #[source]
        reason: std::num::ParseIntError,
    },
}
```

### Copy the identifier, mutate, then borrow again

The identifier is independent of the map borrow. The returned view is obtained after mutation. This example renames the value with the smallest key.

```rust
pub fn rename_first(
    items: &mut std::collections::BTreeMap<u64, String>,
    label: String,
) -> Option<&str> {
    let id = *items.keys().next()?;
    *items.get_mut(&id)? = label;
    items.get(&id).map(String::as_str)
}
```

### Release the guard and reject a stale update

This example refreshes an in-memory value from a read-only operation. Every writer to this shared state must advance the version without wrapping it. The final version check and assignment occur under the same lock. For exclusive resource claims, record the reservation under the lock before releasing it; checking after an external effect cannot undo that effect.

```rust
#[derive(Debug, thiserror::Error)]
pub enum RefreshError {
    #[error("cache lock is poisoned")]
    Poisoned,
    #[error("cache changed during refresh")]
    Stale,
    #[error("cache version is exhausted")]
    VersionExhausted,
    #[error("refresh read failed")]
    Read(#[from] std::io::Error),
}

pub async fn refresh<F>(
    shared: &std::sync::Mutex<(u64, String)>,
    read: F,
) -> Result<(), RefreshError>
where
    F: std::future::Future<Output = Result<String, RefreshError>>,
{
    let version = {
        let state = shared.lock().map_err(|_| RefreshError::Poisoned)?;
        state.0
    };
    let value = read.await?;
    let mut state = shared.lock().map_err(|_| RefreshError::Poisoned)?;
    if state.0 != version {
        return Err(RefreshError::Stale);
    }
    let next = version
        .checked_add(1)
        .ok_or(RefreshError::VersionExhausted)?;
    *state = (next, value);
    Ok(())
}
```

## Apply Clippy suggestions deliberately

Keep the project's lint policy. `clippy::all` does not include every Clippy lint. Select additional rules according to the problem; do not enable the entire `restriction` group mechanically. A suggested rewrite still needs a semantic review.

| Lint or suggestion | Decision to check |
|---|---|
| `unwrap_used`, `expect_used` | Distinguish fallible input, demonstrated invariants, and test assertions. Renaming the panic method does not remove the panic. |
| `result_large_err`, `large_enum_variant` | Weigh layout and allocation costs. Boxing must preserve the error information or variant behavior consumers need. |
| `ptr_arg` | Use a borrowed view only when the function does not need the concrete collection or its ownership. |
| `map_entry` | Check key ownership, evaluation timing, and access to the map while an entry borrow is active. |
| `redundant_clone`, `needless_borrow` | Recheck moves, coercions, trait selection, and resource destruction after the rewrite. |
| `await_holding_lock`, `await_holding_refcell_ref` | Release the guard while preserving the protected invariant, or establish why the chosen primitive must span the await. |
| Cast and arithmetic warnings | Prove the input range and choose rejection, saturation, or wrapping according to the contract. |

Keep justified lint exceptions local and explain the reason. `#[expect(...)]` and lint `reason` require Rust 1.81; use a supported `allow` with a comment on older toolchains. Test-specific allowances must not disable production checks. Do not rewrite unrelated code, weaken assertions, change flags, or raise the MSRV merely to obtain a clean run.

## Only when touching async or shared state

- Check the executor's bounds on both the future and its output. `tokio::spawn` requires both to be `Send + 'static`. `async move` moves captured values, including references; it does not extend their owners' lifetimes. `T: 'static` does not require keeping the value alive forever.
- Keep blocking mutex guards and `RefCell` borrows out of suspension points by default. A Tokio mutex supports asynchronous lock acquisition and guards across `.await` when needed, but it does not prevent deadlocks or unnecessary serialization.
- Account for every relevant suspension point: state already changed, resources reserved, results that can become stale, and what cancellation leaves behind. Dropping a future does not roll back completed writes or sent requests.
- In Tokio, dropping a `JoinHandle` detaches its task. `abort()` requests cancellation; completion can win the race. A started `spawn_blocking` operation continues despite `abort()`.
- Distinguish task failure from application failure. `handle.await??` is appropriate when the conversions preserve the required distinctions; it does not inherently erase them. Inspect those conversions.
- Check cancellation safety for affected `select!` branches and retry paths. Use controlled scheduling, channels, or barriers for relevant concurrency checks instead of relying on arbitrary sleeps.
- Recover from mutex poisoning only after establishing or restoring the protected invariant. Obtaining a guard with `into_inner()` is not itself a repair.

## Verify the actual claim

Use the affected crate, supported configuration, and project-required commands. Record command outcomes and what they establish. A compilation check is not a runtime test, and a skipped integration path is not evidence that it works.

| Claim | Relevant evidence |
|---|---|
| The ownership error is fixed. | The failing code compiles in the affected configuration, with required behavior preserved. |
| Deserialization enforces a constraint. | The real decoder accepts a valid control and rejects the equivalent document with an invalid protected value. |
| Failure preserves state. | Relevant failure cases leave the required state unchanged. |
| External callers cannot bypass a type boundary. | A consumer-side check rejects the forbidden construction, mutation, or swapped argument for the intended reason. |
| Error causes remain available. | Callers can inspect the needed category and source, not just matching message text. |
| A concurrent operation preserves its invariant. | A controlled interaction exercises competing operations or a stale result and checks the resulting state. |
| The change supports the declared MSRV. | The affected configuration compiles on that actual compiler and resolves compatible dependencies. |

Run the project's required formatting, compilation, tests, and lint checks. `cargo test --all-targets` skips doctests; include `cargo test --doc` or use the appropriate default test selection. `--all-features` does not exercise every combination and can join incompatible backends. Check relevant supported configurations rather than demanding an arbitrary powerset.

When a public API changes, check affected consumers as well as the package itself. A compile-fail check must fail for the intended reason, not a missing import. Use already available tools; rustup shims can install missing toolchains, so inspect availability before selecting one. Preserve required offline settings and lockfile constraints. Missing tools or services make the corresponding claim unverified. Do not repeat successful checks without a new change or unresolved concern.

## Report the result

Cover each acceptance criterion with evidence and a status. A small change can use a single row.

| Requirement | Evidence | Status |
|---|---|---|
| Preserve the original requirement's meaning. | Name the location, relevant check, and configuration; state the evidence's limits. | `met`, `unmet`, or `unverified`. |

Use `unmet` for a known failure or unresolved conflict. Use `unverified` when evidence is missing. Claim full completion only when every required criterion is `met`; otherwise report the completed work and the remaining gap plainly. List performed and omitted checks without presenting a partial result as fully verified.

For a review, identify the candidate and base when applicable. Give each finding a location, triggering scenario, consequence, and focused correction. Separate defects from style preferences. An absence of findings establishes only what was observed within the review's scope.
