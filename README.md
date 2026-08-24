# AI Plugins

AI plugins I've written.

## Set Up Marketplace

To set up the marketplace, run the following within your AI shell (Claude Code / Codex CLI):

* Claude: `/plugin marketplace add sooniln/ai-plugins`
* Codex: `codex plugin marketplace add sooniln/ai-plugins`

## Plugins

* Claude: `/plugin install <plugin-name>@sooniln-plugins`
* Codex: `codex plugin install <plugin-name>@ai-plugins`

* **jit-asm-capture**: Capture the JIT-compiled (C2) machine code of a method in any HotSpot JVM. Use to examine
  generated assembly and inlining behavior, understand auto-vectorization, scalar-replacement, and performance behavior,
  or to compare assembly and performance between multiple implementations.
* **pseudo-c**: Represent assembly instructions as pseudo-C for easier readability.
