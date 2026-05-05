# Shared Mutable State Across Threads in Rust

## Full Code

```rust
use rand::RngExt;
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let numbers = Arc::new(Mutex::new(Vec::new()));
    let mut handles = vec![];

    for _ in 0..10 {
        let a = numbers.clone();
        handles.push(thread::spawn(move || {
            let random_num = rand::rng().random_range(0..100);
            let mut vec = a.lock().unwrap();
            vec.push(random_num);
        }));
    }

    for h in handles {
        h.join().unwrap();
    }

    println!("{:?}", numbers.lock().unwrap());
}
```

## Explanation

This code spawns 10 threads, each of which generates a random number and appends it to a shared vector.

### Line by line

```rust
for _ in 0..10 {
```
Loops 10 times. `_` discards the loop index since it's unused.

```rust
let a = numbers.clone();
```
Clones the smart pointer (likely an `Arc<Mutex<Vec<_>>>`). This creates a new reference-counted pointer to the **same** underlying data — it does **not** deep-copy the vector. Each thread gets its own `Arc` handle.

```rust
handles.push(thread::spawn(move || {
```
Spawns a new OS thread. The `move` keyword transfers ownership of captured variables (here, `a`) into the closure, which is required because the thread may outlive the current scope.

```rust
let random_num = rand::rng().random_range(0..100);
```
Generates a random integer in `[0, 100)` using the `rand` crate.

```rust
let mut vec = a.lock().unwrap();
vec.push(random_num);
```
- `.lock()` acquires the `Mutex` lock, blocking until no other thread holds it — this ensures only one thread modifies the vector at a time.
- `.unwrap()` panics if the mutex is **poisoned** (i.e., a previous thread panicked while holding the lock).
- `vec.push(random_num)` appends the number. The lock is released automatically when `vec` drops at end of scope.

## The Pattern

```
Arc<Mutex<Vec<i32>>>
 │       │
 │       └─ guarantees exclusive access (one writer at a time)
 └─────────── allows shared ownership across threads
```

This is the canonical Rust pattern for **shared mutable state across threads** — `Arc` for shared ownership, `Mutex` for interior mutability.
