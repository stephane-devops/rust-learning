# A Complete Tutorial on Rust Closures

In Rust, **closures** are anonymous functions that can capture their environment. They are more flexible than regular functions because they can "close over" (capture) variables from the scope in which they are defined.

---

## 1. Basic Syntax
The syntax for closures is slightly different from `fn` declarations:
- Parameters are placed between vertical pipes `| |`.
- The body can be a single expression or a block `{ }`.
- Type annotations are optional (Rust infers them).

```rust
fn main() {
    // A simple closure that adds 1 to its input
    let add_one = |x: i32| x + 1;

    // A closure with multiple parameters and a block
    let calculate = |a: i32, b: i32| {
        println!("Calculating...");
        a * b + 10
    };

    println!("Result: {}", add_one(5));      // Output: 6
    println!("Calculate: {}", calculate(3, 4)); // Output: 22
}
```

---

## 2. Capturing the Environment
This is the "killer feature" of closures. They can access variables from the outer scope in three ways:

1.  **Borrowing Immutably**: The closure takes a reference (`&T`).
2.  **Borrowing Mutably**: The closure takes a mutable reference (`&mut T`).
3.  **Moving (Taking Ownership)**: The closure takes ownership (`T`).

```rust
fn main() {
    let mut list = vec![1, 2, 3];

    // 1. Immutable borrow
    let only_borrows = || println!("From closure: {:?}", list);
    only_borrows();

    // 2. Mutable borrow (requires 'mut' on the closure variable)
    let mut borrows_mutably = || list.push(4);
    borrows_mutably();
    println!("After mutation: {:?}", list);

    // 3. Taking ownership (using the 'move' keyword)
    let consumes = move || println!("I own the list now: {:?}", list);
    consumes();
    // println!("{:?}", list); // ERROR: 'list' has been moved!
}
```

---

## 3. Closure Traits (`Fn`, `FnMut`, `FnOnce`)
Rust automatically implements one or more of these traits for every closure, depending on how it uses the captured values:

*   **`Fn`**: Borrows values immutably. Can be called multiple times.
*   **`FnMut`**: Borrows values mutably. Can be called multiple times.
*   **`FnOnce`**: Consumes values (moves them). Can only be called **once**.

---

## 4. Closures as Arguments
You often pass closures to functions (like `map`, `filter`, or custom functions). You use the traits mentioned above as generic bounds.

```rust
fn apply_to_3<F>(f: F) -> i32 
where 
    F: Fn(i32) -> i32 
{
    f(3)
}

fn main() {
    let double = |x| x * 2;
    let result = apply_to_3(double);
    println!("Double 3 is: {}", result); // 6
}
```

---

## 5. Returning Closures
Because closures don't have a fixed size (they are "unsized"), you must return them inside a `Box` if you want to return them from a function.

```rust
fn create_adder(increment: i32) -> Box<dyn Fn(i32) -> i32> {
    Box::new(move |x| x + increment)
}

fn main() {
    let add_five = create_adder(5);
    println!("5 + 10 = {}", add_five(10)); // 15
}
```

---

## Summary Comparison
| Feature | Functions (`fn`) | Closures (`||`) |
| :--- | :--- | :--- |
| **Name** | Must be named | Anonymous |
| **Capturing** | Cannot capture environment | Can capture environment |
| **Inference** | Requires type annotations | Can infer types |
| **Usage** | Global logic, APIs | Iterators, callbacks, local logic |
