# CS2 Compilation & Decompilation Tutorial

A practical guide to working with **ClientScript 2 (CS2)** in this 228-based RSPS project. Covers what CS2 is, how we **compile** it with Neptune, how we **decompile** existing scripts pulled from the OSRS cache, and how it all wires into the cache build pipeline.

This document was written so it can be handed to another developer who has not seen the project before. File paths and line numbers are accurate as of the time of writing — verify against the current tree if you read this much later.

---

## 1. What is CS2?

**ClientScript 2** is the scripting language Jagex uses inside the OSRS cache to drive interface logic, button handlers, tooltips, dropdown population, hover effects, and basically anything dynamic the gameframe does on the client side. Scripts are stored in the cache as compiled bytecode in archive `CLIENTSCRIPTS` (archive index 12), one script per group.

A CS2 script:
- Has an integer ID (the cache file ID)
- Has a name (e.g. `[clientscript,tt_reward_selected_onclick]`)
- Takes typed `int`/`string` arguments
- Reads/writes varps, varbits, varcs
- Calls "commands" (engine ops like `cc_setgraphic`, `enum`, `oc_param`)
- Can `gosub` other scripts ("procs")

You will hear three names used interchangeably: **CS2**, **clientscript**, and **RuneScript** (RuneScript is the language family; CS2 is the OSRS dialect Neptune targets).

---

## 2. The Tools We Use

| Job | Tool | Source |
|-----|------|--------|
| **Compile** `.cs2` source → cache bytecode | Neptune (`clientscript-compiler`) | https://github.com/neptune-ps/neptune |
| **Decompile** cache bytecode → readable form | `org.runestar.cs2` (Gradle dep `org.runestar.cs2:cs2`) | https://github.com/runestar/cs2 |
| **Inspect** raw enums/structs | Custom Kotlin dumpers in this repo | `cache/src/main/kotlin/com/near_reality/cache_tool/` |
| **Pack** compiled scripts into the cache | `NearRealityCustomCS2Packer` + `mgi.tools.parser.TypeParser` | `cache/src/main/.../packing/custom/` |

Neptune is the official-ish open-source CS2 compiler maintained by the `neptune-ps` org. The runestar `cs2` library is a third-party decompiler that targets cache bytecode and produces a readable IR.

---

## 3. Where Everything Lives

All CS2 assets sit under:

```
cache/assets/osnr/
├── cs2_source/        ← .cs2 source files we author (input to Neptune)
├── cs2_library/       ← shared command/proc signature libraries (.cs2)
├── cs2_symbols/       ← *.sym files mapping IDs ↔ names for every type
├── custom_cs2/        ← compiled binaries Neptune produces (output)
├── neptune.toml       ← Neptune project config
└── leagues_cs2_decompiled/   ← output of our decompiler tool
```

Symbol files (`cs2_symbols/*.sym`) are the bridge between human-readable names and cache IDs. Neptune needs these so that when you write `if_setgraphic(component_657_42, sprite_id)` it knows what numeric ID to emit. There is one `.sym` per type:

```
category.sym  clientscript.sym  component.sym  enum.sym  graphic.sym
interface.sym inv.sym           loc.sym        npc.sym   obj.sym
param.sym     seq.sym           stat.sym       struct.sym synth.sym
varbit.sym    varc.sym          varp.sym       ...
```

Each `.sym` is a tab-separated `id<TAB>name` file. Example from `interface.sym`:

```
6       interface_6
12      interface_12
657     interface_657   ← (you would name interfaces meaningfully here)
```

---

## 4. Setting Up Neptune

### 4.1 Clone & Build

```bash
cd ~/Desktop
git clone https://github.com/neptune-ps/neptune Neptune
cd Neptune/Neptune     # the inner directory is the actual project
./gradlew :clientscript-compiler:installDist
```

This produces a runnable script at:

```
clientscript-compiler/build/install/cs2/bin/cs2.bat   (Windows)
clientscript-compiler/build/install/cs2/bin/cs2       (Linux/Mac)
```

The `compileCS2` Gradle task in this project (see §5) hard-codes that path.

### 4.2 The Neptune `script.name` Fix

There is an important detail to be aware of in `BinaryScriptWriterContext.kt`:

