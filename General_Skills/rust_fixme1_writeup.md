# Rust fixme 1 - picoCTF Writeup

**Challenge:** Rust fixme 1  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{4r3_y0u_4_ru$t4c30n_n0w?}`

---

## Description

Have you heard of Rust? Fix the syntax errors in this Rust file to print the flag!

Download the Rust code here.

---

## Hints

1. Cargo is Rust's package manager and will make your life easier. See the getting started page here.

---

## Solution

### Step 1: Download and Extract the File

I downloaded the tar file and extracted it:

```bash
wget https://challenge-files.picoctf.net/.../fixme1.tar.gz
tar -xvzf fixme1.tar.gz
cd fixme1
```

This extracted:
```
fixme1/
fixme1/Cargo.toml
fixme1/Cargo.lock
fixme1/src/
fixme1/src/main.rs
```

### Step 2: Try to Build and Read the Errors

I ran `cargo build` to see what errors existed:

```bash
cargo build
```

**Errors found:**
```
error: expected `;`, found keyword `let`
 --> src/main.rs:5:37
  |
5 |     let key = String::from("CSUCKS") // missing semicolon

error: argument never used
  --> src/main.rs:26:9
  | ":?" // wrong format specifier

error[E0425]: cannot find value `ret` in this scope
  --> src/main.rs:18:9
  | ret; // wrong return keyword
```

✅ Found **3 syntax errors** to fix!

### Step 3: Open and Fix the File

```bash
cd src
nano main.rs
```

**Fix 1: Missing semicolon at the end of the `key` statement**

```rust
// BEFORE (broken)
let key = String::from("CSUCKS")

// AFTER (fixed)
let key = String::from("CSUCKS");
```

**Fix 2: Wrong return keyword (`ret` instead of `return`)**

```rust
// BEFORE (broken)
ret;

// AFTER (fixed)
return;
```

**Fix 3: Wrong println format specifier (`:?` instead of `{}`)**

```rust
// BEFORE (broken)
println!(
    ":?",
    String::from_utf8_lossy(&decrypted_buffer)
);

// AFTER (fixed)
println!(
    "{}",
    String::from_utf8_lossy(&decrypted_buffer)
);
```

### Step 4: Build and Run

```bash
cargo build
cargo run
```

**Output:**
```
picoCTF{4r3_y0u_4_ru$t4c30n_n0w?}
```

Got the flag! 🎯

---

## Why This Works

### The 3 Syntax Errors Explained

| Error | Broken Code | Fixed Code | Reason |
|-------|-------------|------------|--------|
| Missing semicolon | `String::from("CSUCKS")` | `String::from("CSUCKS");` | In Rust, every statement must end with `;` |
| Wrong return keyword | `ret;` | `return;` | The correct keyword in Rust is `return`, not `ret` |
| Wrong format specifier | `":?"` | `"{}"` | In Rust's `println!`, use `{}` to print a variable |

### What is Rust?

**Rust** is a systems programming language known for memory safety and performance. Some key syntax rules:
- Every statement ends with `;`
- Use `return` to exit a function early
- Use `{}` in `println!` to print variables

### Simple Breakdown

```
Download file
      |
      v
cargo build shows 3 errors
      |
      v
Fix: add ";" after key statement
Fix: change "ret" to "return"
Fix: change ":?" to "{}"
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
wget https://challenge-files.picoctf.net/.../fixme1.tar.gz
tar -xvzf fixme1.tar.gz
cd fixme1

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
picoCTF{4r3_y0u_4_ru$t4c30n_n0w?}
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `wget` | Download the tar file |
| `tar` | Extract the tar file |
| `cargo build` | Compile and show syntax errors |
| `cargo run` | Run the fixed program |
| `nano` | Edit and fix the source code |

---

## Key Takeaways

- Always run `cargo build` first to see all errors before fixing anything
- In Rust, every statement must end with a semicolon `;`
- The correct keyword to return early in Rust is `return`, not `ret`
- In `println!`, use `{}` to print a variable's value
- `cargo` makes it very easy to build and run Rust projects
