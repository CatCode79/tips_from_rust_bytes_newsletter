# Tips from the [Rust Bytes Newsletter](https://weeklyrust.substack.com/)

### [#[repr(transparent)]](https://doc.rust-lang.org/nomicon/other-reprs.html#reprtransparent) + [#[repr(C)]](https://doc.rust-lang.org/nomicon/other-reprs.html#reprc) for [newtype](https://doc.rust-lang.org/rust-by-example/generics/new_types.html) [FFI](https://doc.rust-lang.org/nomicon/ffi.html) safety

This guarantees the layout is exactly the same as the inner type, lets you do transmute-free conversions, and is the idiomatic way to wrap raw handles/pointers.

```rust
#[repr(transparent)]
pub struct Handle(NonZeroU32);

#[repr(transparent)]
pub struct Opaque(*mut c_void);
```

You can play around with the code on  [Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=3a836c1f1ce6eb235e0cec015d4a5f48).

---

### [ManuallyDrop](https://doc.rust-lang.org/std/mem/struct.ManuallyDrop.html) + [ptr::write](https://doc.rust-lang.org/std/ptr/fn.write.html) for move out without [dro](https://doc.rust-lang.org/rust-by-example/trait/drop.html)p [patterns](https://rust-unofficial.github.io/patterns/idioms/dtor-finally.html)

Want to move a value out of a struct without dropping the rest?

Combine ManuallyDrop with ptr::read/write:

```rust
let mut thing = ManuallyDrop::new(my_struct);
let field = ptr::read(&thing.field); // moves out
// now you can drop the rest manually or forget
```

I use this constantly when implementing arenas, intrusive linked lists, or custom Box-like types.

---

### [std::ptr::addr_of!](https://doc.rust-lang.org/std/ptr/macro.addr_of.html) and [addr_of_mut!](https://doc.rust-lang.org/std/ptr/macro.addr_of_mut.html): your new best friends for [field projection](https://github.com/BennoLossin/rfcs/blob/field-projection-v2/text/3735-field-projections.md)

When you need a raw pointer to a field without going through a reference to avoid stacked borrows or temporary references, use:

```rust
let ptr = std::ptr::addr_of!((*some_ptr).field);
let mut_ptr = std::ptr::addr_of_mut!((*some_ptr).field);
```

This is UB-free in cases where &(*some_ptr).field would create illegal intermediate references (very common in unsafe FFI, kernel, or intrusive data structures).

Check out and run the example on [Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=af52f98af54126950120966b6656fe05).

---

### [#[track_caller]](https://doc.rust-lang.org/reference/attributes/codegen.html#the-track_caller-attribute) + [std::panic::Location](https://doc.rust-lang.org/std/panic/struct.Location.html) for god-tier debugging in libraries

Want your library’s error messages/panics/logs to point to the user’s code, not your internal helper?

```rust
#[track_caller]
pub fn my_library_function() {
    let location = std::panic::Location::caller();
    println!("called from {}:{}", location.file(), location.line());
}
```

I put this on every public-facing helper that could fail. Makes debugging 10× nicer for downstream users.

Check out and run the example on [Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=a5a7bed99d817a4f08ce5af1ec3884cd).

---

### Embed static data with [include_str!](https://doc.rust-lang.org/std/macro.include_str.html)

Need to bundle static text with your program? Use include_str! to read a file at compile time and embed it directly into the binary:
```rust
fn main() {
    const TEXT: &str = include_str!("static-data.txt");
    let program = include_str!("main.rs");
}
```
The file is read at build time, not at runtime. If you need to read a file while the program is running, use fs::read_to_string instead.

A particularly neat trick is to reuse your README.md file as crate documentation:
```rust
#![doc = include_str!("../README.md")]
```
Now you only have to write your documentation once. The same file powers your GitHub page, your cargo doc output, and your page on crates.io, including examples and doctests.

---

### Prefer [From](https://doc.rust-lang.org/std/convert/trait.From.html)/[TryFrom](https://doc.rust-lang.org/std/convert/trait.TryFrom.html) over [as](https://doc.rust-lang.org/std/keyword.as.html)

Using as for numeric conversions can silently truncate values when the destination type is smaller. That’s rarely what you want.