```
clientscript-compiler/src/main/kotlin/me/filby/neptune/clientscript/compiler/writer/BinaryScriptWriterContext.kt
```

Lines 73 and 88 must use `script.name`, **not** `script.fullName`:

```kotlin
// Line 73
buf.writeString(script.name)
...
// Line 88
size += script.name.length + 1
```

`fullName` includes the trigger prefix (`[clientscript,foo]`) which the OSRS client does not expect — using it causes the cache to load corrupted script names and breaks lookup. The version of Neptune you build must have this. If you cloned a fork that already had it patched, you are fine; double-check those two lines before running anything in production.

### 4.3 Java Version

Neptune (and this project) require **Java 17**. Anything newer will fail compilation. On Windows:

```bash
export JAVA_HOME="/c/Users/DevBox2/.jdks/ms-17.0.18"
```

---

## 5. Compiling CS2 in This Project

### 5.1 The Gradle Task

`cache/build.gradle.kts` defines:

```kotlin
tasks.register<Exec>("compileCS2") {
    group = "_nr_data"
    description = "Compile RuneScript CS2 source using Neptune"
    commandLine(
        "${System.getProperty("user.home")}/Desktop/Neptune/Neptune/clientscript-compiler/build/install/cs2/bin/cs2.bat",
        "--config-path",
        projectDir.resolve("assets/osnr/neptune.toml").absolutePath
    )
    workingDir = projectDir
}
```

It is just an `Exec` shelling out to the Neptune binary, pointed at our `neptune.toml`.

> If you are not on Windows, change `cs2.bat` to `cs2`.
> If your Neptune clone is somewhere other than `~/Desktop/Neptune/Neptune`, edit the path here.

### 5.2 `neptune.toml`

```toml
name = "task-tracker"
sources = ["cs2_source/", "cs2_library/"]
symbols = ["cs2_symbols/"]
libraries = ["cs2_library/"]

[writer.binary]
output = "custom_cs2/"
```

Neptune scans `sources` for `.cs2` files, resolves names against `symbols`, treats `libraries` as signature-only references (no codegen), and writes raw cache files to `output`. All paths are relative to the working dir Gradle sets (`cache/assets/osnr/`).

### 5.3 The Workflow

```bash
# 1. Edit or add a .cs2 file under cache/assets/osnr/cs2_source/
#    e.g. cache/assets/osnr/cs2_source/30100.cs2
#    The filename's number is the cache script ID it will become.

# 2. Compile
./gradlew :cache:compileCS2

# 3. Pack the new binaries into the cache and (re)build it
./gradlew :cache:generatecacheprod
```

`compileCS2` produces files in `cache/assets/osnr/custom_cs2/`, e.g. `30100.cs2`. These are **binary cache files**, despite the `.cs2` extension — they are not source.

`NearRealityCustomCS2Packer` then sweeps that folder and writes each one to the cache:

```kotlin
// cache/src/main/kotlin/com/near_reality/cache_tool/packing/custom/NearRealityCustomCS2Packer.kt
internal object NearRealityCustomCS2Packer {
    @JvmStatic
    fun pack() = assets("assets/osnr/custom_cs2/") {
        TypeParser.packClientScript(id, bytes)
    }
}
```

The `id` comes from the filename and `bytes` is the file contents — that's it.

### 5.4 ID Range Rules

**Custom CS2 scripts must use IDs ≥ 30000.** Lower ranges collide with stock Jagex scripts already in the 228 base cache and will silently overwrite them. Look at `cs2_source/` — the convention here is `30000.cs2`, `30001.cs2`, etc. The `10300.cs2`-range files are pre-existing project scripts that pre-date this rule.

If you compile a script and it does not appear in game, the most common cause (after the `script.name` Neptune bug) is an ID collision — pick a higher number.

### 5.5 Anatomy of a Source File

`cache/assets/osnr/cs2_library/all.cs2` is the giant signature file: every engine command lives there as a stub like:

```
[command,cc_setgraphic](int $graphic0)()
[command,if_setgraphic](component $component0, int $graphic1)()
[command,oc_param](obj $obj0, param $param1)(int)
```

Your own scripts in `cs2_source/` use those names directly. A trivial script:

