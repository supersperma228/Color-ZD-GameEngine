# Color-ZD Engine — Lua Scripting Reference

**Version:** 1.0
**Scripting language:** Lua 5.4 (sandboxed)
**Audience:** Gameplay scripters, technical designers, and engine programmers

---

## Table of Contents

1. [Overview](#1-overview)
2. [Quick Start](#2-quick-start)
3. [How Scripts Execute](#3-how-scripts-execute)
4. [The Sandbox](#4-the-sandbox)
5. [Object Context & Selection](#5-object-context--selection)
6. [Object References](#6-object-references)
7. [Collision & Trigger Callbacks](#7-collision--trigger-callbacks)
8. [API Reference](#8-api-reference)
   - 8.1 [`Input`](#81-input)
   - 8.2 [`Object`](#82-object)
   - 8.3 [`World`](#83-world)
   - 8.4 [`Scene`](#84-scene)
   - 8.5 [`Debug`](#85-debug)
   - 8.6 [`Trigger`](#86-trigger)
   - 8.7 [`JSON`](#87-json)
   - 8.8 [`Audio`](#88-audio)
   - 8.9 [`SoundObject`](#89-soundobject)
   - 8.10 [`Message`](#810-message)
   - 8.11 [`UI`](#811-ui)
   - 8.12 [`Render`](#812-render)
   - 8.13 [`Graphics`](#813-graphics)
   - 8.14 [`WorldTime`](#814-worldtime)
   - 8.15 [`NavMesh`](#815-navmesh)
   - 8.16 [`ManualPath`](#816-manualpath)
   - 8.17 [`AI`](#817-ai) (Behavior Trees)
   - 8.18 [`Cursor`](#818-cursor)
   - 8.19 [`ParticleSystem`](#819-particlesystem)
   - 8.20 [`Physics`](#820-physics)
   - 8.21 [`Time`](#821-time)
9. [Object Types](#9-object-types)
10. [Behavior Tree AI Deep Dive](#10-behavior-tree-ai-deep-dive)
11. [Debugging & Error Handling](#11-debugging--error-handling)
12. [Performance & Best Practices](#12-performance--best-practices)
13. [Complete Example Scripts](#13-complete-example-scripts)
14. [Function Index (Alphabetical)](#14-function-index-alphabetical)

---

## 1. Overview

Every scriptable object in ColorCore (meshes, primitives, triggers, particle systems, audio sources, etc.) can carry a **Lua script**. The engine embeds Lua 5.4 through a custom C API binding layer (`RegisterBindings`) and executes each object's script inside an **isolated, memory- and instruction-limited sandbox** (`ColorCore::LuaSandbox`).

Scripts interact with the engine exclusively through a set of **global Lua tables** (`Object`, `World`, `Scene`, `Physics`, `Audio`, `UI`, …) that are injected into the sandbox before the script body runs. There is no `require`, no filesystem access outside the explicit `JSON.*` API, and no ability to reach outside the sandbox.

### Key architectural facts

| Concept | Behavior |
|---|---|
| One sandbox per script **text** | A fresh `lua_State` is created the first time a given script string runs, and is **recreated from scratch** any time its source code changes (hot reload). |
| No `Update()` function | The top-level body of the script *is* the per-frame update. There is no wrapper function you must define — top-level statements run every frame the object is active. |
| Implicit "current object" | Almost every `Object.*`/`Physics.*`/`UI.*` call operates on an implicit **currently selected object**, exposed to C++ as the Lua global `CurrentObject`. |
| Special callback functions | If your script *defines* certain global functions (`OnCollisionEnter`, `OnTriggerEnter`, etc.) the engine will detect and invoke them separately, in addition to the per-frame body. |
| Deferred lifecycle | `Object.Spawn`, `Object.Destroy`, and `Trigger.Create` do **not** happen immediately — they queue an operation that is applied once per frame by the engine after all scripts have run. |

---

## 2. Quick Start

A minimal script that spins the object it's attached to and logs a message once:

```lua
-- This entire block runs every frame.
Object.Rotate(0, 45 * Time.deltaTime, 0)

if Input.GetKeyDown("Space") then
    Debug.Log(Object.GetName() .. " jumped!")
end
```

There is no `function Update()` wrapper — you write the frame body directly.

---

## 3. How Scripts Execute

### 3.1 Compilation & caching

Internally each script is identified by a **script key** (its name, or a hash of its source if unnamed). The engine keeps a cache (`scriptCache`) of compiled chunks:

- If the script's source text is unchanged since the last run, the compiled function reference is reused — **no recompilation cost**.
- If the source text changes (e.g. you edit it in the editor), the engine tears down the old sandbox entirely and builds a brand-new one, then recompiles the script from scratch. **All Lua-side global state for that script is lost on edit** — do not rely on top-level `local`/global variables surviving a hot reload; use `Message.SetGlobal`/`JSON.Save` for anything that must persist.

### 3.2 Per-frame execution (`RunScript`)

Every frame, for every active object with a non-empty script, the engine:

1. Clears the object-selection stack (see [§5](#5-object-context--selection)).
2. Sets `CurrentObject` (the implicit context) to that object.
3. Injects a fresh `Time` global table with a single field: `Time.deltaTime`.
4. Calls the compiled script chunk as a protected call (`pcall`) with **0 arguments**.
5. If the call throws a Lua error, it is caught, logged to the console as
   `Lua Error (<scriptKey>): <message>`, and execution for that object stops for the frame (no crash).
6. After a successful run, the engine re-scans the script text for the special callback function names (see below) and re-caches references to any of them that are defined as globals.

### 3.3 Special callback functions

If the **source text** of your script contains the token `OnCollisionEnter` (etc.) **anywhere** (this is a plain substring search, not a syntax-aware check — even a comment containing the word will trigger detection), the engine will, after your script runs once, look up a **global function** with that exact name and keep a reference to it. The supported names are:

- `OnCollisionEnter(otherObjectName)`
- `OnCollisionStay(otherObjectName)`
- `OnCollisionExit(otherObjectName)`
- `OnTriggerEnter(otherObjectName)`
- `OnTriggerStay(otherObjectName)`
- `OnTriggerExit(otherObjectName)`

These are invoked by the engine's physics system (outside these binding files) whenever a physical collision or trigger overlap event occurs for the object. Each callback receives exactly **one string argument**: the name of the other object involved. `CurrentObject` and `Time.deltaTime` are set up identically to the per-frame body before the callback runs.

```lua
-- Runs every frame
Object.Move(0, 0, 5 * Time.deltaTime)

function OnTriggerEnter(otherName)
    Debug.Log("Entered by: " .. otherName)
    UI.ShowScene("Prompt")
end

function OnTriggerExit(otherName)
    UI.HideScene("Prompt")
end
```

> **Important:** these must be declared as plain **global functions** (`function Name(...) ... end`), not `local function`. Only globals are visible to the engine's lookup.

### 3.4 `RunSandboxTest`

The engine can validate a script in complete isolation (used by editor "Test Script" tooling) via `RunSandboxTest`. This creates a throw-away sandbox, registers all bindings, sets `Time.deltaTime = 0.016`, sets `CurrentObject = nil`, compiles, and runs the script once, returning any compile/runtime error string. Because `CurrentObject` is `nil` in this mode, any call that dereferences the current object silently no-ops or returns nothing — this is expected and safe.

---

## 4. The Sandbox

Each script executes inside a `ColorCore::LuaSandbox` — a fully isolated `lua_State` with hard resource limits enforced at the C level, not just convention.

### 4.1 Resource limits

| Limit | Default | Enforcement |
|---|---|---|
| Memory | **4 MiB** per script instance | Custom Lua allocator rejects any allocation that would exceed the budget; allocation returns `NULL` (Lua raises an out-of-memory error). |
| Instruction count | **200,000** "counted" instructions per protected call | A debug hook fires every 1,000 VM instructions; once the running total exceeds the limit, the hook raises a Lua error (`"Lua sandbox instruction limit exceeded"`), aborting the call. |
| Budget reset | Every `LoadBuffer` / `ProtectedCall` | The instruction counter and memory "limit reached" flags are reset at the start of every top-level script invocation (each frame, each callback) — the budget is **per call**, not cumulative across frames. |

Because the limit is per protected call, an expensive `for` loop that's fine spread across many frames can still fail if it all happens inside a single call — keep any given frame's work well under 200k VM instructions (roughly: a few tens of thousands of Lua statements is safe; hundreds of thousands of iterations of tight math loops is not).

### 4.2 Available standard libraries

Only these Lua standard libraries are opened:

- `_G` (base library: `print`, `pairs`, `ipairs`, `tostring`, `type`, `pcall`, `error`, `setmetatable`, etc.)
- `table`
- `string`
- `math`
- `utf8`
- `os` — **restricted**, see below

`io`, `package`, `debug`, and `coroutine` are **never opened** and are simply absent as globals.

### 4.3 Explicitly removed globals

After the libraries are opened, the following are stripped to `nil`:

- `dofile`
- `load`
- `loadfile`
- `collectgarbage`

And from the `os` table specifically, these fields are stripped:

- `os.execute`
- `os.exit`
- `os.getenv`
- `os.remove`
- `os.rename`
- `os.setlocale`
- `os.tmpname`

Safe `os` functions such as `os.time`, `os.clock`, and `os.date` remain available.

### 4.4 Practical implications

- You **cannot** load or execute arbitrary code strings, read/write files directly, spawn processes, or read environment variables from Lua. All file I/O must go through the explicit `JSON.*` API, which is sandboxed at the C++ level to whatever paths your engine build permits.
- There is no `coroutine` library — you cannot `yield` mid-script. Long-running behavior must be expressed as state machines that progress a little each frame (see the `AI` behavior-tree system for a built-in pattern).
- `print()` exists (from the base library) but writes to the process's standard output, not the in-editor console — use `Debug.Log` for visible in-engine logging.

---

## 5. Object Context & Selection

Scripts do not receive an explicit "self" parameter. Instead, most `Object.*` functions act on a hidden global userdata pointer called `CurrentObject`, which the engine sets to the script's owning object before every run. The `Object` table provides a small **selection stack** so you can temporarily redirect calls to another object (parents, children, or any object in the scene) and restore the original selection afterward — similar in spirit to matrix push/pop in immediate-mode graphics APIs.

### 5.1 Selecting a different object

```lua
Object.Select("Enemy_01")   -- select by query (see path syntax below)
Object.Move(0, 0, 1)        -- moves Enemy_01, NOT the script's own object!
```

`Object.Select`, `Object.SelectChild`, and `Object.SelectParent` **permanently change** `CurrentObject` for the rest of this script call (and any callback re-entry) unless you explicitly restore it. Always pair a manual redirection with `Object.Push` / `Object.Pop`:

```lua
Object.Push()                 -- remember current selection
Object.Select("Enemy_01")
Object.Move(0, 0, 1)
Object.Pop()                  -- restore original selection
-- CurrentObject is back to the script's own object here
```

At the **start of every script/callback invocation** the engine clears the internal selection stack and resets `CurrentObject` to the script's own object, so stray un-popped pushes from a previous frame never leak — but within a single invocation, unmatched pushes/pops will leave the wrong object selected for the remainder of that call.

### 5.2 Selection functions (all live under `Object`)

| Function | Effect |
|---|---|
| `Object.Select(query)` | Resolves `query` relative to the current object (see path syntax) and makes it the new `CurrentObject`. Returns `true`/`false` for success. |
| `Object.SelectChild(query)` | Like `Select`, but the resolved object is only accepted if it is a **descendant** of the previously selected object; otherwise selection becomes `nil`. |
| `Object.SelectParent()` | Selects the direct parent of the current object (or `nil` if none). |
| `Object.Push()` | Pushes the current selection onto an internal stack (does not change selection). |
| `Object.Pop()` | Pops the stack and restores that selection as `CurrentObject`. If the stack is empty, selects `nil`. |
| `World.Select(query)` | Equivalent to `Object.Select`, but resolves the query **without** any object context — always behaves like an absolute/global lookup. |

### 5.3 Object query / path syntax

Every function that accepts a "query" string (`Object.Select`, `World.Select`, the implicit target argument accepted by most `Object.*`/`Trigger.*`/`SoundObject.*`/`Physics.*` calls, etc.) understands the following syntax:

| Pattern | Meaning |
|---|---|
| `"Name"` | First tries a **direct child** of the current context object named `Name`. If not found and the calling function allows deep search, searches all **descendants**. Falls back to a **global name search** across the whole scene if nothing else matches. |
| `"Child/Grandchild"` | Relative path: walks down named children one path segment at a time from the current context. |
| `".."` | Selects the parent of the current context. |
| `"../Sibling"` | Relative path starting with one or more `..` segments to move up the hierarchy before descending again. |
| `"/Root/Child"` or `"World/Root/Child"` | **Absolute path** from the scene root — finds a root-level (parentless) object named `Root`, then walks down named children. The leading `World/` is optional and stripped if present. |
| `"World"` | Selects nothing directly (not a valid absolute target by itself) — used as the table root prefix. |

If no context object is available (e.g., no object currently selected) queries fall back to an absolute/global name search.

---

## 6. Object References

Wherever the API documents a parameter as **"object"** or **"target"**, the underlying C++ resolves it using one of three interchangeable forms, tried in this order:

1. **Omitted argument** → defaults to the current selection (`CurrentObject`).
2. **Light userdata handle** → an opaque object reference previously returned by the engine itself (e.g., from `Physics.Raycast`, `Physics.GroundProbe`, or `Physics.SetPositionLock`'s optional 4th argument). You cannot construct one of these yourself in Lua; you can only pass through a value you received from another engine call.
3. **String query** → resolved with [the path syntax above](#53-object-query--path-syntax), relative to the current selection.

```lua
-- All three forms below are valid wherever "target" is documented:
Physics.SetPositionLock(true, false, true)              -- (1) defaults to CurrentObject
Physics.SetPositionLock(true, false, true, enemyHandle)  -- (2) handle from a previous Raycast
SoundObject.Play("Torch_01")                              -- (3) string query
```

---

## 7. Collision & Trigger Callbacks

See [§3.3](#33-special-callback-functions) for the mechanics. Summary of the six supported events:

| Callback | Fired when |
|---|---|
| `OnCollisionEnter(otherName)` | A solid physics collision begins with another (non-trigger) collider. |
| `OnCollisionStay(otherName)` | A solid collision persists across a frame. |
| `OnCollisionExit(otherName)` | A solid collision ends. |
| `OnTriggerEnter(otherName)` | Another collider begins overlapping a trigger volume. |
| `OnTriggerStay(otherName)` | The overlap persists. |
| `OnTriggerExit(otherName)` | The overlap ends. |

Triggers themselves are created/configured via the [`Trigger` table](#86-trigger); their `scriptName` field is the script that will receive the `OnTrigger*` callbacks when something enters/exits their volume.

---

## 8. API Reference

Unless stated otherwise, functions that don't explicitly document a return value return **nothing** (0 Lua results). Optional parameters and their defaults are noted in brackets.

### 8.1 `Input`

Polls the live input state captured by the engine this frame.

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `Input.GetKey(keyName)` | `keyName: string` | `bool` | `true` while the key is held down. Key names are resolved through the engine's key-code table (e.g. `"W"`, `"Space"`, `"LeftShift"` — use your platform's `KeyCodes` list). Unknown names resolve to an invalid code and always return `false`. |
| `Input.GetKeyDown(keyName)` | `keyName: string` | `bool` | `true` only on the frame the key transitions from up to down. |
| `Input.GetKeyUp(keyName)` | `keyName: string` | `bool` | `true` only on the frame the key transitions from down to up. |
| `Input.GetMouseButton(buttonIndex)` | `buttonIndex: integer` | `bool` | Held-state for a mouse button, using the same underlying key-state table as `GetKey`. |
| `Input.GetMouseDelta()` | — | `dx, dy : number, number` | Raw mouse movement delta for the frame. |

```lua
if Input.GetKey("W") then
    Object.Move(0, 0, 5 * Time.deltaTime)  -- forward, local space
end

local dx, dy = Input.GetMouseDelta()
Object.Rotate(0, dx * 0.1, 0)
```

### 8.2 `Object`

The largest table — transform, hierarchy, lifecycle, and animation control for the currently selected object.

#### Transform

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `Object.Move(dx, dy, dz)` | numbers | — | Moves in **local space** relative to the object's current yaw/pitch: `dx` = right, `dy` = world-up, `dz` = forward. |
| `Object.GetPosition()` | — | `x, y, z` | World-space position. |
| `Object.SetPosition(x, y, z)` | numbers | — | Sets world-space position directly. |
| `Object.GetLocalPosition()` | — | `x, y, z` | If parented, returns the position **relative to the parent** (in the parent's local rotated space); otherwise identical to `GetPosition`. |
| `Object.SetLocalPosition(x, y, z)` | numbers | — | If parented, sets the parent-relative offset and recomputes world position from the parent's current transform; otherwise sets world position directly. |
| `Object.Rotate(dx, dy, dz)` | numbers (degrees) | — | Adds to the current Euler rotation. |
| `Object.GetRotation()` | — | `x, y, z` (degrees) | Current Euler rotation. |
| `Object.SetRotation(x, y, z)` | numbers (degrees) | — | Sets Euler rotation directly. |
| `Object.GetScale()` | — | `x, y, z` | |
| `Object.SetScale(x, y, z)` | numbers | — | |
| `Object.GetVelocity()` | — | `x, y, z` | Current physics velocity. |
| `Object.SetVelocity(x, y, z)` | numbers | — | Directly overwrites physics velocity (useful for jumps, knockback, etc.). |

#### Identity & hierarchy

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `Object.GetName()` | — | `string` | |
| `Object.GetType()` | — | `string` | One of the [object type strings](#9-object-types). |
| `Object.SetParent(parentQuery)` | `string` (query, see §5.3) | `bool` | Parents the current object under the resolved target and preserves world position by recomputing the parent-relative offset. Fails (`false`) if the target can't be resolved or is the object itself. |
| `Object.RemoveParent()` | — | `bool` | Un-parents the object (keeps its current world transform). |
| `Object.GetParent()` | — | `string` or nothing | Name of the parent object, or no result if unparented. |
| `Object.GetChildCount()` | — | `integer` | Number of **direct** children. |
| `Object.GetChildName(childIndex)` | `integer` (0-based) | `string` or nothing | Name of the Nth direct child. |

#### Selection (see [§5](#5-object-context--selection) for full semantics)

| Function |
|---|
| `Object.Select(query)` → `bool` |
| `Object.SelectChild(query)` → `bool` |
| `Object.SelectParent()` → `bool` |
| `Object.Push()` |
| `Object.Pop()` → `bool` |

#### Lifecycle (deferred — applied after all scripts finish this frame)

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `Object.Spawn([source], [newName])` | `source`: handle, string query, or omitted (defaults to current selection); `newName`: string | `bool` | Queues a clone of `source`. If `source` is a string that fails to resolve to an existing object, it's instead used as the **requested name** for the clone of the current selection. Particle systems are deep-cloned with a new unique particle-system name. Returns `false` immediately if the source object can't be found — the queued operation itself always succeeds once queued. |
| `Object.Destroy([target])` | handle, string query, or omitted | `bool` | Queues destruction of `target` (defaults to current selection). Destroying an object also destroys its associated particle system (if any), reparents any children to "no parent" while preserving relative offsets get reset, and fixes up the active main-camera index if needed. |

```lua
if Input.GetKeyDown("E") then
    Object.Spawn(nil, "Bullet_" .. tostring(os.clock()))
end

function OnCollisionEnter(otherName)
    if otherName == "Player" then
        Object.Destroy() -- destroy self
    end
end
```

#### Animation

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `Object.PlayAnimation([animName], [loop=true])` | string, bool | — | With no arguments, simply resumes the currently assigned clip if one exists. With `animName`, switches immediately to that clip from time `0`, no blending. |
| `Object.PauseAnimation()` | — | — | Freezes playback at the current time. |
| `Object.StopAnimation()` | — | — | Stops playback and resets time to `0`, clearing any in-progress blend. |
| `Object.BlendToAnimation(animName, blendDuration, [loop=true], [restart=true])` | string, number, bool, bool | — | Cross-fades from whatever is currently playing into `animName` over `blendDuration` seconds. If `blendDuration <= 0`, or nothing is currently playing, or the target is already playing, behaves like an instant switch. |
| `Object.SwitchAnimation(animName, [blendDuration=0.24], [loop=true], [phaseSync=true], [restart=false])` | string, number, bool, bool, bool | — | Like `BlendToAnimation`, but with **phase synchronization**: when `phaseSync` is true and `restart` is false, the new clip starts at the normalized-time-equivalent point of the clip it's replacing (useful for seamless locomotion blends, e.g. walk→run). |
| `Object.GetAnimationCount()` | — | `integer` | Number of animation clips on the object's mesh. |
| `Object.GetAnimationName(index)` | `integer` (0-based) | `string` | |
| `Object.GetCurrentAnimation()` | — | `string` | Name of the currently active clip (empty string if none). |
| `Object.GetAnimationSpeed()` / `Object.SetAnimationSpeed(speed)` | — / number | `number` / — | Playback speed multiplier (clamped ≥ 0). |
| `Object.GetAnimationTime()` / `Object.SetAnimationTime(t)` | — / number | `number` / — | Current playback time in seconds (clamped ≥ 0 on set). |
| `Object.GetBonePosition(boneName)` | `string` | `x, y, z` or nothing | Sampled bone position from the currently playing animation, in the animation's local space. Returns nothing if there's no active animation or the bone isn't found. |
| `Object.GetBoneRotation(boneName)` | `string` | `x, y, z` or nothing | Sampled bone rotation (Euler degrees), same conditions as above. |

> Animation calls that don't explicitly target another object always act on the **skinned mesh associated with the current object** — for rigged characters composed of a root + mesh child, the engine resolves the correct underlying mesh automatically; you don't need to manually select the mesh child.

### 8.3 `World`

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `World.Select(query)` | `string` | `bool` | Selects an object using an absolute/global lookup (no relative context). Equivalent to calling `Object.Select` with no current object context. |

### 8.4 `Scene`

Scene management and time-of-day/nav-mesh globals live partly here, partly in dedicated tables (`WorldTime`, `NavMesh`).

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `Scene.GetObjectPosition(query)` | `string` | `x, y, z` or nothing | Looks up any object **by global query** (no relative context) and returns its position without changing the current selection. |
| `Scene.GetSceneCount()` | — | `integer` | |
| `Scene.GetSceneName(index)` | `integer` (0-based) | `string` | Empty string if out of range. |
| `Scene.GetActiveSceneIndex()` | — | `integer` | |
| `Scene.GetMainSceneIndex()` | — | `integer` | |
| `Scene.SetActiveScene(indexOrName)` | `integer` or `string` | `bool` | |
| `Scene.SetMainScene(indexOrName)` | `integer` or `string` | `bool` | |
| `Scene.FindScene(name)` | `string` | `integer` | Index of the scene, or a negative/invalid index if not found. |
| `Scene.CreateScene([name])` | `string`, optional | `integer` | Index of the newly created scene. |
| `Scene.DeleteScene(indexOrName)` | `integer` or `string` | `bool` | |

### 8.5 `Debug`

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `Debug.Log(message)` | `string` | — | Prints `[LUA] <message>` to stdout and the in-engine log. **No-op if the project's debug features are disabled** (e.g. in a shipping build) — don't rely on it for gameplay logic, only diagnostics. |

### 8.6 `Trigger`

Creates and configures trigger-volume objects (deferred lifecycle for creation).

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `Trigger.Create(name, x, y, z, sizeX, sizeY, sizeZ, [scriptName])` | string, 6×number, optional string | `bool` (always `true` once queued) | Queues creation of a new box-shaped, static, non-gravity trigger object at world position `(x,y,z)` with half/full extents `(sizeX,sizeY,sizeZ)` (absolute value, clamped to a minimum of `0.001`). If `scriptName` is given, that script is assigned to the new trigger and will receive its `OnTrigger*` callbacks. |
| `Trigger.SetEnabled(enabled, [target])` | bool, object ref | `bool` | Marks the target's physics as a trigger and toggles it enabled/disabled. Gives the collider sane defaults (box, size `1`) if it had none configured. |
| `Trigger.SetBox(sizeX, sizeY, sizeZ, [target])` | 3×number, object ref | `bool` | Sets the trigger's box collider size (and matches the object's render scale to it). |
| `Trigger.SetScript(scriptName, [target])` | string, object ref | `bool` | Reassigns which script handles the target's collision/trigger callbacks. |

```lua
-- Somewhere in a setup script:
Trigger.Create("DoorSensor", 10, 0, 5, 1, 2, 1, "DoorTrigger")
```

### 8.7 `JSON`

Simple text file persistence (despite the name, `Save`/`Load` write and read **raw strings** — you are responsible for encoding/decoding JSON yourself, e.g. by building the string manually or with a small hand-written encoder, since no `json` library is bundled).

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `JSON.Save(path, data)` | string, string | `bool` | Overwrites `path` with `data`. Creates parent directories automatically. |
| `JSON.Load(path)` | string | `string` or nothing | Returns the full file contents, or no result if the file doesn't exist / can't be read. |
| `JSON.Exists(path)` | string | `bool` | |
| `JSON.Delete(path)` | string | `bool` | |
| `JSON.MakeDir(path)` | string | `bool` | Creates the directory (and parents) if missing; also returns `true` if it already exists. |

```lua
local save = "coins=" .. tostring(coins) .. ";level=" .. tostring(level)
JSON.Save("saves/profile.txt", save)

local loaded = JSON.Load("saves/profile.txt")
if loaded then
    Debug.Log("Loaded save: " .. loaded)
end
```

### 8.8 `Audio`

Low-level audio clip management, addressed by string **clip IDs** and integer **voice handles**.

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `Audio.Load(id, path)` | string, string | `bool` | Loads/registers a clip under `id`. |
| `Audio.Unload(id)` | string | `bool` | |
| `Audio.Exists(id)` | string | `bool` | Whether `id` is currently loaded. |
| `Audio.Play(id, [volume=1], [pan=0], [loop=false])` | string, number, number, bool | `integer` (voice handle) or `nil` | Returns `nil` if playback failed to start. |
| `Audio.Stop(handle)` | integer | `bool` | |
| `Audio.StopAll()` | — | — | Stops every currently playing voice. |
| `Audio.IsPlaying(handle)` | integer | `bool` | |
| `Audio.SetVoiceVolume(handle, volume)` | integer, number | `bool` | |
| `Audio.SetVoicePan(handle, pan)` | integer, number | `bool` | |
| `Audio.SetVoicePitch(handle, pitch)` | integer, number | `bool` | |
| `Audio.SetGlobalVolume(volume)` | number | `bool` | Master volume multiplier. |

### 8.9 `SoundObject`

A higher-level convenience wrapper for **scene objects that carry an audio clip** (either a dedicated `AudioSource` object type, or any object with an `audioClipPath` assigned). Each such object owns at most one active voice handle at a time.

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `SoundObject.Play([target], [volume], [pan], [loop])` | object ref, number, number, bool | `integer` (handle) or `nil` | Auto-loads the object's assigned clip on first use. Omitted `volume`/`pan`/`loop` fall back to the object's own configured defaults. Stops any voice already playing on that object first. |
| `SoundObject.Stop([target])` | object ref | `bool` | |
| `SoundObject.IsPlaying([target])` | object ref | `bool` | Also clears the object's stored handle if the voice has already finished. |
| `SoundObject.SetVolume(volume, [target])` | number, object ref | `bool` | Updates the object's stored default volume and, if a voice is currently playing, applies it live. |

### 8.10 `Message`

A lightweight per-object mailbox system plus a simple global key/value string store, useful for decoupled communication between scripts.

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `Message.Send(targetQuery, eventName, [data=""])` | string, string, string | `bool` | Queues a message for the resolved target object. Returns `false` if the target can't be resolved. |
| `Message.Broadcast(eventName, [data=""])` | string, string | — | Queues the message for **every scripted object** in the scene. |
| `Message.Receive()` | — | `table` (array) | Drains and returns **all pending messages for the current object** as an array of `{ event = string, data = string }` tables. The queue is cleared after reading. If there's no current object, returns an empty table. |
| `Message.SetGlobal(key, value)` | string, string | — | Writes into a scene-wide string key/value store, shared across **all** scripts (survives individual script hot-reloads). |
| `Message.GetGlobal(key)` | string | `string` or `nil` | |

```lua
-- Sender
Message.Send("Boss", "PhaseChange", "2")

-- Receiver (Boss's script, runs every frame)
for _, msg in ipairs(Message.Receive()) do
    if msg.event == "PhaseChange" then
        Debug.Log("Entering phase " .. msg.data)
    end
end
```

### 8.11 `UI`

Two distinct sub-systems live under `UI`: **screen-space UI elements** (defined elsewhere, e.g. in an editor UI scene) which you can drive from script, and **world-space UI elements** which scripts create and fully own at runtime.

#### Screen-space UI scenes & elements

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `UI.ShowScene(name)` | string | — | Shows a named UI scene/layout. |
| `UI.HideScene(name)` | string | — | |
| `UI.SetSceneVisible(name, visible)` | string, bool | — | |
| `UI.SetText(elementName, text)` | string, string | — | |
| `UI.SetVisible(elementName, visible)` | string, bool | — | |
| `UI.SetColor(elementName, r, g, b, a)` | string, 4×number | — | Foreground/text color. |
| `UI.SetBackgroundColor(elementName, r, g, b, a)` | string, 4×number | — | |
| `UI.SetBorderColor(elementName, r, g, b, a)` | string, 4×number | — | |
| `UI.SetHoverColor(elementName, r, g, b, a)` | string, 4×number | — | Color applied on mouse-hover. |
| `UI.SetActiveColor(elementName, r, g, b, a)` | string, 4×number | — | Color applied while pressed/active. |
| `UI.SetImage(elementName, imagePath)` | string, string | — | Also clears any assigned SVG content and forces a texture reload. |
| `UI.SetRounding(elementName, rounding)` | string, number | — | Corner radius. |
| `UI.SetBorderSize(elementName, size)` | string, number | — | |
| `UI.SetFontSize(elementName, size)` | string, number | — | |

All of the functions above are no-ops if `elementName` doesn't resolve to an existing UI element (defined in the scene's UI layout).

#### World-space UI (floating text/labels/panels anchored in 3D space)

These fully create-on-demand: calling any `SetWorld*`/`World*` function with a new `id` implicitly creates that element (upsert semantics), so it's always safe to call every frame.

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `UI.WorldText(id, text, x, y, z, [fontScale], [r, g, b, a], [offsetX, offsetY])` | string, string, 3×number, optional number, optional 4×number, optional 2×number | — | Creates/updates a simple floating text label at world position `(x,y,z)`. Safe to call every frame; sets the element visible. |
| `UI.RemoveWorldElement(id)` | string | `bool` | |
| `UI.SetWorldVisible(id, visible)` | string, bool | `bool` | |
| `UI.SetWorldPosition(id, x, y, z, [offsetX, offsetY])` | string, 3×number, optional 2×number | — | `offsetX/Y` is an additional **pixel-space** screen offset applied after 3D→2D projection. |
| `UI.SetWorldText(id, text)` | string, string | — | |
| `UI.SetWorldStyle(id, fontScale, textR,textG,textB,textA, bgR,bgG,bgB,bgA, borderR,borderG,borderB,borderA, width, height, rounding, borderSize, paddingX, paddingY)` | string + 19 numbers | — | Full styling in one call — positional, order-sensitive. Prefer `UI.World` (below) for readability. |
| `UI.World(id, options)` | string, table | — | **Recommended** friendly form. `options` is a table that may contain any of: `text` (string), `position` (`{x,y,z}`), `offset` (`{x,y}` pixels), `color` (`{r,g,b,a}`), `background` (`{r,g,b,a}`), `border` (`{r,g,b,a}`), `size` (`{w,h}`), `padding` (`{x,y}`), `fontSize` (number), `rounding` (number), `borderSize` (number), `visible` (bool, defaults to `true`). Only the fields you supply are updated; everything else keeps its previous value (or the element's defaults if newly created). |

```lua
UI.World("pickup_prompt", {
    text = "[E] Pick up",
    position = { 0, 1.2, 0 },
    color = { 1, 1, 1, 1 },
    background = { 0, 0, 0, 0.7 },
    size = { 180, 44 },
    fontSize = 1.2,
})
```

### 8.12 `Render`

Frame-level post-processing controls (motion blur & static/directional blur).

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `Render.SetMotionBlurStrength(strength)` | number | — | |
| `Render.SetMotionBlurMode(mode)` | integer | — | |
| `Render.SetMotionBlurSamples(samples)` | integer | — | |
| `Render.GetMotionBlur()` | — | `strength, mode, samples` | |
| `Render.AddMotionBlur(deltaStrength, [deltaSamples=0], [mode])` | number, integer, optional integer | `strength, mode, samples` | Additively adjusts current values (clamped: strength `[0,4]`, samples `[1,32]`). |
| `Render.BlendMotionBlur(targetStrength, targetSamples, lerpFactor, [mode])` | number, integer, number `[0,1]`, optional integer | `strength, mode, samples` | Smoothly interpolates current values toward the targets by `lerpFactor` (call every frame with a small factor for a smooth transition). |
| `Render.ResetMotionBlur([strength=0.35], [mode=1], [samples=16])` | optional numbers | — | |
| `Render.SetStaticBlur(enabled, [strength], [mode], [dirX, dirY], [centerX, centerY])` | bool + optional numbers | — | Directional/zoom blur effect. Omitted numeric args keep their current values. |
| `Render.GetStaticBlur()` | — | `enabled, strength, mode, dirX, dirY, centerX, centerY` | |
| `Render.SetStaticBlurDirection(x, y)` | 2×number | — | |
| `Render.SetStaticBlurCenter(x, y)` | 2×number | — | |

### 8.13 `Graphics`

Global rendering quality settings.

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `Graphics.SetTextureQuality(level)` | integer `[0,3]` | — | Clamped to range. |
| `Graphics.SetShadowQuality(level)` | integer `[0,3]` | — | |
| `Graphics.SetReflectionQuality(level)` | integer `[0,3]` | — | |
| `Graphics.SetResolutionScale(scale)` | number `[0.5,1.5]` | — | |
| `Graphics.SetReflectionIntensity(intensity)` | number `[0,2]` | — | |
| `Graphics.SetResolution(width, height)` | 2×integer | `bool` | Rejects (`false`) sizes below `640×360`; on success, resizes the actual window. |
| `Graphics.GetSettings()` | — | `table` | Returns a table with fields: `textureQuality`, `shadowQuality`, `reflectionQuality`, `resolutionScale`, `reflectionIntensity`, `renderWidth`, `renderHeight`. |
| `Graphics.ApplyPreset(preset)` | integer `[0,3]` | — | Applies a bundled quality preset (0 = Low … 3 = Ultra), setting texture/shadow/reflection quality, resolution scale, reflection intensity, and motion-blur defaults together. |

### 8.14 `WorldTime`

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `WorldTime.SetTimeOfDay(hours)` | number | — | Clamped to `[0, 24)`; values above `24` wrap via modulo. |
| `WorldTime.GetTimeOfDay()` | — | `number` | Current time of day in hours. |

> Note: a plain `Time` global table is also set every frame with just a `deltaTime` field (see [§8.21](#821-time)) — `WorldTime` is the separate, persistent day/night-cycle clock.

### 8.15 `NavMesh`

A lightweight, script-driven navigation mesh used for pathfinding queries (grid/obstacle based, not a fully baked polygon mesh).

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `NavMesh.Build()` | — | — | Marks the nav-mesh as "built" using the currently active settings (`enabled` flag). |
| `NavMesh.IsBuilt()` | — | `bool` | |
| `NavMesh.FindPath(startX,startY,startZ, endX,endY,endZ, [drawDebug=true])` | 6×number, optional bool | array-of-`{x,y,z}` tables, or nothing | Computes a path avoiding obstacles per the active nav-mesh settings, for the **currently selected object**. If `drawDebug` is true and debug features are enabled, also stores a debug path visualization on the object. Returns nothing if no path could be found. |
| `NavMesh.SetSettings(enabled, minX,minY,minZ, maxX,maxY,maxZ, cellSize, agentRadius, useStaticObstacles)` | bool + 6×number + 2×number + bool | — | Configures the pathing volume bounds, grid cell size, agent radius (for obstacle inflation), and whether static scene geometry blocks paths. Also invalidates the "built" flag. |
| `NavMesh.GetSettings()` | — | `table` | Returns `{ enabled, minBounds={x,y,z}, maxBounds={x,y,z}, cellSize, agentRadius, useStaticObstacles }`. |

```lua
NavMesh.SetSettings(true, -50,0,-50, 50,10,50, 1.0, 0.5, true)
NavMesh.Build()

local path = NavMesh.FindPath(0,0,0, 20,0,15)
if path then
    for i, point in ipairs(path) do
        Debug.Log(string.format("Waypoint %d: %.1f, %.1f, %.1f", i, point[1], point[2], point[3]))
    end
end
```

### 8.16 `ManualPath`

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `ManualPath.GetPath()` | — | array-of-`{x,y,z}` tables, or nothing | Returns any manually-authored waypoint path assigned to the current object (e.g. an editor-placed patrol route), or nothing if none / no current object. |

### 8.17 `AI` (Behavior Trees)

A minimal, script-drivable behavior-tree runtime for simple AI logic. See [§10](#10-behavior-tree-ai-deep-dive) for the execution model.

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `AI.CreateTree()` | — | `integer` (`treeId`) | Creates a new, empty behavior tree and returns its handle. Typically called once (e.g. in a one-time setup section) and the ID cached via `Message.SetGlobal` or a persisted script variable pattern. |
| `AI.CreateSequence(treeId)` | integer | `integer` (`nodeId`) | Adds a **Sequence** composite node (runs children in order; fails fast). |
| `AI.CreateSelector(treeId)` | integer | `integer` (`nodeId`) | Adds a **Selector** composite node (runs children in order; succeeds fast). |
| `AI.CreateWait(treeId, duration)` | integer, number | `integer` (`nodeId`) | Leaf node: succeeds after `duration` seconds of being ticked. |
| `AI.CreateMoveTo(treeId, x, y, z, speed, stopDistance)` | integer, 3×number, number, number | `integer` (`nodeId`) | Leaf node: paths the ticking object toward `(x,y,z)` (via the same obstacle-avoiding path builder as `NavMesh.FindPath`) at `speed` units/sec, succeeding once within `stopDistance`. |
| `AI.CreateLog(treeId, message)` | integer, string | `integer` (`nodeId`) | Leaf node: logs `message` and immediately succeeds. |
| `AI.AddChild(treeId, parentNodeId, childNodeId)` | 3×integer | — | Appends `childNodeId` to a Sequence/Selector's child list. |
| `AI.SetRoot(treeId, nodeId)` | 2×integer | — | Designates the tree's entry-point node. |
| `AI.Tick(treeId, deltaTime)` | integer, number | `string`: `"Running"`, `"Success"`, or `"Failure"` | Advances the tree by one tick for the **currently selected object**. Call this every frame from the owning object's script. Does nothing (no result) if there's no current object. |
| `AI.Reset(treeId)` | integer | — | Clears the current object's saved runtime progress in the tree (child indices, wait timers, in-progress paths), so the next `Tick` starts the tree from the beginning. |

### 8.18 `Cursor`

| Function | Effect |
|---|---|
| `Cursor.Lock()` | Locks the mouse cursor (typical for first/third-person camera control). |
| `Cursor.Unlock()` | Releases the lock. |
| `Cursor.Hide()` | Hides the OS cursor. |

### 8.19 `ParticleSystem`

Controls a `ParticleSystem`-type object (or one attached to another object). Every function's final/`ps` argument follows the standard [object reference rules](#6-object-references): a name-string query, an omitted argument (defaults to the current selection's attached particle system), etc. **Note the argument order** — value parameters generally come **before** the target reference.

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `ParticleSystem.Start([target])` | object ref | `bool` | Begins/resumes emission. |
| `ParticleSystem.Stop([target])` | object ref | `bool` | Halts emission (existing particles continue simulating until they expire). |
| `ParticleSystem.Restart([target])` | object ref | `bool` | Clears all current particles and starts emitting fresh. |
| `ParticleSystem.IsPlaying([target])` | object ref | `bool` | |
| `ParticleSystem.SetSpawnRate(rate, [target])` | number, object ref | `bool` | Particles/sec, clamped ≥ 0. |
| `ParticleSystem.GetSpawnRate([target])` | object ref | `number` | |
| `ParticleSystem.SetParticleCount(count, [target])` | integer, object ref | `bool` | Resizes the particle pool (`count` must be > 0). |
| `ParticleSystem.GetParticleCount([target])` | object ref | `integer` | |
| `ParticleSystem.SetForces(x, y, z, [target])` | 3×number, object ref | `bool` | Sets a constant summed force (e.g. gravity/wind) applied to every particle. |
| `ParticleSystem.SetEmitterPosition(x, y, z, [target])` | 3×number, object ref | `bool` | |
| `ParticleSystem.SetFixedLifetime(lifetime, [target])` | number, object ref | `bool` | Disables randomized lifetime; all particles live exactly `lifetime` seconds (clamped ≥ `0.001`). |
| `ParticleSystem.SetLifetimeRange(minLife, maxLife, [target])` | 2×number, object ref | `bool` | Enables randomized lifetime within `[min,max]` (values are auto-sorted/clamped). |
| `ParticleSystem.SetFixedVelocity(x, y, z, [target])` | 3×number, object ref | `bool` | Disables randomized initial velocity; every particle spawns with exactly this velocity. |
| `ParticleSystem.SetVelocityRange(minX,minY,minZ, maxX,maxY,maxZ, [target])` | 6×number, object ref | `bool` | Enables per-axis randomized initial velocity within the given box (min/max auto-sorted per axis). |

```lua
ParticleSystem.SetSpawnRate(50)
ParticleSystem.SetLifetimeRange(0.5, 1.5)
ParticleSystem.SetVelocityRange(-1,2,-1, 1,4,1)
ParticleSystem.SetForces(0, -9.8, 0)
ParticleSystem.Start()
```

### 8.20 `Physics`

| Function | Parameters | Returns | Notes |
|---|---|---|---|
| `Physics.SetPositionLock(lockX, lockY, lockZ, [target])` | 3×bool, object ref | — | Freezes the target's position along the locked axes (useful for 2.5D characters, e.g. lock Z). |
| `Physics.SetRotationLock(lockX, lockY, lockZ, [target])` | 3×bool, object ref | — | Freezes rotation along the locked axes (e.g. lock X/Z on a character so it never tips over). |
| `Physics.Raycast(originX,originY,originZ, dirX,dirY,dirZ, maxDistance, [ignoreTarget])` | 6×number, number, optional object handle | `hit:bool, [handle, hitX,hitY,hitZ, normalX,normalY,normalZ]` | Casts a ray. `dir` should be a direction (not necessarily pre-normalized by the caller, but treat it as a direction vector). `ignoreTarget`, if given, must be a **light-userdata handle** (a string query is not accepted here) to exclude that object from the results. On hit, returns `true` plus the hit object's handle, world-space hit position, and surface normal (8 values total including the leading bool). On miss, returns only `false`. |
| `Physics.GroundProbe(x,y,z, [probeUp=0.75], [probeDistance=2.0], [probeRadius=0.3], [maxSlopeAngle=50], [ignoreTarget])` | 3×number + 4 optional numbers + optional handle | `hit:bool, stable:bool, [handle, hitX,hitY,hitZ, normalX,normalY,normalZ, distance]` | Casts straight down from `(x, y + probeUp, z)` up to `probeDistance`, skipping trigger volumes and NavMesh objects, to find ground. `stable` indicates the hit surface's slope is within `maxSlopeAngle` degrees of flat. Returns 10 values on a genuine geometry hit (or 2 — just `false, false` — on a total miss). |
| `Physics.GetGroundState([target], [maxSlopeAngle=52], [extraDistance=0.18])` | optional object handle first, then optional numbers | `anyHit, grounded, stable, handle, x, y, z, normalX, normalY, normalZ, distanceToGround, maxGroundDistance` (12 values) | A robust, character-controller-oriented ground check: samples multiple points under the target's collider footprint, accounts for the object's current collider shape/size and vertical velocity (to avoid jitter when falling fast or jumping), and reports whether the object should be considered "grounded" this frame. Prefer this over a raw `Raycast`/`GroundProbe` pair for player/AI movement controllers. If omitted, `target` defaults to the current selection. |

```lua
-- Simple grounded check + jump
local anyHit, grounded = Physics.GetGroundState()
if grounded and Input.GetKeyDown("Space") then
    Object.SetVelocity(0, 8, 0)
end
```

```lua
-- Raycast example: shoot forward from the object, ignoring itself
local hit, handle, hx, hy, hz, nx, ny, nz = Physics.Raycast(
    Object.GetPosition(), 0, 0, 1, 50 -- (see note below on multi-return splicing)
)
```
> **Note:** Lua does not auto-splice a multi-return like `Object.GetPosition()` into the middle of another call's argument list unless it's the *last* argument. Capture position into locals first:
```lua
local px, py, pz = Object.GetPosition()
local hit, handle, hx, hy, hz, nx, ny, nz = Physics.Raycast(px, py, pz, 0, 0, 1, 50)
if hit then
    Debug.Log("Hit something at " .. hx .. "," .. hy .. "," .. hz)
end
```

### 8.21 `Time`

Not a persistent table you configure — the engine **replaces** the global `Time` table at the start of every script/callback invocation with a fresh one containing a single field:

| Field | Type | Notes |
|---|---|---|
| `Time.deltaTime` | `number` | Seconds elapsed since the previous frame. Inside `RunSandboxTest`, this is fixed at `0.016`. |

Always read `Time.deltaTime` fresh each frame; don't cache the `Time` table itself across frames, since it is a brand-new table every call.

---

## 9. Object Types

`Object.GetType()` returns one of the following strings:

| String | Meaning |
|---|---|
| `"Cube"` | Primitive box mesh. |
| `"Sphere"` | Primitive sphere mesh. |
| `"Plane"` | Primitive plane mesh. |
| `"Light"` | Light source. |
| `"Camera"` | Camera. |
| `"ParticleSystem"` | Particle emitter — controllable via the [`ParticleSystem` table](#819-particlesystem). |
| `"Mesh"` | Generic/imported mesh (static or skinned). |
| `"NavMesh"` | Nav-mesh volume marker (excluded from ground probes). |
| `"SceneInstance"` | An embedded instance of another scene. |
| `"Trigger"` | A trigger volume created via [`Trigger.Create`](#86-trigger) or placed in the editor. |
| `"AudioSource"` | A dedicated audio-emitting object, usable with [`SoundObject`](#89-soundobject). |
| `"Unknown"` | Fallback for unrecognized/uninitialized types. |

---

## 10. Behavior Tree AI Deep Dive

The `AI` table exposes a tiny, purpose-built behavior tree engine. Trees are built once (usually in a one-time setup block guarded by a flag or built externally and referenced by ID), then **ticked every frame** from the owning object's per-frame script body.

### 10.1 Node types & tick semantics

| Node | Created by | Tick behavior |
|---|---|---|
| **Sequence** | `AI.CreateSequence` | Ticks children **in order**, left to right, remembering which child it's currently on between frames. If a child returns `Running`, the sequence also returns `Running` (staying on that child next tick). If a child returns `Failure`, the sequence resets to its first child and returns `Failure`. Only once **all** children have returned `Success` in turn does the sequence return `Success` (and reset for next time). |
| **Selector** | `AI.CreateSelector` | Same iteration model, but success/failure semantics are inverted: the first child to return `Success` makes the whole selector `Success` (and resets); a child returning `Failure` moves on to the next child; if **all** children fail, the selector returns `Failure` (and resets). |
| **Wait** | `AI.CreateWait(treeId, duration)` | Counts down `duration` seconds across ticks; returns `Running` until the timer elapses, then `Success`. |
| **MoveTo** | `AI.CreateMoveTo(treeId, x,y,z, speed, stopDistance)` | Builds an obstacle-avoiding path to `(x,y,z)` on first tick (same pathfinder as `NavMesh.FindPath`), then walks the ticking object along it at `speed` units/sec, returning `Running` until within `stopDistance` of the final target (or immediately `Success` if already close enough), and `Failure` if no path could be built. |
| **Log** | `AI.CreateLog(treeId, message)` | Logs `message` via the same channel as `Debug.Log` and immediately returns `Success` — handy as a debugging leaf while building a tree. |

Runtime progress (which child a Sequence/Selector is on, remaining Wait time, in-progress MoveTo path) is tracked **per tree, per ticking object** — the same tree definition can be shared and ticked independently by multiple objects/instances without interfering with each other. Use `AI.Reset(treeId)` to explicitly restart an object's progress through the tree.

### 10.2 Example: patrol-then-chase

```lua
-- One-time setup: build the tree the first time this script runs.
-- (Because sandboxes persist across frames until the script text changes,
--  a top-level local flag survives between frames — but NOT across a hot reload.)
if TreeId == nil then
    TreeId = AI.CreateTree()

    local root = AI.CreateSelector(TreeId)
    AI.SetRoot(TreeId, root)

    -- Branch 1: patrol
    local patrolSeq = AI.CreateSequence(TreeId)
    local moveA = AI.CreateMoveTo(TreeId, 0, 0, 0, 2.0, 0.5)
    local waitA = AI.CreateWait(TreeId, 1.0)
    local moveB = AI.CreateMoveTo(TreeId, 10, 0, 0, 2.0, 0.5)
    local waitB = AI.CreateWait(TreeId, 1.0)
    AI.AddChild(TreeId, patrolSeq, moveA)
    AI.AddChild(TreeId, patrolSeq, waitA)
    AI.AddChild(TreeId, patrolSeq, moveB)
    AI.AddChild(TreeId, patrolSeq, waitB)
    AI.AddChild(TreeId, root, patrolSeq)
end

AI.Tick(TreeId, Time.deltaTime)
```

---

## 11. Debugging & Error Handling

- **Script errors never crash the engine.** Every top-level run and every collision/trigger callback is wrapped in a protected call. On error, the engine logs:
  `Lua Error (<scriptKey>): <message>`
  to stdout/console and simply skips the rest of that invocation — the object's previous frame state is left as-is.
- **Sandbox exhaustion looks like a normal Lua error.** If you hit the instruction limit or memory limit, you'll see a message such as `Lua sandbox instruction limit exceeded` (or a `not enough memory` style error from Lua's own allocator failure path) via the same error channel. This usually indicates an infinite/runaway loop or an unbounded table growing every frame.
- **`Debug.Log` is gated by debug features.** If your logs aren't appearing, first confirm the project's debug/editor mode is enabled — `Debug.Log` silently no-ops otherwise.
- **Editing a script fully resets its Lua state.** Any top-level `local`/global variables you were using as "instance state" (counters, flags, cached IDs) are wiped the moment the script's source text changes. Persist anything that must survive edits or scene reloads via `Message.SetGlobal`/`JSON.Save`, not top-level Lua variables.
- **`CurrentObject` can be `nil`.** This happens during `RunSandboxTest` and potentially in other edge cases. Guard object-dependent logic defensively, or rely on the many API functions that already return sensible "nothing happened" defaults when there's no current object.

---

## 12. Performance & Best Practices

1. **Respect the instruction budget.** ~200,000 VM instructions per script call is generous for typical gameplay logic but not for heavy per-frame numeric work (e.g. brute-force nearest-neighbor searches over hundreds of objects). Spread expensive work across multiple frames, or move it into engine-side systems instead.
2. **Prefer `Object.Push`/`Object.Pop` over leaving the selection changed.** Forgetting to restore the selection after `Object.Select`/`SelectChild`/`SelectParent` is a common source of "my script randomly stopped moving the right object" bugs, since the *rest of that same invocation* (including any callback re-entrancy) will keep operating on the wrong object.
3. **World-space UI (`UI.World`/`UI.WorldText`) is upsert-safe** — call it every frame without worrying about "did I already create this element."
4. **Avoid relying on top-level Lua state surviving edits.** Use `Message.SetGlobal` (in-memory, scene lifetime) or `JSON.Save`/`JSON.Load` (disk, persists across sessions) for anything that must outlive a hot reload.
5. **Batch expensive queries.** `Physics.GetGroundState` already does multi-sample probing internally and is tuned for character controllers — prefer it over hand-rolled multi-raycast loops.
6. **Behavior trees are cheap to tick but not free to rebuild.** Build tree structure once (guard with a one-time-setup flag) rather than recreating nodes every frame.
7. **String-based object queries walk the hierarchy/scene each call.** For a target you'll reference many times in a single frame, resolve it once into a light-userdata handle where the API supports it (e.g., from `Physics.Raycast`) rather than repeatedly re-querying by name — though note most `Object.*` convenience functions don't expose raw handles back to you except through these physics queries.

---

## 13. Complete Example Scripts

### 13.1 Third-person-style character controller

```lua
local MoveSpeed = 4.0
local JumpVelocity = 7.0
local TurnSpeed = 120.0

-- Turning
if Input.GetKey("A") then Object.Rotate(0, -TurnSpeed * Time.deltaTime, 0) end
if Input.GetKey("D") then Object.Rotate(0,  TurnSpeed * Time.deltaTime, 0) end

-- Forward/back movement (local space)
local move = 0.0
if Input.GetKey("W") then move = move + 1.0 end
if Input.GetKey("S") then move = move - 1.0 end
Object.Move(0, 0, move * MoveSpeed * Time.deltaTime)

-- Grounded check + jump
local anyHit, grounded = Physics.GetGroundState()
if grounded and Input.GetKeyDown("Space") then
    local vx, vy, vz = Object.GetVelocity()
    Object.SetVelocity(vx, JumpVelocity, vz)
end

-- Animation state
if move ~= 0 then
    Object.SwitchAnimation("Run", 0.2)
else
    Object.SwitchAnimation("Idle", 0.2)
end
```

### 13.2 Pickup trigger with world-space prompt

```lua
-- Attached to a Trigger object created via Trigger.Create(...)
local PlayerInside = false

function OnTriggerEnter(otherName)
    if otherName == "Player" then
        PlayerInside = true
        UI.World("pickup_prompt", {
            text = "[E] Pick up",
            position = { 0, 1.2, 0 },
            background = { 0, 0, 0, 0.7 },
            color = { 1, 1, 1, 1 },
        })
    end
end

function OnTriggerExit(otherName)
    if otherName == "Player" then
        PlayerInside = false
        UI.SetWorldVisible("pickup_prompt", false)
    end
end

-- Runs every frame
if PlayerInside and Input.GetKeyDown("E") then
    Message.Broadcast("ItemCollected", "Coin")
    Object.Destroy()
end
```

### 13.3 Simple patrolling guard with line-of-sight chase

```lua
if TreeId == nil then
    TreeId = AI.CreateTree()
    local root = AI.CreateSelector(TreeId)
    AI.SetRoot(TreeId, root)

    local patrol = AI.CreateSequence(TreeId)
    AI.AddChild(TreeId, patrol, AI.CreateMoveTo(TreeId, -5,0,0, 1.5, 0.3))
    AI.AddChild(TreeId, patrol, AI.CreateWait(TreeId, 2.0))
    AI.AddChild(TreeId, patrol, AI.CreateMoveTo(TreeId,  5,0,0, 1.5, 0.3))
    AI.AddChild(TreeId, patrol, AI.CreateWait(TreeId, 2.0))
    AI.AddChild(TreeId, root, patrol)
end

local px, py, pz = Object.GetPosition()
local playerX, playerY, playerZ = Scene.GetObjectPosition("Player")

if playerX then
    local dx, dz = playerX - px, playerZ - pz
    local dist = math.sqrt(dx*dx + dz*dz)
    if dist < 6.0 then
        Debug.Log("Guard spotted the player!")
        AI.Reset(TreeId) -- interrupt patrol
        -- (a real implementation would swap to a dedicated "chase" tree/branch here)
    end
end

AI.Tick(TreeId, Time.deltaTime)
```

### 13.4 Persistent save data

```lua
if Input.GetKeyDown("F5") then
    local data = string.format("score=%d;level=%s", Score or 0, CurrentLevel or "1")
    JSON.Save("saves/slot1.txt", data)
    Debug.Log("Game saved.")
end

if Input.GetKeyDown("F9") then
    local raw = JSON.Load("saves/slot1.txt")
    if raw then
        local score = tonumber(raw:match("score=(%d+)"))
        local level = raw:match("level=(%S+)")
        Score = score or 0
        CurrentLevel = level or "1"
        Debug.Log("Loaded save: score=" .. tostring(Score) .. " level=" .. tostring(CurrentLevel))
    else
        Debug.Log("No save file found.")
    end
end
```

---

## 14. Function Index (Alphabetical)

For quick lookup. See [§8](#8-api-reference) for full parameter/return details.

`AI.AddChild` · `AI.CreateLog` · `AI.CreateMoveTo` · `AI.CreateSelector` · `AI.CreateSequence` · `AI.CreateTree` · `AI.CreateWait` · `AI.Reset` · `AI.SetRoot` · `AI.Tick` ·
`Audio.Exists` · `Audio.IsPlaying` · `Audio.Load` · `Audio.Play` · `Audio.SetGlobalVolume` · `Audio.SetVoicePan` · `Audio.SetVoicePitch` · `Audio.SetVoiceVolume` · `Audio.Stop` · `Audio.StopAll` · `Audio.Unload` ·
`Cursor.Hide` · `Cursor.Lock` · `Cursor.Unlock` ·
`Debug.Log` ·
`Graphics.ApplyPreset` · `Graphics.GetSettings` · `Graphics.SetReflectionIntensity` · `Graphics.SetReflectionQuality` · `Graphics.SetResolution` · `Graphics.SetResolutionScale` · `Graphics.SetShadowQuality` · `Graphics.SetTextureQuality` ·
`Input.GetKey` · `Input.GetKeyDown` · `Input.GetKeyUp` · `Input.GetMouseButton` · `Input.GetMouseDelta` ·
`JSON.Delete` · `JSON.Exists` · `JSON.Load` · `JSON.MakeDir` · `JSON.Save` ·
`ManualPath.GetPath` ·
`Message.Broadcast` · `Message.GetGlobal` · `Message.Receive` · `Message.Send` · `Message.SetGlobal` ·
`NavMesh.Build` · `NavMesh.FindPath` · `NavMesh.GetSettings` · `NavMesh.IsBuilt` · `NavMesh.SetSettings` ·
`Object.BlendToAnimation` · `Object.Destroy` · `Object.GetAnimationCount` · `Object.GetAnimationName` · `Object.GetAnimationSpeed` · `Object.GetAnimationTime` · `Object.GetBonePosition` · `Object.GetBoneRotation` · `Object.GetChildCount` · `Object.GetChildName` · `Object.GetCurrentAnimation` · `Object.GetLocalPosition` · `Object.GetName` · `Object.GetParent` · `Object.GetPosition` · `Object.GetRotation` · `Object.GetScale` · `Object.GetType` · `Object.GetVelocity` · `Object.Move` · `Object.PauseAnimation` · `Object.PlayAnimation` · `Object.Pop` · `Object.Push` · `Object.RemoveParent` · `Object.Rotate` · `Object.Select` · `Object.SelectChild` · `Object.SelectParent` · `Object.SetAnimationSpeed` · `Object.SetAnimationTime` · `Object.SetLocalPosition` · `Object.SetParent` · `Object.SetPosition` · `Object.SetRotation` · `Object.SetScale` · `Object.SetVelocity` · `Object.Spawn` · `Object.StopAnimation` · `Object.SwitchAnimation` ·
`ParticleSystem.GetParticleCount` · `ParticleSystem.GetSpawnRate` · `ParticleSystem.IsPlaying` · `ParticleSystem.Restart` · `ParticleSystem.SetEmitterPosition` · `ParticleSystem.SetFixedLifetime` · `ParticleSystem.SetFixedVelocity` · `ParticleSystem.SetForces` · `ParticleSystem.SetLifetimeRange` · `ParticleSystem.SetParticleCount` · `ParticleSystem.SetSpawnRate` · `ParticleSystem.SetVelocityRange` · `ParticleSystem.Start` · `ParticleSystem.Stop` ·
`Physics.GetGroundState` · `Physics.GroundProbe` · `Physics.Raycast` · `Physics.SetPositionLock` · `Physics.SetRotationLock` ·
`Render.AddMotionBlur` · `Render.BlendMotionBlur` · `Render.GetMotionBlur` · `Render.GetStaticBlur` · `Render.ResetMotionBlur` · `Render.SetMotionBlurMode` · `Render.SetMotionBlurSamples` · `Render.SetMotionBlurStrength` · `Render.SetStaticBlur` · `Render.SetStaticBlurCenter` · `Render.SetStaticBlurDirection` ·
`Scene.CreateScene` · `Scene.DeleteScene` · `Scene.FindScene` · `Scene.GetActiveSceneIndex` · `Scene.GetMainSceneIndex` · `Scene.GetObjectPosition` · `Scene.GetSceneCount` · `Scene.GetSceneName` · `Scene.SetActiveScene` · `Scene.SetMainScene` ·
`SoundObject.IsPlaying` · `SoundObject.Play` · `SoundObject.SetVolume` · `SoundObject.Stop` ·
`Trigger.Create` · `Trigger.SetBox` · `Trigger.SetEnabled` · `Trigger.SetScript` ·
`UI.HideScene` · `UI.RemoveWorldElement` · `UI.SetActiveColor` · `UI.SetBackgroundColor` · `UI.SetBorderColor` · `UI.SetBorderSize` · `UI.SetColor` · `UI.SetFontSize` · `UI.SetHoverColor` · `UI.SetImage` · `UI.SetRounding` · `UI.SetSceneVisible` · `UI.SetText` · `UI.SetVisible` · `UI.SetWorldPosition` · `UI.SetWorldStyle` · `UI.SetWorldText` · `UI.SetWorldVisible` · `UI.ShowScene` · `UI.World` · `UI.WorldText` ·
`WorldTime.GetTimeOfDay` · `WorldTime.SetTimeOfDay` ·
`World.Select`

---

