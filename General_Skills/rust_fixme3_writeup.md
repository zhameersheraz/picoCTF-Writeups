# Rust fixme 3 - picoCTF Writeup

**Challenge:** Rust fixme 3  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{n0w_y0uv3_f1x3d_1h3m_411}`

---

## Description

Have you heard of Rust? Fix the syntax errors in this Rust file to print the flag!

Download the Rust code here.

---

## Hints

1. Read the comments...darn it!

---

## Solution

### Step 1: Download and Extract the File

```bash
wget https://challenge-files.picoctf.net/.../fixme3.tar.gz
tar -xvzf fixme3.tar.gz
cd fixme3
```

### Step 2: Try to Build and Read the Error

```bash
cargo build
```

**Error found:**
```
error[E0133]: call to unsafe function `std::slice::from_raw_parts` is unsafe and requires unsafe function or block
  --> src/main.rs:31:31
   |
31 |     let decrypted_slice = std::slice::from_raw_parts(decrypted_ptr, decrypted_len);
   |                           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |                           call to unsafe function
```

✅ Found **1 error** — the unsafe code block was commented out!

### Step 3: Open and Fix the File

```bash
cd src
nano main.rs
```

Looking at the code, I noticed the `unsafe { }` block was commented out:

```rust
// BEFORE (broken) - unsafe block was commented out
// unsafe {
    let decrypted_buffer = xrc.decrypt_vec(encrypted_buffer);
    let decrypted_ptr = decrypted_buffer.as_ptr();
    let decrypted_len = decrypted_buffer.len();
    let decrypted_slice = std::slice::from_raw_parts(decrypted_ptr, decrypted_len);
    borrowed_string.push_str(&String::from_utf8_lossy(decrypted_slice));
// }
```

The fix was to **uncomment the `unsafe { }` block** so the unsafe function call is properly wrapped:

```rust
// AFTER (fixed) - unsafe block uncommented
unsafe {
    let decrypted_buffer = xrc.decrypt_vec(encrypted_buffer);
    let decrypted_ptr = decrypted_buffer.as_ptr();
    let decrypted_len = decrypted_buffer.len();
    let decrypted_slice = std::slice::from_raw_parts(decrypted_ptr, decrypted_len);
    borrowed_string.push_str(&String::from_utf8_lossy(decrypted_slice));
}
```

### Step 4: Build and Run

```bash
cargo build
cargo run
```

**Output:**
```
Using memory unsafe languages is a: PARTY FOUL! Here is your flag: picoCTF{n0w_y0uv3_f1x3d_1h3m_411}
```

Got the flag! 🎯

---

## Why This Works

### What is Unsafe Rust?

Rust is known for being **memory safe** by default. However, some operations like working with raw pointers require stepping outside of Rust's safety rules. To do this, you must wrap those operations inside an `unsafe { }` block.

This tells the Rust compiler:
- "I know this is dangerous"
- "I take responsibility for making sure it's safe"

Without the `unsafe { }` block, Rust refuses to compile the code.

```rust
// This is NOT allowed in safe Rust
let slice = std::slice::from_raw_parts(ptr, len);

// This IS allowed because it's inside an unsafe block
unsafe {
    let slice = std::slice::from_raw_parts(ptr, len);
}
```

### Simple Breakdown

```
cargo build shows 1 error about unsafe function
        |
        v
Read the code - unsafe block was commented out
        |
        v
Uncomment the "unsafe { }" block
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
wget https://challenge-files.picoctf.net/.../fixme3.tar.gz
tar -xvzf fixme3.tar.gz
cd fixme3

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
picoCTF{n0w_y0uv3_f1x3d_1h3m_411}
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

- In Rust, unsafe operations like raw pointer access must be wrapped in an `unsafe { }` block
- The hint "read the comments" was pointing directly at the commented out `unsafe { }` block
- Rust forces you to explicitly mark unsafe code so you know exactly where the dangerous parts are
- `std::slice::from_raw_parts` is an unsafe function because it dereferences a raw pointer
- Always run `cargo build` first to see all errors before fixing anything
