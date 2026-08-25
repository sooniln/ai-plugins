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

1. The structure of the pseudo-C MUST be a 1:1 representation of the structure of the assembly. Do not elide or
   misrepresent assembly in order to make the higher level code more convenient.
2. Variables MUST correspond to register usage - do not invent variables that are not backed by a register.
3. You may represent low level JVM operations (like uncommon traps / safepoint polls / allocations) via functions calls
   such as uncommon_trap()/safepoint_poll()/jvm_alloc().
4. Use std types (int32_t, uint32_t, etc.).
5. Use idiomatic constructs (++i, min()/max() pseudo-functions, etc.) where applicable.
6. Use if/while/do/for statements where they accurately represent the assembly without a loss of 1:1 fidelity.
7. Double check generated pseudo-C to ensure you have made no mistakes, no inaccurate comments, no inaccurate
   representations of the assembly, and all rules listed here are obeyed.
