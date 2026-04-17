# Rust fixme 2 — picoCTF Writeup

**Challenge:** Rust fixme 2  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{4r3_y0u_h4v1n5_fun_y31?}`

---

## Description

> The Rust saga continues? I ask you, can I borrow that, pleeeeeaaaasseeeee?
> Download the Rust code here.

**Hint shown in challenge:** `https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html`

---

## Background Knowledge (Read This First!)

### What is Mutable Borrowing in Rust?

In Rust, when you pass a variable to a function, you can pass it two ways:

| Type | Syntax | Can modify? |
|------|--------|-------------|
| Immutable reference | `&String` | No |
| Mutable reference | `&mut String` | Yes |

Also, variables must be declared with `mut` before they can be changed:
```rust
let x = ...;      // cannot be changed
let mut x = ...;  // can be changed
```

---

## Solution — Step by Step

### Step 1 — Download and Extract the File

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ tar -xvzf fixme2.tar.gz
└─$ cd fixme2
```

### Step 2 — Build to See Errors

```
┌──(zham㉿kali)-[/media/sf_downloads/fixme2]
└─$ cargo build
```

**Errors found:**
```
error[E0596]: cannot borrow `*borrowed_string` as mutable, as it is behind a `&` reference
 --> src/main.rs:9:5
  | fn decrypt(..., borrowed_string: &String)
  | help: consider changing this to be a mutable reference: `&mut String`
```

✅ Found **2 errors** — both about mutable references!

### Step 3 — Fix the Errors

```
┌──(zham㉿kali)-[/media/sf_downloads/fixme2]
└─$ nano src/main.rs
```

**Fix 1 — Change `&String` to `&mut String` in the function signature:**
```rust
// BEFORE
fn decrypt(encrypted_buffer:Vec<u8>, borrowed_string: &String){

// AFTER
fn decrypt(encrypted_buffer:Vec<u8>, borrowed_string: &mut String){
```

**Fix 2 — Change `let` to `let mut` and `&` to `&mut` when calling:**
```rust
// BEFORE
let party_foul = String::from("Using memory unsafe languages is a: ");
decrypt(encrypted_buffer, &party_foul);

// AFTER
let mut party_foul = String::from("Using memory unsafe languages is a: ");
decrypt(encrypted_buffer, &mut party_foul);
```

### Step 4 — Build and Run

```
┌──(zham㉿kali)-[/media/sf_downloads/fixme2]
└─$ cargo build
└─$ cargo run
Using memory unsafe languages is a: PARTY FOUL! Here is your flag: picoCTF{4r3_y0u_h4v1n5_fun_y31?}
```

Got the flag! 🎯

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `tar` | Extract the tar file |
| `cargo build` | Compile and show errors |
| `cargo run` | Run the fixed program |
| `nano` | Edit and fix the source code |

---

## Key Takeaways

- In Rust, variables are **immutable by default** — add `mut` to make them changeable
- To pass a variable to a function so it can be modified, use `&mut` not just `&`
- Rust's compiler gives very helpful error messages that tell you exactly what to fix
- `cargo build` is your best friend — always run it first to see all errors
