# BKCTF 2026 — supercool Writeup

**Category:** Reverse Engineering  
**Difficulty:** Easy  
**Flag:** `bkctf{sup3rv3ryt074llyc00l}`

---

## What is this challenge?

We get a binary file called `supercool`. No source code. No hints. Just a file.

This is the classic CTF "here is a program, figure out what it does" challenge. The instinct of most beginners here is to open a debugger or write a decompiler script.

The actual solution? Three commands.

---

## Step 1 — What even is this file?

Before doing anything, we check what type of file we are dealing with:

```bash
file supercool
```

Output:
```
supercool: ELF 64-bit LSB executable, x86-64, ...
```

It is an ELF binary — a Linux executable. Good to know. We are not going to run it blindly though.

---

## Step 2 — Look for readable text inside

Developers sometimes hardcode strings directly into their binaries. Passwords, messages, flags — all sitting there in plain text, waiting to be found.

The `strings` tool pulls out every readable ASCII string from a binary:

```bash
strings supercool
```

---

## Step 3 — Filter with grep

Instead of reading thousands of lines manually, we pipe the output into `grep` and search for the flag format:

```bash
strings supercool | grep "bkctf"
```

Output:
```
bkctf{sup3rv3ryt074llyc00l}
```

Done.

---

## Why did this work?

The flag was **hardcoded** inside the binary as a plain string. The developer stored it directly in the code probably to compare it against user input at runtime but never encrypted or obfuscated it.

When a string is hardcoded like this:

```c
if (strcmp(input, "bkctf{sup3rv3ryt074llyc00l}") == 0) {
    printf("Correct!\n");
}
```

It gets compiled into the binary as raw ASCII bytes. `strings` reads those bytes directly from the file without even executing it.

---

## Lesson

> Not every reverse engineering challenge requires a debugger.  
> Always check the obvious before going deep.

`strings` is the first tool you should reach for on any unknown binary. Lazy developers leave flags, passwords, and API keys sitting in plain text all the time — not just in CTFs but in real software too.

The pipeline `strings <file> | grep <pattern>` has found more secrets than most people expect.

---

*Written for BKCTF 2026*
