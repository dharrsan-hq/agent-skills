---
name: Rust
description: Idiomatic Rust guidance distilled from the official Rust book — ownership, error handling, traits, lifetimes, async, concurrency, and code review rules
---

# Rust

Authoritative source: *The Rust Programming Language* (the official Rust book).
When writing or reviewing Rust code, apply the rules below.

## Core mental model

- Rust's central promise is **memory safety without a garbage collector**, enforced
  at compile time through ownership. If code compiles without `unsafe`, memory
  bugs and data races are impossible by construction. Trust the compiler; prefer
  restructuring code over fighting it with `clone()` or `unwrap()`.
- The three rules of ownership:
  1. Each value has exactly one owner.
  2. When the owner goes out of scope, the value is dropped (`Drop::drop`).
  3. Ownership can be moved or borrowed, but at any moment there is either
     **one mutable reference** or **any number of immutable references** — never both.
- **Move semantics are the default.** Assigning a non-`Copy` value to a new
  variable moves it; the old binding becomes invalid. `String`, `Vec`, `Box`,
  and most structs are moved. `Copy` types (integers, floats, `bool`, `char`,
  tuples/arrays of `Copy` types, shared references) are duplicated instead.
- When designing a function or type, always ask: **owned (`String`), borrowed
  (`&str`), or shared (`&Arc<T>`)?** Borrowed is the most flexible for callers;
  owned only when the function must own (store it, return it derived, or need
  to mutate freely).

## Ownership, borrowing, and lifetimes

