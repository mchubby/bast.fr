+++
title = "Holy Rust modules, Batman! - How I structure Rust projects"
+++

![Batman thinking](/blog/batman.jpg)

## Modules and directory structure in a Rust project

First a disclaimer: I called this post "How I structure Rust projects" and not
"How **to** structure Rust projects" since I am still a Rust novice and there
is likely a better and more canonical way. And although I really enjoy the
language and [The Rust Programming Language](https://doc.rust-lang.org/book/)
book, the part with structuring the module tree and directory structure was not
very clear to me when I started. I was unsure about how to arrange my files and
modules and after some trial and error arrived at what I am using right now in
my projects and what I will present here. Hopefully it will be useful for
others. Please review this critically and send improvements!


## Example: A simple project called "gotham"

To have a tangible but hopefully simple example, we will discuss a fantasy
project called `gotham`. The project has the following folder structure:

```
gotham
 ├── Cargo.lock
 ├── Cargo.toml
 ├── README.md
 ├── src
 │    ├── batman.rs
 │    ├── joker.rs
 │    ├── lib.rs
 │    ├── main.rs
 │    ├── riddler.rs
 │    └── robin.rs
 └── tests
      └── integration_test.rs
```

All the sources are under `src`, and the integration tests are under `tests`.
You can find the full example here: <https://github.com/bast/gotham>.

First let's see what the project can do and then we will discuss how the
modules are arranged and how the imports work.  The project contains a main
function and 3 library functions.  We can run the main code:

```
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.02s
     Running `target/debug/gotham`

Na na na na na na na na Batman!
Batman says: Robin, you haven't fastened your safety bat-belt.
Riddler says: Riddle me this, Batman: what are the chilliest 12 inches in the world?
```

And let us also test the 3 available library functions (`batman_quote`,
`robin_exclamation`, and `joker_warning`):

```rust
println!("{}", gotham::batman_quote());
// Robin, you haven't fastened your safety bat-belt.

println!("{}", gotham::robin_exclamation());
// Holy Bat Logic!

println!("{}", gotham::joker_warning());
// And now people of Gotham City, the moment you have all been waiting for.
```


## Arranging the library crate

The library functions are distributed over a couple of files: `batman.rs`,
`joker.rs`, `riddler.rs`, and `robin.rs`. In this simplified example these
files are short but in a real project we can imagine that each of these files
will collect a number of related functions.

First let us discuss `src/lib.rs` which describes the library crate. This file
lists all the files/modules, as well as all 3 public library functions
explicitly.

**src/lib.rs**:
```rust
// list here all files/modules
mod batman;
mod joker;
mod riddler;
mod robin;

// list here all functions that form the
// interface of the library
pub use crate::batman::batman_quote;
pub use crate::joker::joker_warning;
pub use crate::robin::robin_exclamation;
```

We will be able to call `batman_quote`, `joker_warning`, and
`robin_exclamation` from outside the library. But we will not be able to call
any other function defined in any of the source files. For instance, none
of the functions in `riddler.rs` are exported outside the library.


## Relative imports

Let us first have a closer look at the source code belonging to the dynamic duo
(Batman and Robin) and two villains (Joker and Riddler) and then we will point
out few interesting things about the visibility of functions.

**src/batman.rs**:
```rust
use crate::riddler;

fn my_real_name() -> String {
    "Bruce Wayne".to_string()
}

pub fn batman_quote() -> String {
    // nobody will know
    let _my_secret_identity = my_real_name();

    // ask riddler for a riddle
    let _riddle = riddler::difficult_riddle();

    "Robin, you haven't fastened your safety bat-belt.".to_string()
}

pub fn advice() -> String {
    "You've made a hasty generalization, Robin. It's a bad habit to get into.".to_string()
}
```

**src/robin.rs**:
```rust
use crate::batman;

fn _just_a_thought() -> String {
    "Gosh, if I could just figure out that riddle. Why can't I get it?".to_string()
}

pub fn robin_exclamation() -> String {
    let _batmans_advice = batman::advice();

    "Holy Bat Logic!".to_string()
}
```

**src/joker.rs**:
```rust
pub fn joker_warning() -> String {
    "And now people of Gotham City, the moment you have all been waiting for.".to_string()
}
```

**src/riddler.rs**:
```rust
pub fn difficult_riddle() -> String {
    "Riddle me this, Batman: what are the chilliest 12 inches in the world?".to_string()
}

fn _another_riddle() -> String {
    "How many sides has a circle?".to_string()
}
```

A couple of observations:

- Only public functions (with a `pub` classifier) can be accessed outside the
  file where they are defined (`batman_quote`, `advice`, `joker_warning`,
  `difficult_riddle`, and `robin_exclamation`).
- Private functions (no `pub`) cannot be accessed outside the file (e.g.
  `my_real_name`).
- If we want to use a public function in another file (e.g. `difficult_riddle`
  in `src/batman.rs`), we have to "use" the corresponding file (`use
  crate::riddler`) and with this `riddler::difficult_riddle()` becomes
  visible.
- Some private function names start with an underscore (e.g.
  `_batmans_advice`): I did this only to silence `rustc` warnings about unused
  functions. In a real project this is not needed since hopefully all functions
  are used.


## Arranging the binary crate

This is the content of
**src/main.rs**:
```rust
mod riddler;

fn main() {
    println!("Na na na na na na na na Batman!");

    // using a library function
    let quote = gotham::batman_quote();
    println!("Batman says: {}", quote);

    // this function is not part of the library
    let quote = riddler::difficult_riddle();
    println!("Riddler says: {}", quote);
}
```

We brought two functions in:

- `gotham::batman_quote()`: this is a library function of the `gotham` library
  crate, exported in `src/lib.rs`.
- `riddler::difficult_riddle()`: this function is not part of the library
  interface and we brought it into scope with the help of `mod riddler`.
  We can use it here but we cannot use the function outside the library crate.

We could remove `src/main.rs` and we would end up with only a library crate and
the integration tests (below) would still work.


## Testing the library functions

In `tests/integration_test.rs` we test the 3 library functions. Note how we call
them with the `gotham::` namespace.

**tests/integration_test.rs**:
```rust
#[test]
fn batman() {
    let quote = gotham::batman_quote();
    assert_eq!(quote, "Robin, you haven't fastened your safety bat-belt.");
}

#[test]
fn joker() {
    let quote = gotham::joker_warning();
    assert_eq!(
        quote,
        "And now people of Gotham City, the moment you have all been waiting for."
    );
}

#[test]
fn robin() {
    let quote = gotham::robin_exclamation();
    assert_eq!(quote, "Holy Bat Logic!");
}
```


## Conclusion

- Each source file (except `main.rs` and `lib.rs`) is listed in `src/lib.rs`.
- Many projects have no `main.rs`.
- Each function which is supposed to be visible outside the library is
  explicitly listed in `src/lib.rs`.
- I like to test the library functions in `tests/integration_test.rs`: not only
  to test the functions themselves but also that their export is working.

You can find the full example here: <https://github.com/bast/gotham>.
Suggestions and improvements are most welcome!