```
[clientscript,my_button_onclick]
mes("Hello world!");
```

The `[clientscript,...]` header is the trigger; `[proc,...]` is for procs you `gosub` from other scripts.

---

## 6. Decompiling CS2 from the Cache

Sometimes you want to read what Jagex (or whoever built the base cache) wrote — especially when cloning interface behavior. We use the **runestar.cs2** library, pulled in via:

```kotlin
// cache/build.gradle.kts
implementation("org.runestar.cs2:cs2")
```

That library can take raw script bytes from the cache and emit either an opcode dump or, when it knows enough types, a decompiled pseudocode listing.

### 6.1 Our Decompiler Tool

`cache/src/main/kotlin/com/near_reality/cache_tool/DecompileLeagueTasks.kt` is a small `main` you run via:

```bash
./gradlew :cache:decompileLeagueTasks
```

It:

1. Opens the cache (`Cache.openCache("cache/data/cache", true)`).
2. Pulls the `CLIENTSCRIPTS` archive.
3. For each script ID in `scriptIds` (currently `3204, 3216, 3691, 3692, 3202` — leagues task list scripts):
   - Reads the script bytes from `archive.group(id).file(0)`.
   - Wraps them in `org.runestar.cs2.type.Script`.
   - Walks `script.opcodes` and `script.operands` and prints a raw opcode listing.
   - Dumps switch tables.
4. Writes one `opcodes_<id>.txt` per script under `cache/assets/osnr/leagues_cs2_decompiled/`.

The opcode names come from a hand-built `getOpcodeName(opcode: Int)` map at the bottom of the file (CC_SETGRAPHIC, ENUM, OC_PARAM, SWITCH, etc.). Add to it as you encounter unknowns — anything missing prints as `OP_<num>`.

The Gradle task definition:

```kotlin
tasks.register<JavaExec>("decompileLeagueTasks") {
    group = "_nr_data"
    description = "Decompile league task CS2 scripts from the cache"
    classpath = sourceSets.main.get().runtimeClasspath
    mainClass.set("com.near_reality.cache_tool.DecompileLeagueTasksKt")
    workingDir = rootProject.projectDir
}
```

To target different scripts, edit the `scriptIds` list and re-run.

### 6.2 What You Get

A typical output line looks like:

```
  12: [3619] OC_PARAM                       null
  13: [   0] PUSH_CONST_INT                 1851
  14: [3200] STRUCT_PARAM                   null
```

The first column is the instruction index, the bracketed number is the raw opcode, the third is the symbolic name (or `OP_<n>` if unmapped), and the last is the operand. From a dump like that you can reconstruct the original logic — slow, but it works for non-trivial cases where the runestar high-level decompiler chokes.

If you want a higher-level decompile rather than an opcode listing, the runestar library exposes `Decompiler` / `Loader` classes that can produce pseudocode given a complete script set; the current tool deliberately stays at the opcode level because the partial set we feed it is not enough for the full pipeline.

### 6.3 Inspecting Enums and Structs

The other half of reverse-engineering CS2 behavior is the data the script reads — almost always enums and structs. We have a sister tool:

```bash
./gradlew :cache:dumpLeagueData
```

Source: `cache/src/main/kotlin/com/near_reality/cache_tool/DumpLeagueData.kt`. It loads the cache, calls `EnumDefinitions().load()` and `StructDefinitions().load()`, and walks specified IDs printing their params/values to `cache/assets/osnr/leagues_data_dump/enum_analysis.txt`. Edit the lists at the top of `main()` to dump whatever IDs you are investigating.

A typical reverse-engineering session for an interface looks like:

1. `decompileLeagueTasks` → find which enum/struct IDs the script uses.
2. `dumpLeagueData` → dump those enums/structs to see what fields the script is reading.
3. Repeat until you understand the data chain.
4. Write your own packer (in `cache/src/main/.../packing/custom/`) that emits compatible enums/structs with your data.
5. Optionally clone & modify the CS2 (decompile, re-author in `cs2_source/`, re-compile).

---

## 7. End-to-End Cheat Sheet