- Prefer borrowing over taking ownership. A function should take `&T` or `&mut T`
  unless it genuinely needs to own the value (e.g. it stores it, returns it, or
  needs the value's full lifetime).
- Never take `&mut String` as a parameter when `&str` works; never take `&mut Vec<T>`
  when `&[T]` works. Accept the *broadest view* type (`&str`, `&[T]`, `impl Iterator`)
  and return the *most concrete* type that's convenient.
- **Borrow checker rules in practice:**
  - A mutable borrow requires exclusive access; no other borrow may live
    simultaneously (NLL — non-lexical lifetimes — ends borrows at last use,
    not at scope end, so restructure so borrows don't overlap).
  - You cannot borrow a field mutably while the whole struct is borrowed
    immutably (or vice versa). Splitting borrows into two disjoint field
    borrows *is* allowed: `let (a, b) = (&mut s.x, &mut s.y);` works.
  - Returning a reference from a function requires the reference to derive from
    some input reference (`fn longest<'a>(a: &'a str, b: &'a str) -> &'a str`).
    If a function returns a reference to data created inside it, that's a
    compile error — return owned data instead.
- **Lifetimes:** treat them as annotations of *relationships between references*,
  not as lifetimes of values. Use lifetime elision first; only annotate when the
  compiler can't infer. Common cases:
  - One `&self` parameter → output lifetime follows `self`.
  - Multiple input references → output must be annotated or be `'static`.
  - Structs holding references need a named lifetime: `struct Excerpt<'a> { part: &'a str }`.
  - `'static` means "lives for the whole program" — string literals and boxed
    trait objects (`Box<dyn Error + 'static>`). Do not sprinkle `'static` to
    silence errors; it's often a design smell.
- **Clones are a code smell when used to appease the borrow checker.** Use
  `clone()` deliberately (documented cost), not as a fix. Alternatives: split
  borrows, restructure scopes, use `Rc`/`Arc` for shared ownership, or use
  indices instead of simultaneous references into a container.

## String handling

- `String` = owned, growable, heap-allocated UTF-8. `&str` = borrowed string
  slice (view into a string). Function parameters: `&str` (or `&String` never).
- `s.len()` is byte length, not character count. `String` is a `Vec<u8>` over
  UTF-8. Indexing a string by integer (`s[0]`) is a compile error by design.
- Iterate: `s.chars()` for `char`s, `s.bytes()` for bytes. Grapheme clusters need
  a crate (`unicode-segmentation`). Slicing (`&s[0..4]`) can panic on non-char
  boundaries — use `char_indices()` for safe slicing.
- Concatenation: `+` moves the left operand and is O(n) over the whole string;
  `format!` is clearer for multi-part; `push_str` for appending in place.
- Build strings in loops with `String::with_capacity` + `push_str`.

## Collections

- `Vec<T>`: the workhorse. `vec![]` macro for literals. Pre-allocate with
  `Vec::with_capacity` when size is known. `push`/`pop` amortized O(1);
  `insert`/`remove` at arbitrary index are O(n).
- `HashMap<K, V>`: SipHash by default (DoS-resistant, slower than some hashes).
  For non-adversarial keys, `FxHashMap`/`AHash` (crates) are faster. Access via
  `.entry(k).or_insert(0)` / `or_insert_with` for insert-or-update patterns.
- `HashSet<T>`: set semantics. `BTreeMap`/`BTreeSet` when sorted order matters.
- Iterating a collection: prefer `for x in &vec` (immutable borrow) or
  `&mut vec` (mutable borrow). `for x in vec` consumes the Vec — only when you
  mean it.
- **Prefer iterators over index loops.** Iterator adapters are zero-cost:
  `.map()`, `.filter()`, `.sum()`, `.collect()`, `.fold()`, `.take()`, `.skip()`,
  `.zip()`, `.enumerate()`, `.rev()`, `.chunks()`, `.windows()`. Chain them
  freely; they compile to the same machine code as loops.
- `collect()` is polymorphic: `let v: Vec<i32> = iter.collect();` — annotate the
  type. `collect::<Result<Vec<_>, _>>()` turns a `Result` iterator into a
  single `Result` (fails fast on first `Err`). Same for `collect::<Option<Vec<_>>>()`.

## Error handling

- **`Result<T, E>` for recoverable errors; `panic!` for bugs/contracts.** Panics
  are for programmer errors (invariant violations, out-of-range indexes, calling
  `unwrap` on an impossible `None`), not for expected failure modes.
- **`expect(msg)` over `unwrap()`** — always include a message explaining the
  invariant. Reserve `unwrap()` for tests and truly impossible cases.
- **Never `unwrap()` or `expect()` on untrusted/user input** or in library code.
  Propagate with `?` instead.
- **The `?` operator** propagates errors: `let f = File::open(path)?;` works in
  any function returning `Result`/`Option`. `From` conversions happen
  automatically via `?` — the error type must implement `From<E2>` for the
  propagated error type `E2`.
- **Library vs binary error strategy:**
  - Libraries: define a concrete error enum, derive `Debug` and `Error` (via
    `thiserror` in practice), implement `From` for inner errors, return
    `Result<T, MyError>`. Don't force callers into boxed errors.
  - Binaries: `anyhow` / `Box<dyn Error>` is fine; add context with
    `.context("failed to read config")` when propagating.
- **Error trait design:** implement `std::error::Error` (source, description) —
  derive `thiserror::Error` for free. `#[error("...")]` for messages,
  `#[from]` for auto `From` impls, `#[source]` for chaining.
- Do not swallow errors: `let _ = expr;` on a `Result` discards errors silently.
  Handle, propagate, log, or explicitly ignore with a comment.
- `panic!` macro arguments: `panic!("invalid value: {}", x)` or `panic!("invalid value: {x}")`.

## Traits and generics

- **Traits define shared behavior.** Default methods (`fn draw(&self) { ... }`)
  let implementors override. Trait objects (`Box<dyn Trait>`, `&dyn Trait`)
  enable dynamic dispatch; generics enable static dispatch.
- **Rule of thumb:**
  - Generic `impl Trait`/`<T: Trait>` when the concrete type matters for
    performance, or when you need `Sized` / multiple bounds.
  - Trait objects `dyn Trait` when you need many heterogeneous implementations
    (a `Vec<Box<dyn Draw>>`), or to simplify type signatures.
  - Trait objects are *not* allowed on traits with generic type parameters or
    associated functions without receivers.
- `impl Trait` in argument position: anonymous generic — `fn draw(ui: &mut impl Draw)`.
  In return position: a concrete-but-unnamed type — useful for closures and
  iterators, but can't be used in trait objects or as a return of multiple types.
- **Blanket impls** (`impl<T: Display> ToString for T`) are how the standard
  library composes. Be careful defining your own on foreign types.
- **Orphan rule:** you may implement a trait for a type only if the trait or the
  type is local to your crate. Wrap foreign types in newtypes (`struct Wrapper(Vec<T>)`)
  to implement traits on them — deref with `Deref`/`DerefMut`.
- **Associated types** (`type Item;`) vs generic parameters: use associated types
  when there's exactly one type per implementor (`Iterator::Item`); use generics
  when multiple types per implementor are valid (`From<T>`).
- **Core trait checklist:** implement `Debug` (derive), `Clone` (derive), `PartialEq`
  (derive), `Default` when sensible, `Display` manually. `PartialOrd`/`Ord` for
  sortable types. `Send`/`Sync` are auto-derived for threadsafe types.

## Structs, enums, and pattern matching

- **Structs:** named-field structs for data; unit structs for type-level markers;
  tuple structs for thin wrappers. `#[derive(Debug, Clone, PartialEq)]` is the
  default starting point.
- Update syntax: `let user2 = User { email: String::from("x"), ..user1 };` —
  moves the *remaining* fields from `user1` if they're non-`Copy`.
- **Enums model variants that can be one of several.** Use enums over bools/strings
  for state. Example: `enum State { Idle, Running(Duration), Failed(String) }` —
  carries data per-variant.
- `Option<T>` and `Result<T, E>` are enums — the standard for absence and
  failure. Avoid `Option<Box<T>>` duplication: use `#[derive(Debug)]` + `Option`
  with `Box<T>` directly.
- **Match exhaustively.** The compiler enforces all variants; add `_` only when
  genuinely unimportant. Prefer match guards (`Some(x) if x > 0`) over nested ifs.
- `if let` / `let else` / `while let` for single-pattern matches:
  - `if let Some(x) = opt { ... }` — one case, no else.
  - `let Some(x) = opt else { return; };` — return/continue on the *not* case,
    keeps the happy path flat.
- Destructuring: `let Point { x, y } = p;`, tuple patterns, nested patterns,
  `@` bindings (`n @ 1..=5`), or-patterns (`A | B`), reference patterns (`&val`).
- Match ergonomics: matching on a reference automatically binds by reference —
  `match &opt { Some(x) => ... }` gives `x: &T`.

## Modules, crates, workspaces

- **Module system:** a crate is a compilation unit; modules are namespaces inside
  it. `mod` declares; `pub` exposes. `use crate::module::Item` for imports;
  `use super::` for parent; `use self::` for same-module.
- `src/main.rs` (binary crate) + `src/lib.rs` (library crate) — the idiomatic
  split: put logic in `lib.rs`, keep `main.rs` thin (parse args, call library,
  handle errors). This makes the logic testable.
- `pub use` re-exports flatten a public API: `pub use self::errors::MyError;`.
- Visibility: `pub` is crate-visible (and exported if re-exported); `pub(crate)`
  for internal-only; `pub(super)` for parent-only; default is private to the
  module tree.
- Cargo workspaces: multiple related crates under one `Cargo.toml` `[workspace]`
  members list; shared `Cargo.lock` and `target/`. Good for splitting large
  projects into compile-time-parallelizable crates.
- Crates.io conventions: `Cargo.toml` `[dependencies]` with version specs
  (`"2.3"`, `">=1.2, <2.0"`). `cargo update` bumps within the semver range.
  Lockfile `Cargo.lock` is committed for binaries, not for libraries.

## Smart pointers

- `Box<T>`: heap allocation, single owner. Use for recursive types, large
  by-value structs, or trait objects. Deref: `*b` or auto-deref.
- `Rc<T>` / `Arc<T>`: shared ownership with reference counting. `Rc` is
  single-threaded; `Arc` is atomic (thread-safe). Use when multiple owners
  genuinely share one value — not as a general-purpose bypass for the borrow checker.
- `RefCell<T>` / `Cell<T>`: interior mutability — borrow rules checked at runtime.
  `RefCell` single-threaded (`Rc<RefCell<T>>`); `Mutex<T>`/`RwLock<T>` for
  multi-threaded (`Arc<Mutex<T>>`). Runtime panics if borrow rules are violated.
- Avoid `Rc<RefCell<>>` cycles → memory leaks. `Weak<T>` + `upgrade()` breaks
  cycles (use for parent/child or cache references).
- `Cow<'a, T>` (clone-on-write): returns borrowed data when possible, owned when
  mutation is needed. Great for functions that may or may not modify input.
- `Deref`/`DerefMut` coercion: `&String` → `&str`, `&Vec<T>` → `&[T]`,
  `&Box<T>` → `&T`. Implement `Deref` only for newtypes/smart pointers.

## Concurrency and async

- **Ownership makes concurrency safe**: values moved into threads are owned by
  them; shared values need `Arc`/`Mutex`. `Send` = transferable across threads;
  `Sync` = safely shareable by reference (`&T: Send`).
- `std::thread::spawn(move || { ... })` — the `move` closure captures the
  environment by value. Join with `handle.join().unwrap()` for panics.
- Channels: `std::sync::mpsc` (multiple producer, single consumer) —
  `channel()` gives `(Sender, Receiver)`; clones of `Sender` for multiple
  producers; dropping all senders closes the channel (`recv()` returns `Err`).
- `Mutex<T>`: lock before access — `let mut guard = m.lock().unwrap();`.
  RAII: the lock releases when the guard drops. `RwLock<T>` for read-heavy.
  Poisoning: a panic while holding the lock poisons it — `unwrap()` on a
  poisoned lock propagates; use `.unwrap_or_else(|e| e.into_inner())` to recover.
- **`Arc<Mutex<T>>` is the standard shared-state pattern.** `Arc` for the
  shared pointer, `Mutex` for mutation.
- **Async:**
  - `async fn` returns a `Future` — it does *nothing* until polled by an
    executor. `fut.await` yields control to the executor until ready.
  - Async Rust is cooperative: `await` is the only yield point. A long
    CPU-bound loop inside an async fn blocks the executor — use `tokio::spawn`
    + `spawn_blocking` (or `rayon`) for CPU work, or `yield_now()`.
  - Futures are lazy: creating one doesn't run it. You must `.await` it or
    `spawn` it onto a runtime.
  - Runtime choice: `tokio` is the de facto standard; `smol`/`async-std` are
    alternatives. `tokio::main` for the entry point.
  - `async fn` in traits: supported via `async-trait` crate or native async
    trait in newer Rust. Note: native async traits in traits are not dyn-compatible
    (use `async-trait` or manual boxing for `dyn`).
  - `tokio::select!` waits for multiple futures concurrently — first ready wins,
    others are cancelled (cancellation is a real hazard: clean up with `Drop`).
  - **Avoid blocking the executor** — `std::sync::Mutex` in async code is fine
    for short critical sections (contended/long-held locks → `tokio::sync::Mutex`).
  - `Send` bounds on async: `async fn` futures must be `Send` to spawn across
    threads — non-`Send` data (e.g. `Rc`) held across `.await` is a compile error.
- **Closures:** `Fn` (call many times, read-only), `FnMut` (call many times,
  mutate captures), `FnOnce` (call once, consumes captures). `move` closures
  take ownership of captured variables — required for `thread::spawn` and
  async tasks.
- **Rayon** for data parallelism (`par_iter().map(...).collect()`), independent
  of async runtimes.

## Unsafe and FFI

- Five superpowers: dereference raw pointers, call `unsafe` functions/FFI,
  access/modify `static mut`, implement `unsafe` traits, access union fields.
- `unsafe` blocks don't disable the borrow checker — they *unlock* unchecked
  operations. The invariant is yours to maintain.
- Minimize `unsafe` surface: wrap it in a safe abstraction with a clear contract,
  documented invariants. Review `unsafe` code with extra care.
- FFI: `#[repr(C)]` for structs shared with C; `extern "C"` for ABI;
  `extern "C" { ... }` blocks for imported functions; `#[no_mangle]` for
  exported symbols. CString/CStr for C string interop.

## Tooling and testing

- **Clippy is mandatory** — `cargo clippy` before every commit. Treat warnings
  seriously; fix or explicitly allow with a reason:
  `#[allow(clippy::lint_name)] /* reason */`.
- **rustfmt is mandatory** — `cargo fmt` (line length 100 by default). Format
  before committing, don't hand-style.
- `cargo check` for fast type checking; `cargo build` for full compilation;
  `cargo run` for binaries; `cargo test` for tests; `cargo doc --open` for docs.
- **Testing:**
  - Unit tests: `#[cfg(test)] mod tests` at the bottom of the same file — test
    private functions, one module per production module.
  - Integration tests: `tests/` directory, one crate per file — tests the public
    API only.
  - Doc tests: examples in `///` doc comments are compiled and run by `cargo test` —
    keep them correct, they're the first thing users see.
  - Use `assert!`, `assert_eq!`, `assert_ne!`, and `#[should_panic]` for panics;
    `Result<T, E>` in tests enables `?` in test bodies.
  - Test names: `fn test_name_describes_behavior()` — `should_...` or
    `given_..._when_..._then_...` conventions.
- Panic strategy: `panic = "abort"` in `[profile.release]` for smaller binaries,
  but it disables unwinding-based cleanup; keep `unwind` if you need to catch
  panics or use FFI.
- Release profile: `[profile.release] lto = "thin"`, `codegen-units = 1`,
  `opt-level = 3` for max performance; `opt-level = "z"` for size.

## Idioms and style checklist

- Prefer `&str` over `&String`, `&[T]` over `&Vec<T>`, `impl Iterator` over
  concrete iterator types.
- Prefer composition over inheritance — Rust has no inheritance; use traits,
  enums, and delegation.
- Newtype pattern: `struct Meters(f64);` — zero-cost type safety, distinct types
  for distinct units.
- Avoid `pub` fields on structs with invariants; use private fields + constructor
  + accessor methods (`impl` blocks) to keep invariants held.
- `Default` + struct update syntax for configurability: `Config { width: 80, ..Default::default() }`.
- Method naming convention: `get_*` returns a borrowed value, `into_*` consumes,
  `as_*` returns a reference, `to_*` creates a cheap copy/convert. `is_*` returns bool.
- `?` chains over nested `match` statements — keep error paths linear.
- `Debug` output for humans: `{:?}` and `{:#?}` (pretty). Don't rely on `Debug`
  for stable output; use `Display` for user-facing.
- Use `iter().enumerate()` instead of `for i in 0..len { vec[i] }` — the latter
  panics on mutation and can't be optimized as well.
- Use `matches!` macro for boolean pattern checks: `if matches!(state, State::Running(_))`.
- `let-else` over `if let` with `return` in the else branch.
- Don't prefix enum variants with the enum name: `Color::Red`, not `Red`.
- `Vec::with_capacity(n)` before `push` loops with known n.
- Use `#[non_exhaustive]` on public enums/structs that may gain variants later —
  prevents downstream breakage.

## Code review checklist

- [ ] No `unwrap()`/`expect()` on user input or in library code; `?` used instead.
- [ ] No unnecessary `clone()` — especially to appease the borrow checker.
- [ ] No `&mut String`/`&mut Vec<T>` where `&str`/`&[T]` would do.
- [ ] No `panic!` for expected failures — only `Result`/`Option` propagation.
- [ ] Errors are wrapped with context (`anyhow::Context` or `thiserror` messages).
- [ ] `unwrap()` on `Mutex::lock`/`PoisonError` — is it intended, or should the
      error be handled?
- [ ] Every `unsafe` block has a documented safety comment and a minimal surface.
- [ ] Async code: no blocking calls (`std::thread::sleep`, `std::sync::Mutex` for
      long-held locks, heavy CPU work) inside async tasks.
- [ ] Closures: correct `move` usage; no unintended capture of large variables.
- [ ] All `pub` items have doc comments (`///`); doc examples compile.
- [ ] `cargo clippy` clean; `cargo fmt` applied.
- [ ] Exhaustive `match` — no `_` arm that silently swallows new variants.
- [ ] `Send`/`Sync` bounds verified for anything spawned across threads.
- [ ] Tests: unit tests cover behavior, not just happy paths; `#[should_panic]`
      and error cases covered.
- [ ] Newtype/enums used where raw types/strings would be ambiguous.