For infallible conversions, use From:
```rust
let output = u64::from(input);
```
If this compiles, the conversion is guaranteed to be sound. Rust won’t let you accidentally convert a larger type into a smaller one.

For conversions that may or may not succeed depending on the value, use TryFrom:
```rust
if let Ok(output) = u8::try_from(input) {}
```
TryFrom returns a Result:Ok if the value fits, Err if it doesn’t.

You can run the code on [Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=db715f56ed459ea0ac1c9573752a05dd).

---

### [matches!](https://doc.rust-lang.org/std/macro.matches.html) for Fast, Readable Pattern Checks

matches! is a macro that evaluates to true if a value fits a given pattern. It’s essentially a compact, expression-based alternative to if let or a full match when you only care about a boolean result.

Instead of writing
```rust
enum State {
    Ready,
    Busy(u32)
}

fn is_busy(s: &State) -> bool {
    if let State::Busy(_) = s {
        true
    } else {
        false
    }
}
```

You write
```rust
enum State {
    Ready,
    Busy(u32)
}

fn is_busy_with_matches(s: &State) -> bool {
    matches!(s, State::Busy(_))
}
```

It also supports guards for additional constraints.
```rust
fn is_heavily_busy(s: &State) -> bool {
    matches!(s, State::Busy(n) if n > 10)
}
```

You can run the code on [Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=8dd7677b8efa6ed59ea21bea2eba7268).

---

### [Const generics](https://doc.rust-lang.org/reference/items/generics.html#const-generics) for Type-Safe Matrix Operations

Const Generics enable parameterizing types with compile-time constants, such as array or matrix dimensions.

It’s ideal for performance-critical applications like numerics, embedded systems, or games, allowing fixed-size matrices to be allocated without the overhead associated with dynamic vectors.
```rust
#[derive(Clone, Copy)]
struct Matrix<const ROWS: usize, const COLS: usize> {
    data: [[f32; COLS]; ROWS],
}

impl<const ROWS: usize, const COLS: usize> Matrix<ROWS, COLS> {
    fn zero() -> Self {
        Matrix {
            data: [[0.0; COLS]; ROWS],
        }
    }
}

fn add<const ROWS: usize, const COLS: usize>(
    a: &Matrix<ROWS, COLS>,
    b: &Matrix<ROWS, COLS>,
) -> Matrix<ROWS, COLS> {
    let mut result = Matrix::zero();
    for i in 0..ROWS {
        for j in 0..COLS {
            result.data[i][j] = a.data[i][j] + b.data[i][j];
        }
    }
    result
}

fn main() {
    let mat1: Matrix<2, 2> = Matrix {
        data: [[1.0, 2.0], [3.0, 4.0]],
    };
    let mat2: Matrix<2, 2> = Matrix {
        data: [[5.0, 6.0], [7.0, 8.0]],
    };

    let sum = add(&mat1, &mat2);

    println!("Matrix 1: {:?}", mat1.data);
    println!("Matrix 2: {:?}", mat2.data);
    println!("Sum: {:?}", sum.data);

    // Access an element
    println!("Element at [1][1] in sum: {}", sum.data[1][1]); // 12.0
}
```

You can run the code on [Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=b10f24346ec81703b7d56f75f7b28d06).

---

### [bool::then](https://doc.rust-lang.org/std/primitive.bool.html#method.then): Lazy Option Creation

bool::then can be used as an alternative to if-else when building Option<T> conditionally.

It’s also lazy, so the closure only runs if true, skipping all work (allocations, I/O, heavy calculations) when false.

Instead of writing
```rust
fn main() {
    let verbose_a = true;

    let log_prefix_a: Option<String> = if verbose_a {
        Some("[DEBUG]".to_string())
    } else {
        None
    };

    println!("if-else result: {:?}", log_prefix_a); // Some("[DEBUG]")
```

You write
```rust
    let verbose_b = true;

    let log_prefix_b: Option<String> = verbose_b.then(|| {
        "[DEBUG]".to_string() // Closure runs ONLY if true – zero cost otherwise
    });

    println!(".then result:  {:?}", log_prefix_b); // Some("[DEBUG]")
}
```

You can play around with the code on [Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&gist=0c61bb0c5052229eb3d0155790ad76a0).
