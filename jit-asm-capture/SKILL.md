---
name: jit-asm-capture
description: Capture the JIT-compiled (C2) machine code of a method in any HotSpot JVM. Use to examine generated assembly, bounds checking elimination, and inlining behavior, understand auto-vectorization, scalar-replacement, and performance behavior, or to compare assembly and performance between multiple implementations.
license: MIT
metadata:
  author: Soonil Nagarkar
  version: 1.0.0
---

# JIT assembly capture

Captures the actual C2-compiled machine code of a single method under test by building a small throwaway Java harness,
warming it up just enough to force compilation, and dumping disassembly with `-XX:+PrintAssembly`. This captures exactly
one method per run. There's no built-in comparison mode -- if you want to look at two or more implementations side by
side, run the capture for each one.

## Instructions

### Step 1 - Determine the Java SDK to use

If the user has specified a particular Java version/SDK to use, obey that. Otherwise, if Java is
already present on the path, use that. If not, search common Java install locations, and prefer
the latest version available:

- **Windows**: `%JAVA_HOME%`, `~/.jdks/*` (IntelliJ-managed JDKs), `C:\Program Files\Java\*`,
- **macOS**: `$JAVA_HOME`, `~/.jdks/*`, output of `/usr/libexec/java_home -V`
- **Linux**: `$JAVA_HOME`, `~/.jdks/*`, `/usr/lib/jvm/*`

### Step 2 - Determine the classpath to use

Make sure the code under test is built (so that its jar/class files are up-to-date and available), and find its
classpath. How you get there is project-specific -- Maven, Gradle, plain `javac`, whatever the target project uses. If
the method you're inspecting is self-contained (no dependency on project code), you can skip this entirely.

- **Gradle**: generally build with:
    - Java: `./gradlew <module>:classes`
    - Kotlin: `./gradlew <module>:compileKotlin`
    - Kotlin Multiplatform: `./gradlew <module>:jvmMainClasses`
    - Output: compiled classes should land under `build/classes/`. Gradle has no single built-in command that prints a
      flat runtime classpath; the common way is `./gradlew <module>:dependencies --configuration runtimeClasspath` to
      see the resolved artifact list, then locate each jar via
      `find ~/.gradle/caches -iname "<artifact>-<version>*.jar"` once the version is known from the dependency list or
      `build.gradle(.kts)`/version catalog.
- **Maven**: generally build with `mvn compile`.
    - Output: compiled classes should land under `target/classes`. Get the full dependency classpath with
      `mvn dependency:build-classpath`.
- **Plain `javac`/no build tool**: there's no dependency resolution to do -- check the project's own run/build script (
  Makefile, shell script, README) for the `-cp`/`-classpath` value or `CLASSPATH` env var it uses, and reuse that.
- **Other build tools** (Bazel, Ant, sbt, ...): not covered in depth -- look for that tool's own
  classpath/dependency-listing facility.

### Step 3 - Construct a harness to run the method under test

Use `assets/Harness.java.template` as a base to construct the harness Java file (you may be able to more quickly
re-use/edit a pre-existing harness from a prior run as well). Prefer to locate this within your scratchpad or at a
location for temporary files, which allows for later re-use. Add whatever imports may be required.

### Step 4 - Compile the harness

Use the Java SDK and classpath as determined in steps 1 and 2, compile with `javac`, and ensure compilation is
error-free.

**Windows gotcha -- don't mix POSIX/Windows path forms in the same `-cp`/`-classpath` string.** Fix: convert every
classpath component to the same Windows form before joining with `;`, including the harness's own output directory.

### Step 5 - Run the harness

Run the harness with the following command line flags, and capture the output for examination. Assume 
hsdis is present until proved otherwise (no pre-emptive checking).

- `-XX:+UnlockDiagnosticVMOptions` unlocks diagnostic-only VM flags, required to use `-XX:+PrintAssembly` and the other
  diagnostic flags below.
- `-XX:+PrintAssembly` forces java to output the assembly from compilation.
- `-XX:PrintAssemblyOptions=intel` prints the assembly in Intel syntax (instead of the default ATT syntax).
- `-Xbatch` makes compilation synchronous, removing the need to guess a "settling" buffer of extra reps after crossing
  the threshold.
- `-XX:CompileCommand=quiet` hides compilation information not explicitly requested - see following option.
- `-XX:CompileCommand=compileonly,package.path.class::methodUnderTest` restricts which method gets its *own* nmethod,
  but C2 can still inline anything reachable from it regardless of whether the callee is separately compiled -- so
  restricting to just the wrapper method still yields one fully-inlined nmethod covering the whole call chain, which is
  exactly what you want to read.
