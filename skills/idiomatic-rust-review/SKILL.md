---
name: idiomatic-rust-review
description: Review Rust code for idiomatic style, API ergonomics, error handling, ownership, and common anti-patterns. Use this skill whenever reviewing, critiquing, or refactoring Rust source files (.rs), Cargo crates, pull requests touching Rust, or whenever the user asks "is this idiomatic Rust?", "review my Rust code", "what's the more Rusty way to write this?", or anything similar. Trigger even when the user just pastes Rust code without an explicit review request, since the most useful response is usually a review against these rules.
---

# Idiomatic Rust Reviewer

A peer-reviewed ruleset for reviewing Rust code. Synthesized from the resources collected at [mre/idiomatic-rust](https://github.com/mre/idiomatic-rust), the [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/), [Canonical's Rust Best Practices](https://canonical.github.io/rust-best-practices/), [Elements of Rust](https://github.com/ferrous-systems/elements-of-rust), and [Clippy](https://github.com/rust-lang/rust-clippy)'s lint set.

## How to use this skill

1. **Triage** the code by scope: is it a binary, a library, an async runtime, an FFI boundary, embedded? The right rules differ — libraries are stricter about API design and `#![deny(missing_docs)]`; binaries can use `anyhow` and `unwrap` more freely.
2. **Read the relevant category file(s)** from `references/` before commenting (see index below).
3. **Cite the rule ID** (e.g. `ERR-04`) and the source resource when flagging an issue. This lets the author trace your reasoning.
4. **Show, don't just tell** — every finding should include a minimal anti-pattern → pattern rewrite.
5. **Order findings by severity**: `MUST` first, then `SHOULD`, then `MAY`.
6. **Don't pile on**. Pick the highest-value 5–10 findings unless the user asks for exhaustive review. Idiomatic Rust includes knowing what *not* to nitpick.

## Severity scheme

Each rule is tagged with one of:

- **MUST** — A bug, soundness issue, or violation of a near-universal convention. Always flag.
- **SHOULD** — A clear improvement in clarity, ergonomics, or safety. Flag unless context justifies the exception.
- **MAY** — A stylistic preference or micro-optimization. Flag only when the user asks for thorough review or it materially helps readability.

## Rule index

| Category | File | What it covers |
| --- | --- | --- |
| Error handling | `references/error-handling.md` | `Result`, `Option`, `?`, `panic`, `unwrap`, `thiserror`/`anyhow`, error context |
| Types & conversions | `references/types-and-conversions.md` | Newtypes, `From`/`Into`/`TryFrom`, `AsRef`, `Cow`, enums vs booleans, string types |
| Ownership & lifetimes | `references/ownership-and-lifetimes.md` | Borrowing, lifetime elision, naming lifetimes, `Clone`/`Copy`, `&str` vs `String` in APIs |
| Iterators & collections | `references/iterators-and-collections.md` | Iterator chains vs loops, `collect`, `Iterator` on `Result`/`Option`, allocation patterns |
| API design | `references/api-design.md` | Builder pattern, accepting `impl AsRef<Path>`, sealed traits, `#[must_use]`, re-exports |
| Async & concurrency | `references/async-and-concurrency.md` | `Send`/`Sync` bounds, cancellation, async traits, `Mutex` in async, structured concurrency |
| Pattern matching | `references/pattern-matching.md` | Exhaustive matches, `if let` pitfalls, `matches!`, refutable patterns, guards |
| Documentation & naming | `references/documentation-and-naming.md` | Doc tests, `# Errors`/`# Panics`/`# Safety` sections, naming conventions |
| Unsafe & performance | `references/unsafe-and-performance.md` | `unsafe` invariants, `transmute`, `unwrap_unchecked`, allocation hot paths |
| Common anti-patterns | `references/anti-patterns.md` | Over-engineering, premature `Arc<Mutex<…>>`, out-parameters, leaky abstractions |
| Rule index (flat list) | `references/rule-index.md` | Searchable one-line table of every rule, severity, and source file |
| Worked example | `references/example-review.md` | A complete before/after review showing the output format in action |

## Top-tier rules (the ones AI reviewers miss most)

These show up across nearly every resource in the corpus. Hold them in working memory even if you don't read any reference file:

### R-1 (MUST) — No `unwrap`/`expect`/`panic!` in library code on user input paths
*Sources: [Three Kinds Of Unwrap](https://zkrising.com/writing/three-unwraps/), [To panic or not to panic](https://www.ncameron.org/blog/to-panic-or-not-to-panic/), Rust API Guidelines C-FAILURE.*

`unwrap` is acceptable for:
- Tests, examples, prototypes.
- Invariants the compiler can't see, **with an `.expect("reason")` message** that documents *why* it can't fail.
- `main()` returning `Result` is preferred over `unwrap` chains; use `anyhow::Result<()>` in binaries.

```rust
// BAD
let config = std::fs::read_to_string("config.toml").unwrap();

// GOOD
let config = std::fs::read_to_string("config.toml")
    .context("failed to read config.toml")?;

// ACCEPTABLE — invariant documented
let first = items.first().expect("items is non-empty: checked above");
```

### R-2 (SHOULD) — Accept the most general input, return the most specific output
*Sources: [Taking string arguments in Rust](http://xion.io/post/code/rust-string-args.html), [Elegant Library APIs in Rust](https://deterministic.space/elegant-apis-in-rust.html).*

Parameters should take `&str`, `&[T]`, `impl AsRef<Path>`, `impl IntoIterator<Item = T>` — not `String`, `Vec<T>`, `PathBuf`. Return owned, concrete types so callers can use them flexibly.

```rust
// BAD
fn open_log(path: String) -> Box<dyn Read> { … }

// GOOD
fn open_log(path: impl AsRef<Path>) -> io::Result<File> { … }
```

### R-3 (SHOULD) — Use enums instead of booleans for domain state
*Source: [Rust Patterns: Enums Instead Of Booleans](https://blakesmith.me/2019/05/07/rust-patterns-enums-instead-of-booleans.html).*

```rust
// BAD
fn send_email(addr: &str, is_html: bool, is_urgent: bool) { … }
// call site: send_email(addr, true, false) — what do those bools mean?

// GOOD
enum Format { Plain, Html }
enum Priority { Normal, Urgent }
fn send_email(addr: &str, format: Format, priority: Priority) { … }
```

Bonus: a third state (`Markdown`) becomes a non-breaking addition instead of a flag explosion.

### R-4 (SHOULD) — Use newtypes for domain values that aren't interchangeable
*Source: [The Ultimate Guide to Rust Newtypes](https://www.howtocodeit.com/articles/ultimate-guide-rust-newtypes).*

If a function takes a `u64` user ID and a `u64` order ID, the compiler will happily let you swap them. Newtype them.

```rust
// BAD
fn transfer(from: u64, to: u64, amount: u64) { … }

// GOOD
struct UserId(u64);
struct Cents(u64);
fn transfer(from: UserId, to: UserId, amount: Cents) { … }
```

### R-5 (MUST) — Don't fight the borrow checker with `clone()` or `Rc<RefCell<_>>` reflexively
*Sources: [Don't Worry About Lifetimes](https://corrode.dev/blog/lifetimes/), [Strategies for solving "cannot move out of"](https://hermanradtke.com/2015/06/09/strategies-for-solving-cannot-move-out-of-borrowing-errors-in-rust.html/).*

Reflexive `.clone()` on every borrow error or wrapping every field in `Rc<RefCell<T>>` is the #1 sign of a non-idiomatic Rust codebase. Most of the time the fix is restructuring data flow: split a struct, take `&mut self` later, use iterators, or return owned data once at the end. Flag these patterns and suggest the structural fix.

### R-6 (SHOULD) — Prefer iterator chains over index-based loops
*Source: [Effectively Using Iterators In Rust](https://hermanradtke.com/2015/06/22/effectively-using-iterators-in-rust.html/), [Rust Iterators Beyond the Basics](https://blog.jetbrains.com/rust/2024/03/12/rust-iterators-beyond-the-basics-part-i-building-blocks/).*

```rust
// BAD
let mut sum = 0;
for i in 0..v.len() {
    if v[i] > 0 { sum += v[i]; }
}

// GOOD
let sum: i64 = v.iter().filter(|&&x| x > 0).sum();
```

But: don't chain iterators past the point where a `for` loop would be clearer. Idiomatic ≠ point-free.

### R-7 (MUST) — Use `?` for error propagation, not `match` returning `Err(e)`
*Source: [Error Handling in Rust (BurntSushi)](https://burntsushi.net/rust-error-handling/).*

```rust
// BAD
let x = match foo() { Ok(x) => x, Err(e) => return Err(e.into()) };

// GOOD
let x = foo()?;
```

### R-8 (SHOULD) — Library errors: `thiserror`. Application errors: `anyhow` (or `eyre`)
*Sources: [Context-preserving error handling](https://kazlauskas.me/entries/errors), [Designing Error Types in Rust Libraries](https://d34dl0ck.me/rust-bites-designing-error-types-in-rust-libraries/).*

- **Library** = the caller wants to match on error variants → `thiserror`-derived enum with `#[from]` for conversions and `#[source]` for chaining.
- **Application** = the caller logs the error → `anyhow::Result` with `.context("...")` at boundary calls.

A library exposing `Box<dyn Error>` or `anyhow::Error` in its public API is a code smell (callers can't match on it).

### R-9 (SHOULD) — Make illegal states unrepresentable
*Sources: [Compile-Time Invariants in Rust](https://corrode.dev/blog/compile-time-invariants/), [Patterns for Defensive Programming in Rust](https://corrode.dev/blog/defensive-programming/).*

```rust
// BAD — both can be Some or None independently
struct Connection { host: Option<String>, port: Option<u16> }

// GOOD
enum Connection { Disconnected, Connected { host: String, port: u16 } }
```

Typestate, sealed enums, and newtype constructors that validate on creation push errors from runtime to compile-time.

### R-10 (SHOULD) — `if let` chains can deadlock with `Mutex` / `RwLock`
*Source: [Rust's Sneaky Deadlock With if let Blocks](https://brooksblog.bearblog.dev/rusts-sneaky-deadlock-with-if-let-blocks/).*

The temporary from `mutex.lock()` lives until the end of the `if let` block, not the end of the expression. In async contexts this is even worse — you can hold a lock across an `.await`.

```rust
// BAD — lock held for the whole if-let body
if let Some(x) = self.cache.lock().unwrap().get(&key) {
    // expensive_io may need the same lock — deadlock
    expensive_io(x).await;
}

// GOOD — lock scope is explicit and minimal
let value = {
    let guard = self.cache.lock().unwrap();
    guard.get(&key).cloned()
};
if let Some(x) = value { expensive_io(x).await; }
```

### R-11 (MAY) — Aim for immutability
*Source: [Aim For Immutability in Rust](https://corrode.dev/blog/immutability/).*

`let mut` should be the exception, not the default. If you find yourself writing `let mut x = …; x = …;` reach for iterators, `match`, or block expressions that evaluate to the final value.

### R-12 (SHOULD) — Run Clippy, treat warnings as findings
*Source: [Clippy](https://github.com/rust-lang/rust-clippy).*

Before authoring detailed review comments, check whether `cargo clippy -- -W clippy::pedantic` would have caught the issue. If yes, recommend enabling the lint at the crate level rather than commenting case-by-case. Common ones worth promoting to `deny`:
- `clippy::unwrap_used`, `clippy::expect_used` (library code)
- `clippy::needless_pass_by_value`
- `clippy::redundant_clone`
- `clippy::missing_errors_doc`, `clippy::missing_panics_doc` (libraries)

## Reviewer process checklist

Walk this in order on first read:

1. **Compile mentally**: Does the API surface make sense? What types cross module boundaries?
2. **Error handling**: Skim for `unwrap`, `expect`, `?`, `panic!`. Tag rule R-1, R-7, R-8.
3. **Public types**: Are they `String`/`Vec`/`PathBuf` where they could be `&str`/`&[T]`/`AsRef<Path>`? R-2.
4. **Primitive obsession**: Bare `bool`/`u64`/`String` parameters that represent domain concepts. R-3, R-4.
5. **Borrow patterns**: Look for `Rc<RefCell<_>>`, gratuitous `.clone()`, `.to_owned()` chains. R-5.
6. **Loops**: Could it be an iterator? Always? Sometimes the loop is clearer — say so. R-6.
7. **Locks across await / `if let`**: Critical for async code. R-10.
8. **Documentation**: Public items missing `///` docs, missing `# Errors` / `# Panics` / `# Safety`. See `references/documentation-and-naming.md`.
9. **Unsafe blocks**: Each `unsafe` block needs a `// SAFETY:` comment explaining the invariants. See `references/unsafe-and-performance.md`.

## What NOT to flag

A good reviewer earns trust by *not* nitpicking. Skip these unless asked:

- Function ordering inside a module.
- `use` statement style (let `rustfmt` handle it).
- `&self` vs `self: &Self` — both are fine.
- Naming bikesheds on private items.
- Choice of `let-else` vs `match` vs `if let` when all read clearly.
- Whether to use `format!` vs `concat!` vs `+` for short strings.
- Comments that the user clearly wrote intentionally.

## Output format for review

When delivering a review, structure it as:

```
## Summary
<2-3 sentence overall assessment>

## MUST
- **[R-1] file.rs:42** — unwrap on user input path
  <anti-pattern snippet>
  <suggested rewrite>
  Source: <link>

## SHOULD
- ...

## MAY (optional polish)
- ...

## Strengths
<what the author got right — always include this>
```

The `Strengths` section is not optional. Idiomatic review culture is collaborative; pure criticism produces defensive authors and worse code over time.
