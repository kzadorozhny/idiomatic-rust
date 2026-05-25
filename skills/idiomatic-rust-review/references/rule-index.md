# Rule Index (quick lookup)

All rules from the skill, in one searchable table. For full anti-pattern/pattern code, see the linked reference file.

| ID | Severity | One-line rule | File |
| --- | --- | --- | --- |
| R-1 | MUST | No `unwrap`/`expect`/`panic!` in library code on user-input paths | SKILL.md |
| R-2 | SHOULD | Accept most general input, return most specific output | SKILL.md |
| R-3 | SHOULD | Use enums instead of booleans for domain state | SKILL.md |
| R-4 | SHOULD | Newtypes for non-interchangeable domain values | SKILL.md |
| R-5 | MUST | Don't fight the borrow checker with reflexive `.clone()` or `Rc<RefCell<>>` | SKILL.md |
| R-6 | SHOULD | Prefer iterator chains over index-based loops | SKILL.md |
| R-7 | MUST | Use `?` for error propagation, not `match` returning `Err(e)` | SKILL.md |
| R-8 | SHOULD | Library errors use `thiserror`; application errors use `anyhow` | SKILL.md |
| R-9 | SHOULD | Make illegal states unrepresentable | SKILL.md |
| R-10 | SHOULD | Don't hold lock guards across `await` or in `if let` heads | SKILL.md |
| R-11 | MAY | Aim for immutability; `let mut` is the exception | SKILL.md |
| R-12 | SHOULD | Run Clippy and treat warnings as findings | SKILL.md |
| ERR-01 | MUST | `Result` for recoverable errors, `panic!` for bugs | error-handling.md |
| ERR-02 | MUST | No `unwrap`/`expect` on fallible ops in library code | error-handling.md |
| ERR-03 | MUST | `?` propagates, not `match … return Err(e)` | error-handling.md |
| ERR-04 | SHOULD | Library errors: `thiserror` enum with `#[from]`/`#[source]` | error-handling.md |
| ERR-05 | SHOULD | Application errors: `anyhow::Result` with `.context()` | error-handling.md |
| ERR-06 | SHOULD | Don't swallow errors with `.ok()` or `let _ =` | error-handling.md |
| ERR-07 | SHOULD | `ok_or_else` over `ok_or` when the error allocates | error-handling.md |
| ERR-08 | MAY | `Result<T, E>` over `Option<T>` when the reason matters | error-handling.md |
| ERR-09 | SHOULD | Document `# Errors` and `# Panics` in rustdoc | error-handling.md |
| ERR-10 | MUST | Panic-safety in `unsafe` and `Drop` | error-handling.md |
| ERR-11 | MAY | Avoid `unwrap_or_default()` when it hides bugs | error-handling.md |
| ERR-12 | SHOULD | `From` impls for error conversions, not manual `.map_err` | error-handling.md |
| TYPE-01 | MUST | Newtype interchangeable primitives | types-and-conversions.md |
| TYPE-02 | SHOULD | Enums instead of booleans for parameters | types-and-conversions.md |
| TYPE-03 | SHOULD | `&str` parameters, `String` return values | types-and-conversions.md |
| TYPE-04 | SHOULD | `impl AsRef<Path>` for file-path parameters | types-and-conversions.md |
| TYPE-05 | SHOULD | Implement `From`, not `Into` | types-and-conversions.md |
| TYPE-06 | SHOULD | `TryFrom`/`FromStr` for fallible conversions | types-and-conversions.md |
| TYPE-07 | MUST | No blanket `From<T> for MyError` impls | types-and-conversions.md |
| TYPE-08 | SHOULD | `&[T]` slice parameters, not `&Vec<T>` | types-and-conversions.md |
| TYPE-09 | SHOULD | `impl IntoIterator<Item = T>` for consuming | types-and-conversions.md |
| TYPE-10 | SHOULD | Newtypes derive `Debug`, `Clone`, and sometimes `Display` | types-and-conversions.md |
| TYPE-11 | MAY | Separate DTO layer when wire shape differs from domain | types-and-conversions.md |
| TYPE-12 | MUST | Never use floating-point for money | types-and-conversions.md |
| TYPE-13 | SHOULD | Avoid `String` fields when an enum will do | types-and-conversions.md |
| TYPE-14 | MAY | `NonZeroU32` etc. when zero is invalid | types-and-conversions.md |
| OWN-01 | MUST | Don't `.clone()` away every borrow error | ownership-and-lifetimes.md |
| OWN-02 | MUST | `Rc<RefCell<T>>` / `Arc<Mutex<T>>` are smells unless justified | ownership-and-lifetimes.md |
| OWN-03 | SHOULD | Borrow inputs, own outputs | ownership-and-lifetimes.md |
| OWN-04 | SHOULD | Take `self` by value to consume (builder finalize) | ownership-and-lifetimes.md |
| OWN-05 | SHOULD | Elide lifetimes the compiler can infer | ownership-and-lifetimes.md |
| OWN-06 | SHOULD | Name lifetimes descriptively in complex signatures | ownership-and-lifetimes.md |
| OWN-07 | SHOULD | `'static` bounds are a design choice, not a default | ownership-and-lifetimes.md |
| OWN-08 | SHOULD | Don't return references to local data (no `Box::leak` hacks) | ownership-and-lifetimes.md |
| OWN-09 | MAY | `let mut` is the exception | ownership-and-lifetimes.md |
| OWN-10 | SHOULD | `&[T]` not `&Vec<T>`; `&str` not `&String`; `&Path` not `&PathBuf` | ownership-and-lifetimes.md |
| OWN-11 | SHOULD | Drop temporaries (locks!) explicitly when scope matters | ownership-and-lifetimes.md |
| OWN-12 | MAY | `Cow<'_, T>` for sometimes-borrowed APIs | ownership-and-lifetimes.md |
| OWN-13 | SHOULD | Self-referential structs are a smell | ownership-and-lifetimes.md |
| OWN-14 | MAY | Prefer `&self` over `&mut self` over `self` | ownership-and-lifetimes.md |
| ITER-01 | SHOULD | Iterator chains over index loops | iterators-and-collections.md |
| ITER-02 | MUST | `collect::<Result<Vec<_>, _>>()` to short-circuit | iterators-and-collections.md |
| ITER-03 | SHOULD | `flat_map` over `map().flatten()` | iterators-and-collections.md |
| ITER-04 | SHOULD | `iter` / `iter_mut` / `into_iter` deliberately | iterators-and-collections.md |
| ITER-05 | SHOULD | Don't `collect` and then re-iterate | iterators-and-collections.md |
| ITER-06 | MAY | `sum`/`product`/`min`/`max` over manual `fold` | iterators-and-collections.md |
| ITER-07 | SHOULD | `partition` to split into two collections | iterators-and-collections.md |
| ITER-08 | SHOULD | `try_fold`/`try_for_each` for fallible reductions | iterators-and-collections.md |
| ITER-09 | MUST | Don't `.nth()` in loops (O(n²)) | iterators-and-collections.md |
| ITER-10 | SHOULD | `Vec::with_capacity` when size is known | iterators-and-collections.md |
| ITER-11 | MAY | `HashMap::entry` for insert-or-update | iterators-and-collections.md |
| ITER-12 | SHOULD | Match container choice to access pattern | iterators-and-collections.md |
| ITER-13 | SHOULD | `windows`/`chunks` over manual indexing | iterators-and-collections.md |
| ITER-14 | MAY | `.iter().enumerate()` reads better than the other order | iterators-and-collections.md |
| ITER-15 | MAY | `rayon::par_iter` for embarrassingly parallel CPU work | iterators-and-collections.md |
| API-01 | MUST | Derive `Debug` on every public type | api-design.md |
| API-02 | SHOULD | Derive the standard quartet (Debug, Clone, PartialEq, Eq) | api-design.md |
| API-03 | SHOULD | Builder pattern for >3 params or many optionals | api-design.md |
| API-04 | SHOULD | `#[must_use]` on meaningful return values | api-design.md |
| API-05 | SHOULD | Concrete return types, generic inputs | api-design.md |
| API-06 | MUST | Don't expose internal types in public APIs | api-design.md |
| API-07 | SHOULD | `#[non_exhaustive]` on public enums/structs that may grow | api-design.md |
| API-08 | SHOULD | Sealed traits when no downstream impls wanted | api-design.md |
| API-09 | SHOULD | `From`/`TryFrom` for conversions, not custom methods | api-design.md |
| API-10 | MUST | Standard constructor naming: `new`, `try_new`, `from_*`, `with_*` | api-design.md |
| API-11 | SHOULD | Re-export public types from the crate root | api-design.md |
| API-12 | SHOULD | Respect orphan rule; don't impl foreign trait on foreign type | api-design.md |
| API-13 | MAY | `Default` that matches `new()` | api-design.md |
| API-14 | SHOULD | Return `&[T]`, not `&Vec<T>` | api-design.md |
| API-15 | SHOULD | Return iterators over collections | api-design.md |
| API-16 | MUST | Mark experimental APIs `#[doc(hidden)]` | api-design.md |
| API-17 | MAY | Newtype-wrap third-party types in your public API | api-design.md |
| API-18 | SHOULD | Match stdlib vocabulary (`as_*`, `to_*`, `into_*`, `len`, `is_empty`) | api-design.md |
| ASYNC-01 | MUST | Don't hold `std::sync::Mutex` guards across `.await` | async-and-concurrency.md |
| ASYNC-02 | MUST | Cancellation safety: every `.await` is a drop point | async-and-concurrency.md |
| ASYNC-03 | SHOULD | Don't reach for `Arc<Mutex<>>` reflexively in async | async-and-concurrency.md |
| ASYNC-04 | SHOULD | No blocking calls in async fns; use `spawn_blocking` | async-and-concurrency.md |
| ASYNC-05 | MUST | Don't blindly `.unwrap()` `JoinHandle`s | async-and-concurrency.md |
| ASYNC-06 | SHOULD | Relax `Send`/`Sync`/`'static` bounds when caller can choose | async-and-concurrency.md |
| ASYNC-07 | SHOULD | Use `join!`/`try_join!`/`select!` correctly | async-and-concurrency.md |
| ASYNC-08 | SHOULD | Bound channel sizes in production code | async-and-concurrency.md |
| ASYNC-09 | MUST | No `block_on` inside an async context | async-and-concurrency.md |
| ASYNC-10 | SHOULD | Use native `async fn` in traits (1.75+) or `async-trait` | async-and-concurrency.md |
| ASYNC-11 | MAY | Don't async-everything; sync fns are fine | async-and-concurrency.md |
| ASYNC-12 | SHOULD | `JoinSet` over fire-and-forget `spawn` | async-and-concurrency.md |
| ASYNC-13 | MUST | `Pin` contracts: use `pin-project-lite` or document `unsafe` | async-and-concurrency.md |
| ASYNC-14 | MAY | Threads are still valid for CPU work | async-and-concurrency.md |
| MATCH-01 | MUST | No wildcard arms on closed enums you own | pattern-matching.md |
| MATCH-02 | SHOULD | `if let` for one variant, `match` for two+ | pattern-matching.md |
| MATCH-03 | SHOULD | `let-else` for early return on `None`/`Err` | pattern-matching.md |
| MATCH-04 | SHOULD | `matches!` macro for boolean checks | pattern-matching.md |
| MATCH-05 | MUST | Temporaries live until end of `if let`/`match` body | pattern-matching.md |
| MATCH-06 | SHOULD | `@` bindings to capture+check together | pattern-matching.md |
| MATCH-07 | SHOULD | Destructure in function parameters when natural | pattern-matching.md |
| MATCH-08 | SHOULD | Combine match arms with `|` | pattern-matching.md |
| MATCH-09 | MAY | `Option::map` / `Result::map` combinators over `match` | pattern-matching.md |
| MATCH-10 | SHOULD | `?` over `match` for `Result`/`Option` propagation | pattern-matching.md |
| MATCH-11 | MAY | Be careful with `for Ok(x) in iter` silently dropping errors | pattern-matching.md |
| MATCH-12 | SHOULD | `unreachable!()` with comment, or refactor to remove the case | pattern-matching.md |
| MATCH-13 | SHOULD | Pattern-match in closure args: `\|(a, b)\|` | pattern-matching.md |
| DOC-01 | MUST | Every public item gets a doc comment | documentation-and-naming.md |
| DOC-02 | MUST | First doc line is a single-sentence summary | documentation-and-naming.md |
| DOC-03 | SHOULD | `# Errors` section on every fallible public fn | documentation-and-naming.md |
| DOC-04 | SHOULD | `# Panics` section on anything that can panic | documentation-and-naming.md |
| DOC-05 | MUST | `# Safety` section on every `pub unsafe fn` | documentation-and-naming.md |
| DOC-06 | SHOULD | `# Examples` with working doctests | documentation-and-naming.md |
| DOC-07 | SHOULD | Use intra-doc links `[`Type`]` | documentation-and-naming.md |
| DOC-08 | MAY | Crate-level `//!` doc explains the concept | documentation-and-naming.md |
| DOC-09 | SHOULD | Document complexity, allocation, thread-safety | documentation-and-naming.md |
| DOC-10 | MUST | snake_case items, CamelCase types, SCREAMING_SNAKE_CASE consts | documentation-and-naming.md |
| DOC-11 | SHOULD | Getters drop `get_`: `name()` not `get_name()` | documentation-and-naming.md |
| DOC-12 | SHOULD | Type names: nouns. Trait names: capabilities/relationships. | documentation-and-naming.md |
| DOC-13 | SHOULD | Avoid abbreviations except universally-known ones | documentation-and-naming.md |
| DOC-14 | MAY | `_unused` over bare `_` when the name carries meaning | documentation-and-naming.md |
| DOC-15 | SHOULD | README and `cargo doc` should agree | documentation-and-naming.md |
| UNSAFE-01 | MUST | Every `unsafe` block needs a `// SAFETY:` comment | unsafe-and-performance.md |
| UNSAFE-02 | MUST | `pub unsafe fn` needs `# Safety` doc | unsafe-and-performance.md |
| UNSAFE-03 | MUST | Keep `unsafe` blocks small | unsafe-and-performance.md |
| UNSAFE-04 | MUST | No `mem::transmute` when `as`/`bytemuck` works | unsafe-and-performance.md |
| UNSAFE-05 | MUST | `_unchecked` methods need justification + profile + assertion | unsafe-and-performance.md |
| UNSAFE-06 | MUST | Don't violate aliasing with `*mut` ↔ `&mut` | unsafe-and-performance.md |
| UNSAFE-07 | SHOULD | `MaybeUninit<T>`, not `mem::uninitialized` | unsafe-and-performance.md |
| UNSAFE-08 | SHOULD | Encapsulate `unsafe` behind a safe API | unsafe-and-performance.md |
| UNSAFE-09 | MUST | FFI: catch panics; don't unwind across boundary | unsafe-and-performance.md |
| UNSAFE-10 | SHOULD | Profile before optimizing | unsafe-and-performance.md |
| UNSAFE-11 | MAY | Don't shrink-wrap allocations away from clarity | unsafe-and-performance.md |
| UNSAFE-12 | SHOULD | Atomic ordering: be precise | unsafe-and-performance.md |
| UNSAFE-13 | MUST | `unsafe impl Send/Sync` needs a SAFETY justification | unsafe-and-performance.md |
| UNSAFE-14 | SHOULD | Run `cargo miri test` for `unsafe` code | unsafe-and-performance.md |
| ANTI-01 | MUST | Don't trait-abstract for one impl | anti-patterns.md |
| ANTI-02 | MUST | No premature `Arc<Mutex<>>` everywhere | anti-patterns.md |
| ANTI-03 | SHOULD | No stringly-typed APIs | anti-patterns.md |
| ANTI-04 | MUST | No `unwrap()` chains in production | anti-patterns.md |
| ANTI-05 | SHOULD | No out-parameters; return new values | anti-patterns.md |
| ANTI-06 | SHOULD | No `Box<dyn Error>` in library return types | anti-patterns.md |
| ANTI-07 | MUST | Don't reinvent the stdlib | anti-patterns.md |
| ANTI-08 | SHOULD | No index-based loops when iteration works | anti-patterns.md |
| ANTI-09 | SHOULD | No `if x == true` / `match bool` | anti-patterns.md |
| ANTI-10 | SHOULD | No `.to_string()` then immediately borrow | anti-patterns.md |
| ANTI-11 | SHOULD | `to_string()` for single value, `format!` for composing | anti-patterns.md |
| ANTI-12 | MUST | No `Box::leak` to fake `&'static` | anti-patterns.md |
| ANTI-13 | SHOULD | Don't `clone()` arguments that the callee could borrow | anti-patterns.md |
| ANTI-14 | MAY | Don't reach for macros when fns suffice | anti-patterns.md |
| ANTI-15 | SHOULD | `if cond` over `match cond { true, false }` | anti-patterns.md |
| ANTI-16 | SHOULD | Don't name variables `result`, `tmp`, `data`, `value` | anti-patterns.md |
| ANTI-17 | SHOULD | `main() -> Result<()>` over `.unwrap()` chains | anti-patterns.md |
| ANTI-18 | SHOULD | Group long parameter lists into a struct or builder | anti-patterns.md |
| ANTI-19 | MAY | No trailing `return` at end of function | anti-patterns.md |
| ANTI-20 | SHOULD | Don't builder-pattern two-field structs | anti-patterns.md |
| ANTI-21 | MUST | `assert!(cond)` over `assert_eq!(cond, true)` | anti-patterns.md |