- `-XX:CompileCommand=print,package.path.class::methodUnderTest` forces the compiled assembly of the given method to be
  output.
- `-XX:+PrintCompilation` logs each compilation event as it happens, including OSR (`%`) vs. steady-state entry
  compiles -- needed to tell the two apart when reading the capture (see Step 6).
- `-XX:+PrintInlining` prints the full inlining tree for the compiled method, annotating each call site as
  `inline (hot)`, `too large`, `not inlined`, etc.

### Step 6 - Examine the output

It's possible that the hsdis library is missing, which shows up as a `warning: Loading hsdis library failed` line and
the assembly mnemonics sections falling back to raw hex (`[MachCode]`) instead of real mnemonics under `[Disassembly]`.
If this is the case, evaluate whether the output is currently sufficient to answer the user's prompt appropriately:

- Whether a method **reaches C2 at all**, its **OSR vs. steady-state timing**, and its **inlining tree** (
  `inline (hot)` / `too large` / `not inlined`) all come from `-XX:+PrintCompilation` and `-XX:+PrintInlining`, none of
  which needs hsdis.
- **Vectorization** (AVX/`ymm`/`zmm` usage), **allocation elimination / escape analysis**, and anything else that
  depends on reading the literal instruction stream generally requires hsdis.

If there is enough information to answer the user's prompt, do so, but also note that hsdis is missing. If there is not
enough information to answer the user's prompt, and real assembly mnemonics are required, follow the hsdis missing
instructions below and then rerun and continue from step 5.

Some other notes on output:

- **OSR vs. steady state**: a method can appear compiled twice -- once as an OSR compile
  (`Compiled method (c2) ... %  ... YourMethod`, triggered mid-loop on an early call) and once as
  a normal entry compile (no `%`, triggered once the invocation counter crosses threshold). The
  OSR version compiles earlier with less profiling data and may be noticeably less optimized (e.g.
  not vectorized) than the later steady-state one. **Read the later, non-OSR compile** for a fair
  "fully warmed up" picture -- both appear in the same capture file, later one wins.
- **Vectorization**: grep the capture file for `ymm` (AVX2, 256-bit) or `zmm` (AVX-512). Their
  presence/absence and the surrounding unroll factor tell you whether C2's superword optimizer
  kicked in, and how many elements it processes per unrolled iteration.
- **Allocation / escape analysis**: if the method allocates an object (iterator, lambda, boxed
  wrapper) that never escapes, C2 often scalar-replaces it entirely -- the compiled code just reads
  primitive fields with no `new`/allocation runtime call at all. Its *absence* in the disassembly is
  the evidence it worked; if you instead see a call into an allocation stub, escape analysis failed
  and that's a real, reportable finding.
- **`-XX:+PrintInlining`** (already included in the flags above) shows the full inline tree with
  `inline (hot)` / `too large` / `not inlined` annotations -- when reading two capture files
  together, use it to confirm both got inlined to comparable depth before concluding a difference
  in the assembly reflects the source change rather than an inlining-budget artifact.
- **"made not entrant" / "uncommon trap"** lines mean a compile was invalidated (deopt). Treat any
  compile before the last one for that method with suspicion; only the final surviving compile is
  representative.

#### If hsdis is missing

If the prompt genuinely needs real assembly mnemonics, check if there is a copy of hsdis in the current project or on
the current working path. If not, tell the user the hsdis disassembler library isn't present/working and ask how they'd
like to proceed:

- **They point to a copy of hsdis.**
- **You download a prebuilt copy from an online source on their behalf.** If they choose this, tell them plainly first:
  this means fetching and loading a native shared library you haven't built or reviewed directly into the JVM process's
  address space -- a malicious version could execute arbitrary code in that process. Only do this with their explicit
  permission, prefer a source they'd recognize as reputable, and say where you got it from.

Once you know where the copy is located, verify that it is correct for the JVM architecture -- `hsdis-<arch>.dll` on
Windows, `hsdis-<arch>.so` on Linux, `hsdis-<arch>.dylib` on MacOS. If the source publishes a checksum for the file,
verify it programmatically (e.g. `Get-FileHash`/`sha256sum`) before use. Then install the library under the relevant
JDK installation:

- **Linux/macOS**: `<JDK_HOME>/lib/server/` or `<JDK_HOME>/lib/`.
- **Windows**: `<JDK_HOME>\bin\server\` or `<JDK_HOME>\bin\`.

This may require administrator privileges (confirm, don't assume), but should be the preferred option as it means the
library will always be available in the future. If unable to obtain administrator privileges, you can also copy the
library into the same directory as the harness, where `java` should be able to find it.
