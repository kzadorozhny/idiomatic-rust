# Ownership & Lifetimes

Sources: [Don't Worry About Lifetimes](https://corrode.dev/blog/lifetimes/), [Strategies for solving 'cannot move out of' borrowing errors in Rust](https://hermanradtke.com/2015/06/09/strategies-for-solving-cannot-move-out-of-borrowing-errors-in-rust.html/), [Naming Your Lifetimes](https://www.possiblerust.com/pattern/naming-your-lifetimes), [Aim For Immutability in Rust](https://corrode.dev/blog/immutability/), [When should I use String vs &str?](https://steveklabnik.com/writing/when-should-i-use-string-vs-str/), Rust API Guidelines.

---

## OWN-01 (MUST) — Don't `.clone()` away every borrow-checker error

Reflexive cloning is the #1 sign of a Rust-by-translation codebase. Most "cannot move out of" / "borrowed value does not live long enough" errors have a structural fix:

| Symptom | Often-better fix |
| --- | --- |
| Can't move out of `&self`. | Take `self` by value, or return references with the right lifetime, or split the struct. |
| Multiple mutable borrows of fields. | Use `split_at_mut`, destructure, or restructure the struct. |
| Closure outlives function. | Take `'static` data, use `Arc`, or change ownership direction. |
| Iterator yields `&T`, caller needs `T`. | Pass ownership of the source, or use `into_iter()`. |

```rust
// BAD — clones every element just to placate the borrow checker
let names: Vec<String> = users.iter().map(|u| u.name.clone()).collect();

// GOOD — if you'll consume `users` anyway, take ownership
let names: Vec<String> = users.into_iter().map(|u| u.name).collect();

// GOOD — if you need both, separate the read from the consume
let names: Vec<&str> = users.iter().map(|u| u.name.as_str()).collect();
```

`.clone()` is sometimes the right answer — for small `Copy`-able types it's free, and for cheaply-cloneable types like `Arc<T>` it's idiomatic. The rule is: each `.clone()` should be a *deliberate* choice, not a panic-button.

## OWN-02 (MUST) — `Rc<RefCell<T>>` / `Arc<Mutex<T>>` is a code smell unless justified

These types exist for shared mutable state. If the structure can be expressed with normal `&` / `&mut` borrows, do that instead. Common refactors:

- A struct that "needs" `RefCell` to satisfy a callback often just needs `&mut self` plumbed through.
- A graph "needs" `Rc<RefCell<Node>>` only if you accept the interior-mutability tax — arena allocation (`generational_arena`, `slotmap`, indices into `Vec`) is usually cleaner and faster.

Flag any `Rc<RefCell<…>>` in new code and ask: "what does this enable that ownership can't?"

## OWN-03 (SHOULD) — Borrow inputs, own outputs

Default function signature mental model:

```rust
fn process(input: &Input) -> Output { … }
```

Borrow what you read, return what the caller will keep. Reach for `&mut self` / `&mut T` only when you genuinely need to modify in place.

## OWN-04 (SHOULD) — Take `self` by value to consume

If a method's purpose is to finalize, build, or transform an object so the old form is meaningless, take `self` by value. This is the **builder finalize** pattern.

```rust
// BAD — caller could try to .build() twice and get inconsistent state
impl RequestBuilder {
    pub fn build(&self) -> Request { Request { … } }
}

// GOOD
impl RequestBuilder {
    pub fn build(self) -> Request { Request { … } }
}
```

## OWN-05 (SHOULD) — Elide lifetimes when the compiler can infer them

Rust's elision rules cover the vast majority of single-input/single-output cases. Don't write what the compiler will write for you.

```rust
// BAD — noise
fn first_word<'a>(s: &'a str) -> &'a str { … }

// GOOD — elided
fn first_word(s: &str) -> &str { … }
```

## OWN-06 (SHOULD) — Name lifetimes descriptively in complex signatures

When elision *can't* handle it, `'a` and `'b` are uninformative. Name the lifetime after what it refers to.

```rust
// BAD
fn merge<'a, 'b>(left: &'a Map, right: &'b Map) -> Iterator<…> { … }

// GOOD
fn merge<'left, 'right>(left: &'left Map, right: &'right Map) -> Iterator<…> { … }

// EVEN BETTER — does it really need two lifetimes?
fn merge<'a>(left: &'a Map, right: &'a Map) -> Iterator<…> { … }
```

## OWN-07 (SHOULD) — `'static` bounds are a design choice, not a default

A `'static` bound on a generic parameter forces the caller's value to live forever (or be owned). It's required for threads and many async runtimes. Outside those cases, prefer relaxed bounds.

```rust
// BAD — forces every caller to pass a fully-owned type
fn handle<T: 'static>(value: T) { … }

// GOOD — accept anything that lives long enough
fn handle<'a, T: 'a>(value: T) { … }
```

## OWN-08 (SHOULD) — Don't return references to local data

The compiler will catch this, but reviewers often see attempts that "almost work" with leaking or `Box::leak`. If you need to return owned data, just say so.

```rust
// BAD — Box::leak to satisfy a &'static signature is almost always wrong
fn name() -> &'static str { Box::leak(format!("{}", x).into_boxed_str()) }

// GOOD
fn name() -> String { format!("{}", x) }
```

## OWN-09 (MAY) — `let mut` should be the exception

After writing a function, scan for `let mut`. Each one is a small opportunity to rewrite as a `match`, iterator chain, or `if`-expression that produces the final value directly. The goal isn't purism — it's that single-assignment code is easier to refactor and reason about.

```rust
// OK
let mut total = 0;
for x in xs { if x > 0 { total += x; } }

// BETTER
let total: i64 = xs.iter().filter(|&&x| x > 0).sum();
```

## OWN-10 (SHOULD) — Use `&[T]` slices, not `&Vec<T>`

Taking `&Vec<T>` is strictly less flexible than `&[T]`. The compiler will warn (`clippy::ptr_arg`).

```rust
// BAD
fn first(v: &Vec<i32>) -> Option<&i32> { v.first() }

// GOOD
fn first(v: &[i32]) -> Option<&i32> { v.first() }
```

Same goes for `&String` → `&str` and `&PathBuf` → `&Path`.

## OWN-11 (SHOULD) — Drop temporaries explicitly when scope matters

`MutexGuard`s, `RwLockReadGuard`s, and similar RAII types release on drop. The drop point is the end of the enclosing block — not the end of the statement.

```rust
// BAD — guard held across the await; can deadlock or hurt throughput
let guard = mutex.lock().await;
let value = guard.compute();
some_async_work().await;
use_it(value);

// GOOD — release before await
let value = {
    let guard = mutex.lock().await;
    guard.compute()
};                // guard dropped here
some_async_work().await;
use_it(value);
```

## OWN-12 (MAY) — Use `Cow<'_, T>` when sometimes-borrowed, sometimes-owned

`Cow` (clone-on-write) is for APIs that return the input unchanged when possible and only allocate on modification. Don't over-use — it adds a layer to every access.

```rust
fn trim_prefix<'a>(s: &'a str, prefix: &str) -> Cow<'a, str> {
    s.strip_prefix(prefix).map(Cow::Borrowed).unwrap_or(Cow::Borrowed(s))
}
```

## OWN-13 (SHOULD) — Self-referential structs are a smell

If you find yourself wanting a struct that holds both a `String` and a `&str` pointing into it, you need either a different design (store indices instead of references) or a crate like `ouroboros` / `self_cell`. Plain Rust can't express this safely.

## OWN-14 (MAY) — Prefer `&self` over `&mut self` over `self` in trait methods

For ergonomics: a `&self` method can be called on `Arc<T>`, in a shared context, behind a mutex guard. `&mut self` and `self` lock in stricter usage. This is a design choice, not a hard rule — sometimes you genuinely need `&mut self`.
