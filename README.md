# Tips From [Rust Bytes Newsletter](https://weeklyrust.substack.com/)

### ManuallyDrop + ptr::write for move out without drop patterns
Want to move a value out of a struct without dropping the rest?

Combine ManuallyDrop with ptr::read/write:
```rust
let mut thing = ManuallyDrop::new(my_struct);
let field = ptr::read(&thing.field); // moves out
// now you can drop the rest manually or forget
```
I use this constantly when implementing arenas, intrusive linked lists, or custom Box-like types.

---

### std::ptr::addr_of! and addr_of_mut!: your new best friends for field projection
When you need a raw pointer to a field without going through a reference to avoid stacked borrows or temporary references, use:
```rust
let ptr = std::ptr::addr_of!((*some_ptr).field);
let mut_ptr = std::ptr::addr_of_mut!((*some_ptr).field);
```
This is UB-free in cases where &(*some_ptr).field would create illegal intermediate references (very common in unsafe FFI, kernel, or intrusive data structures).

---

### #[track_caller] + std::panic::Location for god-tier debugging in libraries
Want your library’s error messages/panics/logs to point to the user’s code, not your internal helper?
```rust
#[track_caller]
pub fn my_library_function() {
    let location = std::panic::Location::caller();
    println!("called from {}:{}", location.file(), location.line());
}
```
I put this on every public-facing helper that could fail. Makes debugging 10× nicer for downstream users.

---

### Export commonly-used items in a prelude module
If your crate has items that most users will need, create a prelude module to save them repetitive imports:
```rust
pub mod prelude {
    pub use crate::RoastBeef;
    pub use crate::carbs::pasta::filled::Tortelloni;
    pub use crate::snacks::Popcorn;
}
```
Now users can get your whole delicious menu delivered with just one call:

---

### Embed static data with include_str!
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
