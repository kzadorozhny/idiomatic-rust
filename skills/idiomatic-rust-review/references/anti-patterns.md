# Common Anti-Patterns

Sources: [The Four Horsemen of Bad Rust Code](https://github.com/corrode/four-horsemen-talk), [Be Simple](https://corrode.dev/blog/simple/), [The Mediocre Programmer's Guide to Rust](https://www.hezmatt.org/~mpalmer/blog/2024/05/01/the-mediocre-programmers-guide-to-rust.html), [Pitfalls of Safe Rust](https://corrode.dev/blog/pitfalls-of-safe-rust/), [Rustic Bits](https://llogiq.github.io/2016/02/11/rustic.html), [Are out parameters idiomatic in Rust?](https://steveklabnik.com/writing/are-out-parameters-idiomatic-in-rust/).

---

## ANTI-01 (MUST) — Over-engineering: traits for one impl

If you write a trait and then write exactly one `impl` of it, you've added indirection for nothing. Wait until there's a second implementation before generalizing.

```rust
// BAD — abstraction for a future that may never come
trait UserRepository { fn get(&self, id: u64) -> Option<User>; }
struct PostgresUserRepository { … }
impl UserRepository for PostgresUserRepository { … }

// GOOD — start concrete; extract the trait when you actually need to swap
struct UserRepository { db: PgPool }
impl UserRepository { fn get(&self, id: u64) -> Option<User> { … } }
```

This is sometimes called "speculative generality." When the second implementation arrives, *then* extract the trait — the second use case will tell you what should and shouldn't be in it.

## ANTI-02 (MUST) — Premature `Arc<Mutex<T>>` everywhere

A sign of porting from a GC language. Shared mutable state is rarely the right answer in Rust; usually one of these is better:

- An owned value with `&mut` plumbed where needed.
- An actor task that owns the state, communicated to via a channel.
- An `Arc<T>` of immutable data + per-thread state.
- An `Arc<RwLock<T>>` when reads vastly outnumber writes.

Flag any struct with multiple `Arc<Mutex<...>>` fields and ask whether the design can be inverted.

## ANTI-03 (SHOULD) — Stringly-typed APIs

```rust
// BAD
fn run(action: &str, args: &[&str]) { … }
run("delete", &["force", "recursive"]);

// GOOD
enum Action { Create, Delete { force: bool, recursive: bool } }
fn run(action: Action) { … }
```

A function taking a `&str` and dispatching on its value is asking for typos and missing arms.

## ANTI-04 (MUST) — `unwrap()` chains in production code

```rust
// BAD
let user = db.query("SELECT * FROM users WHERE id = ?", &[id])
    .unwrap()
    .into_iter()
    .next()
    .unwrap()
    .into_user()
    .unwrap();
```

Each `unwrap` is a future support ticket. Replace with `?` propagation and a sensible error type.

## ANTI-05 (SHOULD) — Out-parameters (mutate-via-`&mut` instead of return)

In Rust, prefer returning new values to mutating arguments. Out-parameters are a C/C++ habit and don't survive contact with the borrow checker well.

```rust
// BAD
fn compute(input: &Input, output: &mut Output) { … }

// GOOD
fn compute(input: &Input) -> Output { … }
```

Exceptions:
- The output is a `Vec`/`String` and the caller wants to reuse the allocation (`vec.clear(); fill(&mut vec);`).
- The output is huge and you want to avoid a move (rare; `Vec` is just a pointer).

## ANTI-06 (SHOULD) — `Box<dyn Error>` in library return types

Opaque, can't be matched on, prevents structured error handling. See `ERR-04`. Acceptable in binaries and tests where the only consumer is a logger or `?`.

## ANTI-07 (MUST) — Reinventing the standard library

```rust
// BAD
fn is_empty<T>(v: &Vec<T>) -> bool { v.len() == 0 }

// JUST USE
v.is_empty()
```

Common reinventions to flag:
- Custom `Result` types when `Result<T, E>` works.
- Custom option-like enums when `Option<T>` works.
- Custom iterator structs when `impl Iterator` of standard combinators works.
- Hand-rolled `min`/`max`/`sort` instead of slice methods.

## ANTI-08 (SHOULD) — Indexing in loops when iteration would do

```rust
// BAD
for i in 0..v.len() { do_something(&v[i]); }

// GOOD
for x in &v { do_something(x); }

// GOOD (with index)
for (i, x) in v.iter().enumerate() { … }
```

Indexing has bounds-check cost and obscures intent.

## ANTI-09 (SHOULD) — `if x == true` / `if x == false` / `match bool`

```rust
// BAD
if is_ready == true { … }
if is_ready == false { … }
match is_ready { true => …, false => … }

// GOOD
if is_ready { … }
if !is_ready { … }
```

`clippy::bool_comparison`, `clippy::match_bool`.

## ANTI-10 (SHOULD) — `.to_string()` then immediately borrow

```rust
// BAD
fn greet(name: &str) -> String { format!("hi, {}", name) }
let s = "world".to_string();
greet(&s);

// GOOD
greet("world");
```

Each `.to_string()` is an allocation. Most idiomatic Rust code allocates only when storing or returning.

## ANTI-11 (SHOULD) — `format!("{}", x)` instead of `x.to_string()` (or vice versa, in the wrong situation)

- For a single value, `x.to_string()` is clearer and slightly faster.
- For composing multiple values, `format!("{a}-{b}")` is clearer than chaining `+` or `push_str`.

```rust
// REDUNDANT
let s = format!("{}", x);

// IDIOMATIC
let s = x.to_string();
```

## ANTI-12 (MUST) — Returning `&'static str` via `Box::leak` to satisfy a lifetime

```rust
// BAD — leaks memory every call
fn name(id: u32) -> &'static str {
    Box::leak(format!("user-{id}").into_boxed_str())
}

// GOOD — return owned String
fn name(id: u32) -> String { format!("user-{id}") }
```

If you genuinely need `'static`, use `Arc<str>` or a string interner (`string-cache`, `lasso`, `internment`).

## ANTI-13 (SHOULD) — `clone()` arguments to call a function

If a function takes ownership, ask whether it really needs to. Often `&T` or `&[T]` is enough.

```rust
// BAD (when send() doesn't actually need ownership)
send(message.clone());
println!("sent: {}", message);

// GOOD — change send to take &Message
send(&message);
println!("sent: {}", message);
```

## ANTI-14 (MAY) — Over-using macros for things functions can do

```rust
// BAD — macros are great, but not for everything
macro_rules! add { ($a:expr, $b:expr) => { $a + $b } }

// GOOD
fn add(a: i32, b: i32) -> i32 { a + b }
```

Reach for macros when you need variadic arguments, repetition, syntax extension, or compile-time code generation. Don't reach for them as "fast functions."

## ANTI-15 (SHOULD) — `match` on a single boolean condition

```rust
// BAD
match check() {
    true => proceed(),
    false => abort(),
}

// GOOD
if check() { proceed() } else { abort() }
```

## ANTI-16 (SHOULD) — Naming variables `result`, `tmp`, `data`, `value`

These names tell you nothing. Use the type or domain term.

```rust
// BAD
let result = parse_config(&s);
let tmp = result.users;
for value in tmp { process(value); }

// GOOD
let config = parse_config(&s);
for user in config.users { process(user); }
```

`result` is acceptable when the function literally returns a `Result` to be inspected immediately.

## ANTI-17 (SHOULD) — Returning `Result<(), Error>` from `main` instead of `unwrap`-chains

```rust
// BAD
fn main() {
    let cfg = load_config().unwrap();
    let srv = bind(&cfg).unwrap();
    srv.run().unwrap();
}

// GOOD
fn main() -> anyhow::Result<()> {
    let cfg = load_config()?;
    let srv = bind(&cfg)?;
    srv.run()?;
    Ok(())
}
```

`anyhow::Result` formats the chain on exit.

## ANTI-18 (SHOULD) — Long parameter lists without grouping

```rust
// BAD
fn connect(host: &str, port: u16, user: &str, pass: &str,
           tls: bool, timeout: Duration, retries: u8) { … }

// GOOD
struct ConnectionParams { host: String, port: u16, … }
fn connect(params: &ConnectionParams) { … }
// or a builder; see API-03
```

The threshold is roughly 4–5 parameters. Beyond that, group them.

## ANTI-19 (MAY) — Trailing `return` statements

```rust
// BAD
fn add(a: i32, b: i32) -> i32 { return a + b; }

// GOOD
fn add(a: i32, b: i32) -> i32 { a + b }
```

Trailing `return` is fine for early returns from a loop or branch; gratuitous at the end of a function.

## ANTI-20 (SHOULD) — Reinventing builders for two-field structs

If `Foo { a, b }` works as a literal, you don't need `Foo::builder().a(...).b(...).build()`. The builder pattern earns its complexity at >3 optional parameters.

## ANTI-21 (MUST) — `assert_eq!(x, true)` / `assert_eq!(x, false)` instead of `assert!(x)` / `assert!(!x)`

```rust
// BAD
assert_eq!(condition, true);

// GOOD
assert!(condition);
```

`clippy::bool_assert_comparison`.
