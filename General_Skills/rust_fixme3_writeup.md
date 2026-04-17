# Rust fixme 3 — picoCTF Writeup

**Challenge:** Rust fixme 3  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{n0w_y0uv3_f1x3d_1h3m_411}`

---

## Description

> Have you heard of Rust? Fix the syntax errors in this Rust file to print the flag!
> Download the Rust code here.

**Hint shown in challenge:** `Read the comments...darn it!`

---

## Background Knowledge (Read This First!)

### What is Unsafe Rust?

Rust is known for being **memory safe** by default. However, some operations like working with raw pointers require stepping outside Rust's safety rules. To do this, you must wrap those operations inside an `unsafe { }` block.

```rust
// NOT allowed in safe Rust
let slice = std::slice::from_raw_parts(ptr, len);

// ALLOWED inside unsafe block
unsafe {
    let slice = std::slice::from_raw_parts(ptr, len);
}
```

Without the `unsafe { }` block, Rust refuses to compile.

---

## Solution — Step by Step

### Step 1 — Download and Extract the File

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ tar -xvzf fixme3.tar.gz
└─$ cd fixme3
```

### Step 2 — Build to See the Error

```
┌──(zham㉿kali)-[/media/sf_downloads/fixme3]
└─$ cargo build
```

**Error found:**
```
error[E0133]: call to unsafe function `std::slice::from_raw_parts` is unsafe
  --> src/main.rs:31:31
  | call to unsafe function
```

✅ Found **1 error** — the `unsafe { }` block was commented out!

### Step 3 — Fix the File

```
┌──(zham㉿kali)-[/media/sf_downloads/fixme3]
└─$ nano src/main.rs
```

The `unsafe { }` block was commented out. I uncommented it:

```rust
// BEFORE (broken)
// unsafe {
    let decrypted_slice = std::slice::from_raw_parts(decrypted_ptr, decrypted_len);
    borrowed_string.push_str(&String::from_utf8_lossy(decrypted_slice));
// }

// AFTER (fixed)
unsafe {
    let decrypted_slice = std::slice::from_raw_parts(decrypted_ptr, decrypted_len);
    borrowed_string.push_str(&String::from_utf8_lossy(decrypted_slice));
}
```

### Step 4 — Build and Run

```
┌──(zham㉿kali)-[/media/sf_downloads/fixme3]
└─$ cargo build
└─$ cargo run
Here is your flag: picoCTF{n0w_y0uv3_f1x3d_1h3m_411}
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

- In Rust, unsafe operations like raw pointer access must be wrapped in `unsafe { }`
- The hint "read the comments" pointed directly at the commented-out `unsafe { }` block
- Rust forces you to explicitly mark unsafe code so you know exactly where the dangerous parts are
- `std::slice::from_raw_parts` is an unsafe function because it dereferences a raw pointer
