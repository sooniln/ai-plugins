---
name: pseudo-c
description: Represent assembly instructions as pseudo-C for easier readability. Use when the user asks to convert, translate, or represent assembly as C-like/pseudo-C code.
license: MIT
metadata:
  author: Soonil Nagarkar
  version: 1.0.0
---

# Pseudo-C Representation

The user may ask you to represent assembly in a format close to C for easier readability. In this case, follow these
rules:

1. Use std types (int32_t, uint32_t, etc.).
2. If the assembly contains JVM uncommon traps or safepoint polls, represent these as uncommon_trap()/safepoint_poll()
   functions. JVM heap allocations should be represented as jvm_alloc() (represents TLAB bump + fallback).
3. Local variables must correspond to register usage.
4. Prefer to avoid goto usage unless there is no other option to accurately represent the real assembly.
5. Use idiomatic constructs (++i, etc.) where applicable.
6. The structure of the pseudo-C should attempt to mirror the structure of the assembly.
7. Double check generated pseudo-C to ensure you have made no mistakes, no inaccurate comments, no inaccurate
   representations of the assembly, and all rules listed here are obeyed.
