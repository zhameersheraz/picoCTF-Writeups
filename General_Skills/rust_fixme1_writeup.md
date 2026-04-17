# Rust fixme 1 — picoCTF Writeup

**Challenge:** Rust fixme 1  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{4r3_y0u_4_ru$t4c30n_n0w?}`

---

## Description

> Have you ever heard of Rust? Fix the syntax errors in this Rust file to print the flag!
> Download the Rust code here.

**Hint shown in challenge:** `Cargo is Rust's package manager and will make your life easier.`

---

## Background Knowledge (Read This First!)

### What is Rust?

**Rust** is a systems programming language known for memory safety and performance. Key syntax rules:
- Every statement ends with `;`
- Use `return` to exit a function early
- Use `{}` in `println!` to print variables

### What is Cargo?

**Cargo** is Rust's built-in package manager and build tool. Two commands used here:
- `cargo build` — compiles the code and shows errors
- `cargo run` — builds and runs the program

---

## Solution — Step by Step

### Step 1 — Download and Extract the File

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ tar -xvzf fixme1.tar.gz
└─$ cd fixme1
```

### Step 2 — Build to See Errors

```
┌──(zham㉿kali)-[/media/sf_downloads/fixme1]
└─$ cargo build
```

**Errors found:**
```
error: expected `;`, found keyword `let`
 --> src/main.rs:5:37  (missing semicolon)

error: argument never used
  --> src/main.rs:26:9  (wrong format specifier ":?")

error[E0425]: cannot find value `ret` in this scope
  --> src/main.rs:18:9  (wrong return keyword)
```

✅ Found **3 syntax errors** to fix!

### Step 3 — Fix the Errors

```
┌──(zham㉿kali)-[/media/sf_downloads/fixme1]
└─$ nano src/main.rs
```

**Fix 1 — Missing semicolon:**
```rust
// BEFORE
let key = String::from("CSUCKS")

// AFTER
let key = String::from("CSUCKS");
```

**Fix 2 — Wrong return keyword:**
```rust
// BEFORE
ret;

// AFTER
return;
```

**Fix 3 — Wrong format specifier:**
```rust
// BEFORE
println!(":?", String::from_utf8_lossy(&decrypted_buffer));

// AFTER
println!("{}", String::from_utf8_lossy(&decrypted_buffer));
```

### Step 4 — Build and Run

```
┌──(zham㉿kali)-[/media/sf_downloads/fixme1]
└─$ cargo build
└─$ cargo run
picoCTF{4r3_y0u_4_ru$t4c30n_n0w?}
```

Got the flag! 🎯

---

## The 3 Errors Explained

| Error | Broken Code | Fixed Code | Reason |
|-------|-------------|------------|--------|
| Missing semicolon | `String::from("CSUCKS")` | `String::from("CSUCKS");` | Every statement in Rust must end with `;` |
| Wrong return keyword | `ret;` | `return;` | The correct Rust keyword is `return`, not `ret` |
| Wrong format specifier | `":?"` | `"{}"` | In `println!`, use `{}` to print a variable |

---

## Tools Used

| Tool | Purpose |
|------|---------|
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
