# Implementing a Custom OSRS-Style Interface — Complete Guide

This document walks through every file, tool, and step required to build a fully functional custom interface with authentic OSRS styling on a Near Reality server source. It uses the Task Tracker interface (ID 5103) as the reference implementation.

---

## Table of Contents

1. [Prerequisites & Tools](#prerequisites--tools)
2. [Architecture Overview](#architecture-overview)
3. [Step 1: Choose an Interface ID](#step-1-choose-an-interface-id)
4. [Step 2: Create the Cache Packer](#step-2-create-the-cache-packer)
5. [Step 3: Write CS2 Scripts](#step-3-write-cs2-scripts)
6. [Step 4: Update Symbol Files](#step-4-update-symbol-files)
7. [Step 5: Add Library Stubs](#step-5-add-library-stubs)
8. [Step 6: Register in GameInterface](#step-6-register-in-gameinterface)
9. [Step 7: Register the Packer in TypeParser](#step-7-register-the-packer-in-typeparser)
10. [Step 8: Create the Server Plugin](#step-8-create-the-server-plugin)
11. [Step 9: Register the Plugin Module](#step-9-register-the-plugin-module)
12. [Step 10: Add a Command to Open It](#step-10-add-a-command-to-open-it)
13. [Step 11: Build and Test](#step-11-build-and-test)
14. [Key Concepts & Lessons Learned](#key-concepts--lessons-learned)
15. [Complete File Reference](#complete-file-reference)

---

## Prerequisites & Tools

### Required Tools

| Tool | Purpose | Location |
|------|---------|----------|
| **Neptune CS2 Compiler** | Compiles `.cs2` source into binary clientscripts | `~/Desktop/Neptune/clientscript-compiler/` |
| **Gradle** | Build system for the server project | Project root |
| **JDK 21** | Java runtime for compilation | System-installed |

### Required Knowledge

- Basic understanding of the OSRS client-server architecture
- Familiarity with Kotlin/Java
- Understanding of the IF3 component system (containers, rectangles, text, sprites)

### Key Gradle Commands

| Command | What it does |
|---------|-------------|
| `./gradlew :cache:compileCS2` | Compiles CS2 source files via Neptune |
| `./gradlew :app:generateCacheProd` | Compiles Java/Kotlin AND runs TypeParser to pack the cache |
| `./gradlew :app:runProduction` | Starts the server |

**Build order**: Always `compileCS2` → `generateCacheProd` → `runProduction`. You can skip `compileCS2` if no `.cs2` files changed. After changing interface components, nuke your client's cache so it re-downloads.

---

## Architecture Overview

A custom interface has **4 layers**:

```
┌──────────────────────────────────────────────────────────┐
│  1. CACHE PACKER (Kotlin)                                │
│     Defines IF3 component layout programmatically        │
│     Writes components to INTERFACES archive at build     │
├──────────────────────────────────────────────────────────┤
│  2. CS2 SCRIPTS (RuneScript, compiled by Neptune)        │
│     Runs on the CLIENT at runtime                        │
│     Creates dynamic UI elements (steelborder, tabs)      │
│     Handles hover effects, scroll management             │
├──────────────────────────────────────────────────────────┤
│  3. SERVER PLUGIN (Kotlin)                               │
│     Handles player interactions (tab clicks, etc.)       │
│     Populates interface with data (text, visibility)     │
│     Sends clientscript calls to trigger CS2              │
├──────────────────────────────────────────────────────────┤
│  4. REGISTRATION (Java/Gradle)                           │
│     GameInterface enum, TypeParser, Gradle modules       │
│     Wires everything together                            │
└──────────────────────────────────────────────────────────┘
```

### Critical Design Rule

**Do NOT manually build the border, title bar, or close button.** The IF3 format does not serialize `alternateSpriteId` (hover sprites), so manual close buttons won't have hover effects. Always use the stock OSRS `steelborder` CS2 procedure, which dynamically creates all the chrome with proper hover animations.

### Steelborder Z-Order Rule

Steelborder creates dynamic children on whatever component you target. Dynamic children render **after** all static children of the same parent, covering them. To prevent this:

- Create a **dedicated border layer** component (e.g., component 1) for steelborder to target
- All your content components are siblings with higher IDs, so they render **on top** of the border

```
Root (0)
  ├── Border layer (1) ← steelborder targets this
  │     └── [dynamic] border chrome, title, close button
  ├── Tab container (2) ← renders on top of border
  ├── Content (5) ← renders on top of border
  └── ...
```

---

## Step 1: Choose an Interface ID

Pick an unused interface ID. Check `GameInterface.java` for existing IDs. The Task Tracker uses **5103**.

---

## Step 2: Create the Cache Packer

**File**: `cache/src/main/kotlin/com/near_reality/cache_tool/packing/custom/TaskTrackerInterfacePacker.kt`

The packer extends `ClownInterfacePacker` and defines every pre-built component programmatically. Each component is a `ComponentDefinitions` object.

### Component Types

| Type | Name | Key Fields |
|------|------|------------|
| 0 | Container | scrollWidth, scrollHeight, noClickThrough |
| 3 | Rectangle | color, filled, opacity |
| 4 | Text | font, text, xAllignment, yAllignment, color, textShadowed |
| 5 | Sprite | spriteId, textureId, spriteTiling, opacity |
| 6 | Model | modelId, rotation, zoom, animation |
| 9 | Line | lineWidth, color, lineDirection |

### Component Reference Encoding

Listeners use packed component references: `(interfaceId << 16) | componentId`

For interface 5103, component 6: `5103 << 16 | 6` = `334430214`

### Scrollbar Setup

The OSRS scrollbar requires 3 components working together:

```kotlin
// 1. Scroll layer — the scrollable content container
ComponentDefinitions().apply {
    interfaceId = ID; componentId = SCROLL_LAYER
    isIf3 = true; type = 0; parentId = CONTENT
    scrollWidth = W; scrollHeight = MAX_HEIGHT  // total scrollable canvas
}.pack()

// 2. Scroll wheel handler — invisible overlay that captures wheel events
ComponentDefinitions().apply {
    interfaceId = ID; componentId = SCROLL_WHEEL
    isIf3 = true; type = 0; parentId = CONTENT
    onScrollWheelListener = arrayOf(
        36,                              // script ID: scrollbar_scroll
        ID shl 16 or SCROLL_BAR,        // scrollbar component ref
        ID shl 16 or SCROLL_LAYER,      // scroll layer component ref
        -2147483646                      // scroll direction constant (INT_MIN + 2)
    )
}.pack()

// 3. Scrollbar — initialized by client script 30 (scrollbar_vertical)
ComponentDefinitions().apply {
    interfaceId = ID; componentId = SCROLL_BAR
    isIf3 = true; type = 0; parentId = CONTENT
    onLoadListener = arrayOf(
        30,                              // script ID: scrollbar_vertical
        -2147483645,                     // "this component" constant (INT_MIN + 3)
        ID shl 16 or SCROLL_LAYER       // scroll layer component ref
    )
}.pack()
```

### Content Inset

All content must be inset ~6px from the steelborder frame edges (left, right, bottom) to avoid overlapping the border:

```kotlin
const val FRAME_INSET = 6
const val INNER_W = WIDTH - FRAME_INSET * 2
```

### Full Packer Example

```kotlin
class TaskTrackerInterfacePacker(cache: Cache) : ClownInterfacePacker(cache) {

    companion object {
        const val INTERFACE_ID = 5103
        const val WIDTH = 480
        const val HEIGHT = 320
        // ... dimensions, colors, child IDs
    }

    override fun pack() {
        // Root container (centered on screen)
        ComponentDefinitions().apply {
            interfaceId = INTERFACE_ID; componentId = 0
            isIf3 = true; type = 0
            width = WIDTH; height = HEIGHT
            xMode = 1; yMode = 1        // centered
            noClickThrough = true
        }.pack()

        // Border layer (steelborder target, component 1)
        ComponentDefinitions().apply {
            interfaceId = INTERFACE_ID; componentId = 1
            isIf3 = true; type = 0; parentId = 0
            x = 0; y = 0; width = WIDTH; height = HEIGHT
        }.pack()

        // Tab container (CS2 creates tabs here, component 2)
        // ... summary text, separator, content area, scroll, rows
    }
}
```

See the [Complete File Reference](#complete-file-reference) for the full packer source.

---

## Step 3: Write CS2 Scripts

CS2 (ClientScript 2) runs on the **client** and handles dynamic UI that can't be pre-built in the packer.

### Script Files Location

Source: `cache/assets/osnr/cs2_source/`
Compiled output: `cache/assets/osnr/custom_cs2/`
Config: `cache/assets/osnr/neptune.toml`

### Neptune Configuration

```toml
name = "task-tracker"
sources = ["cs2_source/", "cs2_library/"]
symbols = ["cs2_symbols/"]
libraries = ["cs2_library/"]

[writer.binary]
output = "custom_cs2/"
```

### Script ID Assignment

The filename IS the script ID. File `10400.cs2` compiles to script ID 10400. The script name inside the file must match what's in `clientscript.sym`.

### The Task Tracker Uses 3 CS2 Scripts

#### 10400.cs2 — Interface Initialization

Called once when the interface opens. Creates the steelborder chrome and builds the initial tabs.

```cs2
[clientscript,task_tracker_init]
~steelborder(task_tracker:1, "Task Tracker", 0);
~task_build_tabs(0);
```

**Key points:**
- `~steelborder(component, title, flags)` is a stock OSRS proc already in the cache
- It creates the border frame, title bar, and close button with hover animations
- Target the **border layer** (component 1), not the root (component 0)
- `~task_build_tabs(0)` calls our custom proc to create tabs with tab 0 selected

#### 10401.cs2 — Tab Switch Handler

Called by the server each time the player switches tabs. Rebuilds tabs, resizes scroll area, and resets scrollbar.

```cs2
[clientscript,test_tab_switch](int $selectedTab, int $scrollHeight)
~task_build_tabs($selectedTab);
if_setscrollsize(if_getwidth(task_tracker:6), $scrollHeight, task_tracker:6);
if_setscrollpos(0, 0, task_tracker:6);
if_setonresize("scrollbar_vertical(task_tracker:8, task_tracker:6)", task_tracker:8);
cc_deleteall(task_tracker:8);
if_callonresize(task_tracker:8);
```

**Scrollbar reset trick:**
1. `if_setonresize` — sets a resize handler on the scrollbar that re-calls `scrollbar_vertical`
2. `cc_deleteall` — nukes the scrollbar's existing dynamic children (track, thumb, arrows)
3. `if_callonresize` — triggers the handler, which rebuilds the scrollbar fresh with the new scroll height

#### 10402.cs2 — Tab Builder Proc

Creates 6 stone-bordered tabs as dynamic children of the tab container. Each tab has 11 sub-components: background rect, stone center texture, 4 edges, 4 corners, and a text label.

**Key techniques:**

Stone border sprites (style 3712):
```
Center: tradebacking (ID 297)
Edges: stoneborder_top/bottom/left/right (IDs 3528-3531)
Corners: stoneborder_tl/tr/bl/br (IDs 3524-3527)
```

Tab selection state — opacity controls brightness:
```cs2
def_int $trans = 100;     // idle: darkened
if ($sel = 1) {
    $trans = 0;            // selected: fully opaque
}
cc_settrans($trans);       // applied to all stone sprites
```

Hover effect using stock OSRS `cc_colour_swapper`:
```cs2
cc_setonmouseover("cc_colour_swapper(event_com, event_comsubid, 9075297)");
cc_setonmouseleave("cc_colour_swapper(event_com, event_comsubid, 7496785)");
```

**Hook syntax in Neptune:** Hooks are **string literals** containing a clientscript call expression. The quotes are required. Event variables (`event_com`, `event_comsubid`) are built-in commands available inside handlers.

Clickable tabs:
```cs2
cc_setop(1, "Select");     // makes the component clickable
```

The server receives clicks on the parent component (tab container) with `slotID` = the sub-ID of the clicked dynamic child. Divide by 11 (components per tab) to get the tab index.

### CS2 Command Reference

| Command | Description |
|---------|-------------|
| `cc_create(parent, type, subId)` | Create a dynamic child component |
| `cc_deleteall(component)` | Delete all dynamic children |
| `cc_setposition(x, y, xMode, yMode)` | Set position on active component |
| `cc_setsize(w, h, wMode, hMode)` | Set size on active component |
| `cc_setcolour(rgb)` | Set color (rectangles/text) |
| `cc_setfill(boolean)` | Set rectangle fill |
| `cc_settrans(0-255)` | Set transparency (0=opaque, 255=invisible) |
| `cc_setgraphic(graphicName)` | Set sprite graphic |
| `cc_settiling(boolean)` | Tile the sprite |
| `cc_settext(string)` | Set text content |
| `cc_settextfont(fontName)` | Set font (p11_full, p12_full, verdana_11pt_regular) |
| `cc_settextalign(h, v, lineHeight)` | Set text alignment |
| `cc_settextshadow(boolean)` | Enable text shadow |
| `cc_setop(slot, text)` | Set click operation (makes clickable) |
| `cc_setonmouseover("hook")` | Set mouseover handler |
| `cc_setonmouseleave("hook")` | Set mouseleave handler |
| `if_setscrollsize(w, h, component)` | Resize scroll canvas |
| `if_setscrollpos(x, y, component)` | Set scroll position |
| `if_setonresize("hook", component)` | Set resize handler on component |
| `if_callonresize(component)` | Trigger resize handler |
| `~procName(args)` | Call a proc |

### Position/Size Constants

```
^setpos_abs_left / ^setpos_abs_top = 0
^setpos_abs_centre = 1
^setpos_abs_right / ^setpos_abs_bottom = 2
^setsize_abs = 0
^setsize_minus = 1
^iftype_rectangle = 3
^iftype_text = 4
^iftype_graphic = 5
```

---

## Step 4: Update Symbol Files

Neptune uses symbol files to resolve human-readable names to numeric IDs.

### interface.sym — Interface ID → name

**File**: `cache/assets/osnr/cs2_symbols/interface.sym`

```
5103	task_tracker
```

### component.sym — Packed component ref → name

**File**: `cache/assets/osnr/cs2_symbols/component.sym`

Computed as `interfaceId << 16 | componentId`:

```
334430208	task_tracker:0
334430209	task_tracker:1
334430210	task_tracker:2
334430214	task_tracker:6
334430216	task_tracker:8
```

Only components referenced in CS2 scripts need entries.

### clientscript.sym — Script ID → name

**File**: `cache/assets/osnr/cs2_symbols/clientscript.sym`

```
10400	[clientscript,task_tracker_init]
10401	[clientscript,test_tab_switch]
10402	[proc,task_build_tabs]
```

The script ID must match the filename (10400.cs2 → ID 10400).

### graphic.sym — Graphic ID → name

**File**: `cache/assets/osnr/cs2_symbols/graphic.sym`

Only needed if your CS2 uses `cc_setgraphic` with custom sprite references:

```
3524	stoneborder_tl
3525	stoneborder_tr
3526	stoneborder_bl
3527	stoneborder_br
3528	stoneborder_top
3529	stoneborder_bottom
3530	stoneborder_left
3531	stoneborder_right
```

**Important**: Duplicate names across symbol files will cause Neptune to crash with `"Unable to add basic"`.

---

## Step 5: Add Library Stubs

**File**: `cache/assets/osnr/cs2_library/stubs.cs2`

Stubs declare the signatures of **external** procedures/scripts that already exist compiled in the OSRS cache. Neptune needs these to type-check your calls.

```
[proc,steelborder](component $component0, string $text0, int $flags1)(int)
[proc,scrollbar_vertical](component $component0, component $component1, string $graphic2, string $graphic3, string $graphic4, string $graphic5, string $graphic6, string $graphic7)()
[clientscript,cc_colour_swapper](component $component0, int $subId1, int $colour2)
[clientscript,scrollbar_vertical](component $scrollbar0, component $scrollLayer1)
```

**Note**: `scrollbar_vertical` has BOTH a proc stub (8 params, for direct calls) and a clientscript stub (2 params, for use in hooks). These are different things — the clientscript version is script ID 30 used in `if_setonresize("scrollbar_vertical(...)")`.

---

## Step 6: Register in GameInterface

**File**: `core/src/main/java/com/zenyte/game/GameInterface.java`

```java
TASK_TRACKER(5103, CENTRAL),
```

The second parameter is the `InterfacePosition`:
- `CENTRAL` — opens in the center of the screen (most common)
- `OVERLAY` — overlays on top of the game view
- `SINGLE_TAB` — opens in a tab slot

---

## Step 7: Register the Packer in TypeParser

**File**: `cache/src/main/java/mgi/tools/parser/TypeParser.java`

### Add the import:

```java
import com.near_reality.cache_tool.packing.custom.TaskTrackerInterfacePacker;
```

### Add the packer call (near the other packer calls in `main()`):

```java
new TaskTrackerInterfacePacker(cache).pack();
```

---

## Step 8: Create the Server Plugin

The plugin handles all server-side logic: what happens when the interface opens, when tabs are clicked, and what data to display.

### Plugin Build File

**File**: `plugins/interfaces/task-tracker/build.gradle.kts`

```kotlin
plugins {
    id("near-reality-project-kotlin")
}
group = "com.near_reality.plugins.interfaces"
version = "0.1.0"
dependencies {
    implementation(projects.scripts.interfaces)
}
```

### Plugin Implementation

**File**: `plugins/interfaces/task-tracker/src/main/kotlin/com/near_reality/plugins/interfaces/task_tracker/TaskTrackerInterface.kt`

Key patterns:

```kotlin
class TaskTrackerInterface : InterfaceScript() {
    init {
        GameInterface.TASK_TRACKER {

            // Handle clicks on dynamic tab children (CS2-created)
            "TabClick"(CHILD_TAB_CONTAINER) {
                val tabIndex = slotID / COMPONENTS_PER_TAB
                // ... handle tab switch
            }

            opened {
                sendInterface()                              // Send the interface to client
                packetDispatcher.sendClientScript(10400)      // Trigger steelborder + tabs
                packetDispatcher.sendComponentSettings(       // Enable clicking on tabs
                    INTERFACE_ID, CHILD_TAB_CONTAINER,
                    0, NUM_TABS * COMPONENTS_PER_TAB,
                    AccessMask.CLICK_OP1
                )
                switchToTab(0)                                // Populate initial data
            }

            closed { /* cleanup */ }
        }
    }
}
```

### Server → Client Communication

| Method | Purpose |
|--------|---------|
| `sendInterface()` | Opens the interface on the client |
| `sendClientScript(id, args...)` | Calls a CS2 clientscript |
| `sendComponentText(ifId, childId, text)` | Set text content (supports `<col=hex>` tags) |
| `sendComponentVisibility(ifId, childId, hidden)` | Show/hide a component (`false` = show) |
| `sendComponentSettings(ifId, childId, start, end, masks)` | Enable click operations on components |
| `send(IfSetScrollPos(ifId, childId, scrollY))` | Reset scroll position |

### Text Color Tags

Use inline HTML-style color tags in `sendComponentText`:

```kotlin
"<col=ff981f>Gold text</col>"
"<col=999999>✔ Completed task</col>"
"<col=00ff00>100</col> points"
```

### Important: Use `.open()` Not `.sendInterface()`

When opening the interface from commands or other code, always use:

```java
GameInterface.TASK_TRACKER.open(player);
```

**NOT** `player.getInterfaceHandler().sendInterface(GameInterface.TASK_TRACKER)` — the latter bypasses the plugin's `opened` block, so steelborder and data population won't run.

---

## Step 9: Register the Plugin Module

### settings.gradle.kts

```kotlin
":plugins:interfaces:task-tracker",
```

### app/build.gradle.kts

```kotlin
runtimeOnly(project(":plugins:interfaces:task-tracker"))
```

---

## Step 10: Add a Command to Open It

**File**: `core/src/main/java/com/zenyte/game/world/entity/player/GameCommands.java`

```java
new Command(PlayerPrivilege.DEVELOPER, "tasklist", "Open the Task Tracker.",
    (p, args) -> GameInterface.TASK_TRACKER.open(p));
```

---

## Step 11: Build and Test

```bash
# 1. Compile CS2 scripts (only if .cs2 files changed)
./gradlew :cache:compileCS2

# 2. Compile Java/Kotlin + pack the cache
./gradlew :app:generateCacheProd

# 3. Start the server
./gradlew :app:runProduction
```

After changing interface components, **nuke your client's cache** so it re-downloads the updated version.

In-game: `::tasklist` or `::openif 5103`

---

## Key Concepts & Lessons Learned

### IF3 Limitations

- **`alternateSpriteId` is NOT serialized in IF3.** Hover sprites set in the packer are silently ignored. Use CS2 event handlers instead.
- **Dynamic children render after static children.** Steelborder's chrome will cover your pre-built components if they share the same parent. Use a dedicated border layer.

### CS2 Hook Syntax

Hooks in Neptune are **string literals** containing a clientscript call:

```cs2
// Correct — string literal with parentheses
cc_setonmouseover("cc_colour_swapper(event_com, event_comsubid, 9075297)");

// WRONG — bare reference
cc_setonmouseover(some_script);

// WRONG — raw script ID
cc_setonmouseover(85, args...);
```

### Scrollbar Re-initialization

After changing scroll size with `if_setscrollsize`, the scrollbar's visual thumb doesn't automatically update. The workaround:

```cs2
if_setonresize("scrollbar_vertical(component:scrollbar, component:scrollLayer)", component:scrollbar);
cc_deleteall(component:scrollbar);
if_callonresize(component:scrollbar);
```

This sets a resize handler → nukes existing scrollbar children → triggers the handler to rebuild.

### Performance: Minimize Packets

Each `sendComponentText` and `sendComponentVisibility` call sends a network packet. For tab switching:

- Only send packets for rows that have content
- Track the previous tab's row count and only hide the delta
- Send the CS2 script call BEFORE row data so tabs update first

### Neptune Compilation Gotchas

- Graphic references in `cc_setgraphic()` must use symbolic names from `graphic.sym`, not raw IDs
- Duplicate names in symbol files cause a crash: `"Unable to add basic"`
- Script IDs must match filenames (10400.cs2 → ID 10400)
- Clientscript stubs need separate entries from proc stubs even if they share a name (different namespaces)

---

## Complete File Reference

### Files to CREATE

| # | File | Purpose |
|---|------|---------|
| 1 | `cache/src/.../TaskTrackerInterfacePacker.kt` | IF3 component layout (packer) |
| 2 | `cache/assets/osnr/cs2_source/10400.cs2` | Init script (steelborder + tabs) |
| 3 | `cache/assets/osnr/cs2_source/10401.cs2` | Tab switch script (resize + scroll reset) |
| 4 | `cache/assets/osnr/cs2_source/10402.cs2` | Tab builder proc (stone-bordered tabs) |
| 5 | `plugins/interfaces/task-tracker/build.gradle.kts` | Plugin module build config |
| 6 | `plugins/interfaces/task-tracker/.../TaskTrackerInterface.kt` | Server plugin |

### Files to MODIFY

| # | File | Change |
|---|------|--------|
| 7 | `cache/assets/osnr/cs2_symbols/interface.sym` | Add `5103 task_tracker` |
| 8 | `cache/assets/osnr/cs2_symbols/component.sym` | Add packed component refs |
| 9 | `cache/assets/osnr/cs2_symbols/clientscript.sym` | Add script IDs |
| 10 | `cache/assets/osnr/cs2_symbols/graphic.sym` | Add stone border sprite names |
| 11 | `cache/assets/osnr/cs2_library/stubs.cs2` | Add external proc/script stubs |
| 12 | `core/.../GameInterface.java` | Add enum entry |
| 13 | `cache/.../TypeParser.java` | Add import + packer call |
| 14 | `settings.gradle.kts` | Register plugin module |
| 15 | `app/build.gradle.kts` | Add runtime dependency |
| 16 | `core/.../GameCommands.java` | Add `::tasklist` command |

### Key Directory Structure

```
cache/
├── assets/osnr/
│   ├── cs2_source/          ← CS2 source files (human-readable)
│   │   ├── 10400.cs2
│   │   ├── 10401.cs2
│   │   └── 10402.cs2
│   ├── cs2_library/
│   │   └── stubs.cs2        ← External proc/script signatures
│   ├── cs2_symbols/
│   │   ├── interface.sym    ← Interface ID → name
│   │   ├── component.sym   ← Packed component ref → name
│   │   ├── clientscript.sym ← Script ID → name
│   │   ├── graphic.sym     ← Graphic ID → name
│   │   └── commands.sym     ← Built-in CS2 commands (read-only)
│   ├── custom_cs2/          ← Compiled CS2 output (auto-generated)
│   └── neptune.toml         ← Neptune compiler config
├── src/main/kotlin/.../packing/custom/
│   └── TaskTrackerInterfacePacker.kt
└── src/main/java/.../TypeParser.java

core/src/main/java/.../
├── GameInterface.java
├── GameCommands.java
└── content/skillpoints/
    ├── SkillPointTask.java
    ├── SkillPointManager.java
    ├── GameDifficulty.java
    └── TaskCategory.java

plugins/interfaces/task-tracker/
├── build.gradle.kts
└── src/main/kotlin/.../TaskTrackerInterface.kt

scripts/interfaces/src/main/kotlin/.../InterfaceScript.kt  ← Base class (DSL)
```

---

## Adapting for a New Interface

To create a completely different interface (e.g., ID 5104):

1. **Pick an ID** — check GameInterface.java for unused IDs
2. **Create a packer** — copy TaskTrackerInterfacePacker, change INTERFACE_ID and component layout
3. **Write CS2** — at minimum: one script calling `~steelborder(your_interface:1, "Title", 0)`
4. **Update symbols** — interface.sym, component.sym, clientscript.sym
5. **Register** — GameInterface enum, TypeParser packer call, Gradle module
6. **Write plugin** — extend InterfaceScript, handle opened/closed/clicks
7. **Build** — compileCS2 → generateCacheProd → runProduction

The steelborder + border layer + content inset pattern works for any interface that needs the authentic OSRS look.
