# Rust fixme 2 - picoCTF Writeup

**Challenge:** Rust fixme 2  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{4r3_y0u_h4v1n5_fun_y31?}`

---

## Description

The Rust saga continues? I ask you, can I borrow that, pleeeeeaaaasseeeee?

Download the Rust code here.

---

## Hints

1. https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html

---

## Solution

### Step 1: Download and Extract the File

```bash
wget https://challenge-files.picoctf.net/.../fixme2.tar.gz
tar -xvzf fixme2.tar.gz
cd fixme2
```

### Step 2: Try to Build and Read the Errors

```bash
cargo build
```

**Errors found:**
```
error[E0596]: cannot borrow `*borrowed_string` as mutable, as it is behind a `&` reference
 --> src/main.rs:9:5
  |
3 | fn decrypt(encrypted_buffer:Vec<u8>, borrowed_string: &String){
  |                                                        ^ help: consider changing this to be a mutable reference: `&mut String`

error[E0596]: cannot borrow `*borrowed_string` as mutable, as it is behind a `&` reference
  --> src/main.rs:20:5
```

✅ Found **2 errors** — both about mutable references!

### Step 3: Open and Fix the File

```bash
cd src
nano main.rs
```

**Fix 1: Change `&String` to `&mut String` in the function signature**

```rust
// BEFORE (broken)
fn decrypt(encrypted_buffer:Vec<u8>, borrowed_string: &String){

// AFTER (fixed)
fn decrypt(encrypted_buffer:Vec<u8>, borrowed_string: &mut String){
```

**Fix 2: Change `let party_foul` to `let mut party_foul` in main**

```rust
// BEFORE (broken)
let party_foul = String::from("Using memory unsafe languages is a: ");
decrypt(encrypted_buffer, &party_foul);

// AFTER (fixed)
let mut party_foul = String::from("Using memory unsafe languages is a: ");
decrypt(encrypted_buffer, &mut party_foul);
```

### Step 4: Build and Run

```bash
cargo build
cargo run
```

**Output:**
```
Using memory unsafe languages is a: PARTY FOUL! Here is your flag: picoCTF{4r3_y0u_h4v1n5_fun_y31?}
```

Got the flag! 🎯

---

## Why This Works

### What is Mutable Borrowing in Rust?

In Rust, when you pass a variable to a function, you can pass it in two ways:

| Type | Syntax | Can modify? |
|------|--------|-------------|
| Immutable reference | `&String` | No |
| Mutable reference | `&mut String` | Yes |

The original code used `&String` (immutable) but the function was trying to **modify** the string using `push_str()`. This is not allowed in Rust. The fix was to change it to `&mut String` so the function has permission to modify it.

Also, in Rust, a variable must be declared with `mut` before it can be changed:

```rust
let party_foul = ...;      // cannot be changed
let mut party_foul = ...;  // can be changed
```

### Simple Breakdown

```
cargo build shows 2 errors about mutable references
        |
        v
Fix 1: &String -> &mut String in function signature
Fix 2: let -> let mut for party_foul variable
Fix 3: &party_foul -> &mut party_foul when calling function
        |
        v
cargo build succeeds
        |
        v
cargo run prints the flag!
```

---

## Commands Used

```bash
# Download and extract
wget https://challenge-files.picoctf.net/.../fixme2.tar.gz
tar -xvzf fixme2.tar.gz
cd fixme2

# Build to see errors
cargo build

# Fix the file
cd src
nano main.rs

# Build and run after fixing
cargo build
cargo run
```

---

## Flag

```
picoCTF{4r3_y0u_h4v1n5_fun_y31?}
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `wget` | Download the tar file |
| `tar` | Extract the tar file |
| `cargo build` | Compile and show errors |
| `cargo run` | Run the fixed program |
| `nano` | Edit and fix the source code |

---

## Key Takeaways

- In Rust, variables are **immutable by default** — you must add `mut` to make them changeable
- To pass a variable to a function so it can be modified, use `&mut` not just `&`
- Rust's compiler gives very helpful error messages that tell you exactly what to fix
- `cargo build` is your best friend — always run it first to see all errors