```bash
# One-time setup
git clone https://github.com/neptune-ps/neptune ~/Desktop/Neptune
cd ~/Desktop/Neptune/Neptune
# verify BinaryScriptWriterContext.kt lines 73 + 88 use script.name (not script.fullName)
./gradlew :clientscript-compiler:installDist
export JAVA_HOME="/c/Users/DevBox2/.jdks/ms-17.0.18"

# Edit a script
$EDITOR cache/assets/osnr/cs2_source/30100.cs2

# Compile + pack into the cache
./gradlew :cache:compileCS2
./gradlew :cache:generatecacheprod

# Investigate an existing script
$EDITOR cache/src/main/kotlin/com/near_reality/cache_tool/DecompileLeagueTasks.kt   # set scriptIds
./gradlew :cache:decompileLeagueTasks
# → cache/assets/osnr/leagues_cs2_decompiled/opcodes_<id>.txt

# Inspect enums/structs the script depends on
$EDITOR cache/src/main/kotlin/com/near_reality/cache_tool/DumpLeagueData.kt          # set IDs
./gradlew :cache:dumpLeagueData
# → cache/assets/osnr/leagues_data_dump/enum_analysis.txt
```

---

## 8. Common Gotchas

| Symptom | Likely cause |
|---|---|
| Script compiles but in-game lookup fails / wrong script runs | Neptune writing `script.fullName` instead of `script.name` (re-apply the §4.2 fix and rebuild Neptune) |
| Compiled script silently overwrites Jagex behavior | Used a script ID below 30000 |
| `compileCS2` errors with "unknown symbol" for `interface_X` / `component_X` | Missing line in the corresponding `.sym` file under `cs2_symbols/` |
| New CS2 not visible in game after compile | Forgot to re-run `:cache:generatecacheprod` (the packer only runs during cache build) |
| Decompiled output is full of `OP_3619`-style names | Opcode not in `getOpcodeName()` map — add it |
| Neptune build fails | Wrong JDK; needs **Java 17** |
| Gradle reports "cs2.bat: not found" on Linux/Mac | Edit `compileCS2` task to use `cs2` instead of `cs2.bat` |

---

## 9. Reference Links

- **Neptune (compiler)** — https://github.com/neptune-ps/neptune
  - The `clientscript-compiler` subproject is what we invoke.
  - Project-wide README explains the broader Neptune goals (it is also a partial server, but we only use the compiler).
- **runestar/cs2 (decompiler library)** — https://github.com/runestar/cs2
  - Pulled in as the Maven dep `org.runestar.cs2:cs2`.
- **OpenRS2 cache archive** — https://archive.openrs2.org/
  - Where the 228 base cache and key files come from (`download228Cache` task in `cache/build.gradle.kts`).

---

## 10. Files Worth Knowing

| Path | Purpose |
|---|---|
| `cache/build.gradle.kts` | Defines `compileCS2`, `decompileLeagueTasks`, `dumpLeagueData`, `download228Cache` tasks |
| `cache/assets/osnr/neptune.toml` | Neptune project config |
| `cache/assets/osnr/cs2_source/` | Your `.cs2` source |
| `cache/assets/osnr/cs2_library/all.cs2` | Engine command signatures |
| `cache/assets/osnr/cs2_symbols/*.sym` | ID ↔ name maps for every type |
| `cache/assets/osnr/custom_cs2/` | Compiled binaries (input to the packer) |
| `cache/src/main/kotlin/com/near_reality/cache_tool/packing/custom/NearRealityCustomCS2Packer.kt` | Packs `custom_cs2/` into the cache |
| `cache/src/main/kotlin/com/near_reality/cache_tool/DecompileLeagueTasks.kt` | Opcode-level CS2 decompiler |
| `cache/src/main/kotlin/com/near_reality/cache_tool/DumpLeagueData.kt` | Enum/struct inspector |
| `~/Desktop/Neptune/Neptune/clientscript-compiler/src/main/kotlin/me/filby/neptune/clientscript/compiler/writer/BinaryScriptWriterContext.kt` | The file containing the `script.name` fix |

---

That should be everything a new contributor needs to start writing, compiling, and reverse-engineering CS2 on this project. If something here drifts out of date — particularly the Neptune path or the `script.name` line numbers — fix this doc as you go.
