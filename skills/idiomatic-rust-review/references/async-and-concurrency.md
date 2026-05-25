# Async & Concurrency

Sources: [Cancelling async Rust](https://sunshowers.io/posts/cancelling-async-rust/), [Async Rust can be a pleasure to work with (without Send + Sync + 'static)](https://emschwartz.me/async-rust-can-be-a-pleasure-to-work-with-without-send-sync-static/), [Why choose async/await over threads?](https://notgull.net/why-not-threads/), [Rust's Sneaky Deadlock With if let Blocks](https://brooksblog.bearblog.dev/rusts-sneaky-deadlock-with-if-let-blocks/), [Await a minute](https://docs.rs/dtolnay/0.0.3/dtolnay/macro._01__await_a_minute.html).

---

## ASYNC-01 (MUST) — Don't hold a `std::sync::Mutex` guard across `.await`

The guard isn't `Send`, and even when it is, blocking the executor on a contended mutex defeats the purpose of async.

```rust
// BAD — guard lives across the await; type doesn't compile in multi-threaded executor,
//       and even with single-threaded it stalls the runtime
let guard = self.cache.lock().unwrap();
let value = guard.get(&key).cloned();
slow_io().await;          // ← BAD
use_it(value);

// GOOD — drop before await
let value = {
    let guard = self.cache.lock().unwrap();
    guard.get(&key).cloned()
};                        // guard dropped
slow_io().await;

// GOOD — use an async-aware mutex when you really need to hold across await
let guard = self.cache.lock().await;   // tokio::sync::Mutex
slow_io().await;
```

Flag this aggressively. It's the #1 source of async deadlocks.

## ASYNC-02 (MUST) — Cancellation safety: every `.await` is a potential drop point

When an async function is dropped at an `await`, partial work disappears. Functions that mutate shared state across awaits must be written so an early drop doesn't leave invariants broken.

```rust
// BAD — if dropped between increment and decrement, counter is wrong forever
async fn do_work(state: &State) {
    state.active.fetch_add(1, Ordering::Relaxed);
    do_slow_thing().await;
    state.active.fetch_sub(1, Ordering::Relaxed);
}

// GOOD — RAII guard handles cleanup even on cancel
async fn do_work(state: &State) {
    let _g = ActiveGuard::new(&state.active);
    do_slow_thing().await;
}  // _g dropped here, decrement runs even if .await was cancelled
```

Document cancellation behavior in rustdoc. The standard answer for "what happens if this future is cancelled" should not be "I haven't thought about it."

## ASYNC-03 (SHOULD) — Don't reach for `Arc<Mutex<...>>` reflexively in async code

Many "I need shared state in tasks" patterns are better expressed as:
- A channel (`tokio::sync::mpsc`) feeding a single actor task that owns the state.
- An `Arc<T>` of an *immutable* configuration plus per-task local state.
- `Arc<RwLock<T>>` when reads dominate writes.

```rust
// OK — but consider channels
let state = Arc::new(Mutex::new(State::new()));

// OFTEN BETTER — actor pattern
let (tx, mut rx) = mpsc::channel::<Cmd>(64);
tokio::spawn(async move {
    let mut state = State::new();
    while let Some(cmd) = rx.recv().await { state.apply(cmd); }
});
```

## ASYNC-04 (SHOULD) — Avoid blocking calls in async functions

`std::fs::read`, `std::thread::sleep`, large CPU loops — these stall the executor. Move them to `tokio::task::spawn_blocking` or use async variants.

```rust
// BAD
async fn load(p: PathBuf) -> io::Result<Vec<u8>> {
    std::fs::read(p)                    // blocks the executor thread
}

// GOOD
async fn load(p: PathBuf) -> io::Result<Vec<u8>> {
    tokio::fs::read(p).await
}

// GOOD — for CPU work
let result = tokio::task::spawn_blocking(|| expensive_cpu_work(input)).await?;
```

## ASYNC-05 (MUST) — Don't `.unwrap()` `JoinHandle`s without thought

`spawn(...).await.unwrap()` propagates *panic* from the spawned task. That may be what you want — or it may be that the task can fail for benign reasons (cancellation, runtime shutdown). Match on `JoinError::is_panic()` / `is_cancelled()` explicitly.

## ASYNC-06 (SHOULD) — Prefer `?Send` / `?Sync` bounds when the caller can choose

A library that hardcodes `T: Send + Sync + 'static` on every type forces users into specific runtime models. When the constraint isn't actually needed, relax it.

```rust
// BAD — forces 'static + Send on every callback
pub fn on_event<F: Fn() + Send + 'static>(&self, f: F) { … }

// GOOD — let the caller pick the bound
pub fn on_event<F: Fn()>(&self, f: F) { … }
// and document which executor model is intended
```

For trait objects, prefer `Box<dyn Future<Output = T>>` over `Box<dyn Future<Output = T> + Send + 'static>` until you actually need the bound.

## ASYNC-07 (SHOULD) — Use `select!` and `join!` correctly

- `tokio::join!` waits for *all* futures; semantically parallel.
- `tokio::try_join!` short-circuits on first error.
- `tokio::select!` polls until *one* completes; the others are dropped (read: cancelled).

```rust
// GOOD — fan-out fetch
let (a, b, c) = tokio::try_join!(fetch_a(), fetch_b(), fetch_c())?;

// GOOD — timeout
tokio::select! {
    res = work() => res,
    _ = tokio::time::sleep(timeout) => Err(Timeout),
}
```

Flag `select!` arms where dropping the loser would corrupt state (see ASYNC-02).

## ASYNC-08 (SHOULD) — Bound channel sizes in production code

Unbounded channels (`mpsc::unbounded_channel`) hide backpressure issues. Use bounded channels and let `send` await when the consumer is slow — that's the signal.

```rust
// SUSPICIOUS in production
let (tx, rx) = mpsc::unbounded_channel();

// GOOD
let (tx, rx) = mpsc::channel(1024);
```

## ASYNC-09 (MUST) — Don't `block_on` inside an async context

Calling `Runtime::block_on` from within an async function (whether on the same runtime or nested) deadlocks or panics depending on the runtime. If you genuinely need to bridge sync and async, use `spawn_blocking` + a `oneshot` channel.

## ASYNC-10 (SHOULD) — `async fn` in traits: prefer `async fn` (stable since 1.75) or `async-trait`

Modern Rust supports `async fn` directly in traits. Use it unless you need dyn dispatch, in which case the `async-trait` crate is still the answer (it boxes the future).

```rust
// GOOD (Rust 1.75+)
trait Storage {
    async fn put(&self, k: &str, v: &[u8]) -> Result<(), Error>;
}

// GOOD (needs dyn Storage)
#[async_trait]
trait Storage {
    async fn put(&self, k: &str, v: &[u8]) -> Result<(), Error>;
}
```

## ASYNC-11 (MAY) — Don't async-everything

If a function does no I/O and no awaiting, it doesn't need to be `async`. Marking it `async` only burns the caller's life on a needless `.await`.

```rust
// BAD
async fn add(a: i32, b: i32) -> i32 { a + b }

// GOOD
fn add(a: i32, b: i32) -> i32 { a + b }
```

## ASYNC-12 (SHOULD) — Structured concurrency: prefer `JoinSet` over fire-and-forget `spawn`

A bare `tokio::spawn(...)` whose `JoinHandle` is dropped is a "leak" — you can't await it, cancel it, or know when it finished. Prefer `JoinSet` or store the handles.

```rust
// FRAGILE
for item in items { tokio::spawn(process(item)); }

// BETTER
let mut set = tokio::task::JoinSet::new();
for item in items { set.spawn(process(item)); }
while let Some(res) = set.join_next().await { res??; }
```

## ASYNC-13 (MUST) — `Pin` is a contract, not a suggestion

Manual `Pin<&mut Self>` implementations are easy to get wrong. Reach for `pin-project-lite` / `pin-project`. If reviewing handwritten `unsafe impl Unpin`, demand a `// SAFETY:` comment explaining the pinning invariants.

## ASYNC-14 (MAY) — Threads still exist

Async is overkill for tasks that don't need to wait on many concurrent I/O operations. For "do CPU work in the background and return," `std::thread::spawn` + a `oneshot::channel` (or `crossbeam::channel`) is simpler than async.
