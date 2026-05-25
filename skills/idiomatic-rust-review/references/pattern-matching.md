# Pattern Matching

Sources: [Level Up your Rust pattern matching](https://blog.cuongle.dev/p/level-up-your-rust-pattern-matching), [Rust's Sneaky Deadlock With if let Blocks](https://brooksblog.bearblog.dev/rusts-sneaky-deadlock-with-if-let-blocks/), [Russian Dolls and clean Rust code](https://web.archive.org/web/20220126183049/https://blog.mgattozzi.dev/russian-dolls/), [Pretty State Machine Patterns in Rust](https://hoverbear.org/2016/10/12/rust-state-machine-pattern/), [Iteration patterns for Result & Option](http://xion.io/post/code/rust-iter-patterns.html).

---

## MATCH-01 (MUST) — Don't write wildcard arms on closed enums you control

Wildcard arms silently absorb new variants when the enum grows. For an enum you own, list every variant — the compiler tells you when something new needs handling.

```rust
// BAD — adding `Event::Reconnected` later silently routes to the wildcard
match event {
    Event::Connected => connect(),
    Event::Disconnected => disconnect(),
    _ => {},
}

// GOOD
match event {
    Event::Connected => connect(),
    Event::Disconnected => disconnect(),
    Event::MessageReceived(_) => {},
}
```

For `#[non_exhaustive]` foreign enums, a wildcard is required.

## MATCH-02 (SHOULD) — `if let` for one variant, `match` for two or more

```rust
// BAD
match maybe { Some(x) => use_it(x), None => {} }

// GOOD
if let Some(x) = maybe { use_it(x); }
```

For three variants of one enum, `match` reads better than chained `if let else if let`.

## MATCH-03 (SHOULD) — `let-else` for early return on `None` / `Err`

```rust
// VERBOSE
let value = match get_value() {
    Some(v) => v,
    None => return,
};

// IDIOMATIC
let Some(value) = get_value() else { return };
```

Stable since 1.65. Replaces the "match, return on None, otherwise bind" boilerplate.

## MATCH-04 (SHOULD) — `matches!` for boolean checks

```rust
// VERBOSE
let is_ready = match state { State::Ready | State::Idle => true, _ => false };

// IDIOMATIC
let is_ready = matches!(state, State::Ready | State::Idle);
```

## MATCH-05 (MUST) — Pattern temporaries live until the end of the `if let` block

The lifetime trap: in `if let Some(x) = mutex.lock().get(&k)`, the lock guard is the temporary, and it lives for the *entire body* of the `if let`. This is why `if let` chains can deadlock with mutexes and hold borrows longer than expected.

```rust
// BAD — lock held for the whole branch
if let Some(value) = self.cache.lock().get(&key).cloned() {
    expensive(value);   // lock still held
}

// GOOD — bind explicitly, scope-control the temporary
let cached = self.cache.lock().get(&key).cloned();
if let Some(value) = cached { expensive(value); }
```

The same trap applies to `match` scrutinees and `while let`.

## MATCH-06 (SHOULD) — Use binding `@` patterns to capture and check together

```rust
// VERBOSE
match n {
    n if n < 0 => negative(n),
    n if n > 100 => big(n),
    n => normal(n),
}

// IDIOMATIC
match n {
    x @ ..0    => negative(x),
    x @ 101..  => big(x),
    x          => normal(x),
}
```

## MATCH-07 (SHOULD) — Destructure in function parameters when it helps

```rust
// OK
fn area(p: Point) -> f64 { p.x * p.y }

// CLEANER WHEN THE POINT FIELDS ARE THE WHOLE STORY
fn area(Point { x, y }: Point) -> f64 { x * y }
```

Don't overuse — when the struct name carries information ("this is a `Point`, not just coordinates"), keep it named.

## MATCH-08 (SHOULD) — Combine arms with `|` instead of duplicating bodies

```rust
// BAD
match c {
    'a' | 'e' | 'i' | 'o' | 'u' => …,
    'A' => vowel(),
    'E' => vowel(),
    'I' => vowel(),
    _ => consonant(),
}

// GOOD
match c {
    'a' | 'e' | 'i' | 'o' | 'u' | 'A' | 'E' | 'I' | 'O' | 'U' => vowel(),
    _ => consonant(),
}
```

## MATCH-09 (MAY) — Reach for `Option::map` / `Result::map` over `match`

```rust
// VERBOSE
let upper = match name {
    Some(s) => Some(s.to_uppercase()),
    None => None,
};

// IDIOMATIC
let upper = name.map(|s| s.to_uppercase());
```

Same for `and_then`, `or_else`, `unwrap_or`, `unwrap_or_else`, `ok_or_else`, etc. Learn the combinator family. But: when the body is large (>5 lines) or involves error handling, `match` is clearer.

## MATCH-10 (SHOULD) — `?` instead of `match` for `Result`/`Option` propagation

Already covered in error handling but worth repeating here. Most "`match` returning `Err(e)`" or "`match` returning `None`" patterns should just be `?`.

## MATCH-11 (MAY) — Use refutable patterns in `for` loops to filter

```rust
// IDIOMATIC
for Ok(line) in reader.lines() { process(line); }   // silently skips Err

// MORE EXPLICIT
for line in reader.lines() {
    match line {
        Ok(s) => process(s),
        Err(e) => tracing::warn!(?e, "skipping bad line"),
    }
}
```

The first form is concise but can mask bugs; prefer the explicit form unless dropping errors is genuinely fine.

## MATCH-12 (SHOULD) — Avoid `panic!` from unreachable arms; use `unreachable!()` or refactor

If you're writing `match` and one arm is "this can't happen," reach for `unreachable!()` *and* leave a comment explaining the invariant. Better: refactor so the impossible case isn't representable.

```rust
// OK with explanation
match state {
    State::Open(conn) => conn.send(msg),
    State::Closed => unreachable!("send() requires open state; checked by guard"),
}

// BETTER — separate types per state
fn send(open: &OpenConnection, msg: Msg) { … }
```

## MATCH-13 (SHOULD) — Pattern matching in closures with `|(a, b)|`

```rust
// VERBOSE
pairs.iter().map(|pair| { let (k, v) = pair; format!("{k}={v}") });

// IDIOMATIC
pairs.iter().map(|(k, v)| format!("{k}={v}"));
```
