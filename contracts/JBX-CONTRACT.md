<!-- SNAPSHOT -- read this first -------------------------------------------
This is a published copy of a file that ships inside JuiceBox - LLM Integration,
captured at vocabulary version pro-106. It is here so that language models and
coding agents can read the format without owning the asset.

The copy installed in your Unity project is the authoritative one, and it is
regenerated to match the installed build. If you have the add-on, read it there
(Tools > JuiceBox > LLM Hooks > Setup...) rather than this file. Confirm the
vocabulary your build actually carries with:

    unity command juicebox_read --content 'vocabulary'
--------------------------------------------------------------------------- -->

# JBX Contract — authoring JuiceBox animations from text

This file is the complete contract for agents authoring JuiceBox animations. Read it fully; it is sized to fit in context. Verify names against the vocabulary in this file when in doubt — every name in a `.jbx` file comes from a closed vocabulary and unknown names are validation errors.

## For Unity's in-Editor AI Assistant (RunCommand)

If you are Unity's built-in Assistant (Ask/Plan/Agent): if you can see the dedicated
`JuiceBox.Read` / `JuiceBox.WriteScene` / `JuiceBox.WriteShared` tools, use them. If you
CANNOT, do NOT grep the project or parse `.asset` files by hand — that bypasses validation,
tokens, and undo and can corrupt animations. Instead call the SAME three verbs on the
governed facade `JuiceBox.LLMHooks.JbxOps` from a RunCommand C# script. Every method
takes/returns strings (JSON in, JSON report out) and never throws; a report is
`{ "status": "pass" | "fail", ... }` with, on failure, an `errors` array of
`{ code, path, message, suggestion, valid }` (`valid` = the whole set an unknown
name was checked against; `suggestion` = a near-spelling guess). Use a read-only command for the read call.

- `JbxOps.Read(string target)` — the one read verb. Empty/`null`: an OVERVIEW (every
  definition asset + every JuiceBoxAnimation in the OPEN scenes, with objectId and
  definition-vs-inline). `"vocabulary"`: the closed name vocabulary (effect kinds, fields,
  easings, modes) — read before authoring; invented names are rejected. An ADDRESS (a
  definition asset path, or an inline animation's `objectId` / `/scene/path`): the full DUMP
  of that one animation, including the CAS tokens a patch needs. The dump is ground truth.
- `JbxOps.WriteScene(string content)` — mutates only the OPEN scene (marks it dirty, never
  saves). A `.jbxpatch` whose `asset` is an inline address MODIFIES that inline animation; an
  insert (root `jbxInsertVersion`) PLACES an animation on an existing scene GameObject; an
  ownership payload with `op` `copy` / `makeUnique` / `linkDefinition`.
- `JbxOps.WriteShared(string content)` — writes shared definition ASSETS (a project-wide
  change; every instance follows). A `.jbx` (root `jbxVersion`, with `outputAsset`) CREATES a
  definition; a `.jbxpatch` (root `jbxPatchVersion`) whose `asset` is an asset path MODIFIES
  one; an ownership payload (root `jbxOwnershipVersion`) with `op` `extractDefinition`
  (inline → new asset) or `linkDefinition`.

Each write verb rejects a payload whose blast radius belongs to the other and points you at
the right one. Applies register an editor Undo step, so the human can revert your edit.

The editing loop: `Read(address)` → author a `.jbxpatch` citing tokens **from that dump** →
`WriteScene` / `WriteShared` → read the returned report. Report what the report actually
says; never claim success without a `pass`.

**The headless path.** No agent tools, no bridge, no editor open. There is no import MENU item — creating animations by hand is the `.jbx` file plus `-executeMethod`, and everything interactive lives in `Tools > JuiceBox > LLM Hooks > Setup...`.

1. Write a `.jbx` file (schema below).
2. Apply it: `Unity -batchmode -projectPath <path> -executeMethod JuiceBox.LLMHooks.JbxImport.Run -jbxFile <file> [-jbxOut Assets/Path.asset] [-jbxOverwrite]` (only when the project is not open in an editor; Unity locks projects to one instance). Import validates the file, and on success creates a `JuiceBoxAnimationDefinition` asset; either way it writes an **apply report** to `<file>.apply.json` — status, plus every finding with a code, JSON path, message, and (for unknown names) a `suggestion` and the `valid` set. Read it after every run: it is the feedback channel for fixing and retrying. It is regenerated on each run and safe to delete. (Humans see the same findings in the Unity Console.)
3. The asset is now the source of truth. Never regenerate and re-import to change it — modifications go through the `.jbxpatch` flow, documented in `JBXPATCH-CONTRACT.md` beside this file.

**The tools (preferred).** If you can see `juicebox_read` / `juicebox_writescene` / `juicebox_writeshared`, use them: they carry the payloads described in this document with no files in between, and no human clicking menus. Three registrations publish that same trio under those same names — the Unity CLI (`unity command juicebox_read`, and the MCP tools `unity mcp configure` writes into your client's config), Unity's older MCP server in `com.unity.ai.assistant`, and the in-editor Assistant — so which one you are holding changes nothing below. From a terminal with no tools wired up at all, `unity command juicebox_read` and `unity command juicebox_writescene --content '<json>'` reach the same code. `juicebox_read` returns the dump itself; a successful write returns its report with the fresh dump already in the report's `dump` field, so the next edit needs no second call. The two write tools are split by blast radius — open scene versus shared asset — and each refuses a payload belonging to the other, naming it. Everything else in this document is unchanged by the transport.

**The agent bridge (fallback).** When those tools are absent. If `<project>/JuiceBoxAgent/` exists, work through it instead of asking a human to click menus — the Unity editor watches it while running:

- Drop a `.jbx` (create), `.jbxpatch` (modify), `.jbxinsert` (place on a scene object), or `.jbxownership` (copy / makeUnique / linkDefinition / extractDefinition) into `JuiceBoxAgent/in/` → read `JuiceBoxAgent/out/<name>.report.json` for the verdict. A `.jbx` may carry a root `"outputAsset": "Assets/..."` naming where its definition is created.
- Write `in/<x>.request.json` with `{"action":"list"}` to enumerate definitions, `{"action":"scene"}` to survey the JuiceBoxAnimation components in the open scenes (scene, hierarchy path, objectId, and `definition` vs `inline` data), or `{"action":"dump","asset":"…"}` to get a fresh dump in `out/`. Dump's `asset` takes a definition asset path, or an inline address from the scene survey — the `objectId` (preferred) or `/scene/path`. Dumping a definition-backed component redirects to its shared asset (`addressedComponent` names what you asked for); inline dumps carry `"data": "inline"` and real `objectRef`s instead of `objSlotHints`. Scenes are never loaded automatically — an address into an unopened scene is refused.
- Processing may pause while the editor is unfocused; the queue drains when it regains focus. Applies are all-or-nothing, CAS-guarded, and undoable in the editor.
- **Check for `in/BRIDGE-OFF.txt` first.** If that file is present the watcher is OFF, so anything you drop will sit unprocessed and no report will appear — do not wait on `out/`; ask the human to enable the bridge in Unity (Tools > JuiceBox > LLM Hooks). The file is gone whenever the bridge is watching.

**Where to put your files.** Working files belong under `JuiceBoxAgent/` or elsewhere **outside** `Assets/`. Anything inside `Assets/` gets imported by Unity and gains a `.meta` file — churning the asset database and the user's version control on every run. Only created `.asset` definitions belong in `Assets/`.

## Data model in brief

An animation is a list of **sequences**. Each sequence animates one value of one **type** (`Float`, `Vector2`, `Vector3`, `Vector4`, `Quaternion`) and holds an ordered list of **effects** that run one after another. The sequence's **property** block wires the value to the world: `onUpdate` applies the value each frame (this is where "move the transform" happens), `setStartingValue` reads the initial value once. Effects are one of five kinds: `Tween` (start to target over a fixed time, shaped by easing), `Follow` (chase a possibly moving target at a speed), `Swing` (spring physics toward a target), `Wait` (hold until a condition), `Shake` (oscillate around the baseline). Every effect needs an `endCondition` or it never finishes.

Values come into slots through a **slot union** — exactly one of:

- `{"literal": [numbers]}` — a constant.
- `{"binding": {"mode", "class", "method", "descriptor", "objectRef"}}` — a method somewhere. Prefer the descriptor modes (`RelativeInstance`, `RelativeStatic`, `Static`): they are pure text and resolve at runtime. `Instance`/`BoundStatic` need an `objectRef` (GlobalObjectId string) and should be avoided unless one was supplied to you.
- `{"ref": {"sequenceIndex", "type", "index"}}` — a value another sequence publishes, addressed by the publisher's array position (see cross-sequence values).
- `{"instanceParam": "name"}` — one of the sequence's declared runtime parameters (see instance parameters).

**Literal arity depends on the slot, not just the sequence.** Three kinds:

- *Value slots* (`target`, `amplitude`, `setStartingValue`, `SetStartingVelocity`) take the sequence's type: Float=1 number, Vector2=2, Vector3=3, Vector4/Quaternion=4.
- *Float slots* (`Duration`, `Speed`, `Stiffness`, `Resistance`, `Frequency`, `EndConditionRange`, `EndConditionVelocity`, `timescale`) always take **1** number, even in a Vector3 sequence.
- *Callback slots* (`onUpdate`, `onComplete`, `finally`, `OnStart`, `OnDone`, `ModifyEffectState`, `EvaluateCondition`) take **no literal** — they need a binding.

**Effect slots come in three groups.** An effect's fields are checked against its **kind**, and `slots` is held to the same standard — it is not a way around it.

1. **Never written through `slots`** — these have a dedicated field, and that field is the only spelling a dump emits: `GetTargetValue` → **`target`**, `Easing` → **`easing`**, `EvaluateCondition` → **`endCondition.condition`**. Naming one inside `slots` is **JBX035**.
2. **Kind-scoped** — legal, but only on kinds that can read them, and putting one elsewhere is **JBX038** (the error's `valid` lists the kinds that accept it). `Speed` is Follow's; `Stiffness`/`Resistance` are Swing's own spring; `Amplitude`/`Frequency` are Shake's; `EndConditionRange`/`EndConditionVelocity` need a kind that can take a `WithinRange` condition, so not Tween or Shake. Note the field form (`"speed": 5`) is a literal and the slot form (`"slots": {"Speed": {...}}`) is a binding — both are legal on the right kind.
3. **Universal** — every kind may carry them: `OnStart`, `OnDone`, `ModifyEffectState`, `SetStartingVelocity`, `Duration`.

**Randomness is a binding mode**, not a separate slot kind: `{"binding": {"mode": "Random", "method": "FloatRange", "descriptor": "0.4:1.2"}}`. The function goes in `method`; its parameters are **colon-separated in `descriptor`**. Available functions and parameter counts depend on the slot's type:

| Slot type | Functions (parameter count) |
|---|---|
| Float | `FloatRange`(2), `PerlinFloat`(4), `SimplexFloat`(5) |
| Vector2 | `V2Range`(4), `OnUnitCircle`(0), `InsideUnitCircle`(0), `PerlinV2`(6), `SimplexV2`(7) |
| Vector3 | `V3Range`(6), `OnUnitSphere`(0), `InsideUnitSphere`(0), `PerlinV3`(8), `SimplexV3`(9) |
| Vector4 | `V4Range`(8) |
| Quaternion | `Rotation`(0), `RotationUniform`(0) |

A random `Duration` is a *Float* slot, so it uses `FloatRange` even inside a Vector3 sequence. Zero-parameter functions take an empty `descriptor`.

## Animation-level settings — the `animation` block

The root `animation` block holds settings of the **animation itself**, not any one sequence. It is stored on the shared definition, so every component instance of the animation follows it:

```json
{ "jbxVersion": 1, "name": "Dash",
  "animation": { "timescale": 0.5, "whenDisabled": "Pause", "whenOffscreen": "NoHandling" },
  "sequences": [ ... ] }
```

- `timescale` — global playback-speed multiplier applied on top of every sequence's own timescale (1 = normal, 0 pauses everything, negative plays backwards).
- `whenDisabled` — what running sequences do when the owning GameObject is deactivated: `Quit` (abort), `Pause` (hold, resume on re-enable), `KeepRunning`.
- `whenOffscreen` — while the object is not visible to any camera (needs a Renderer on the GameObject): `NoHandling`, `Pause`, `Quit`.

All fields optional; defaults are `1` / `Quit` / `NoHandling`, and a dump **omits the block entirely when everything is default**. The dump always carries an `animationToken` (`a:xxxxxxxx`) — the CAS token a `configureAnimation` patch op must cite (see JBXPATCH-CONTRACT.md). At runtime, code can still override these **per instance** (`SetAnimationTimescale` on one component slows just that object) — such play-mode overrides are transient and never touch the authored values.

## Combiners and cross-sequence values

Two ways sequences interact, both fully authorable:

**Combiner** — several sequences blend into one output instead of each driving a property. Declare a top-level combiner and point member sequences at its `nodeId`:

```json
"combiners": [ { "type": "Vector3", "nodeId": 3,
  "onUpdate": { "binding": { "mode": "RelativeInstance", "method": "set_localPosition", "descriptor": "./" } } } ],
"sequences": [ { "name": "Drift", "type": "Vector3", "combiner": 3, ... },
               { "name": "Shiver", "type": "Vector3", "combiner": 3, ... } ]
```

One combiner per value type. The combiner's `getInitialValue` supplies the base position/value, and each member contributes only **its own delta** on top — so a member should set neither an `onUpdate` (JBX084) nor a `setStartingValue` (JBX086, discarded at runtime). The combiner's `onUpdate` is what applies the blended result — a member with no `onUpdate` is correct, not broken. Never bind `CombinerSink` by hand (JBX085); setting `"combiner": <nodeId>` is what makes a sequence a member, and it is what the graph reads to draw the → combiner link.

**Cross-sequence values** — a sequence publishes one of its outputs into a value slot, and another sequence's slot reads it:

```json
// sequence 0:
{ "name": "Ramp", "type": "Float",
  "outputs": { "Result": { "type": "Float", "index": 1 } }, ... }

// an effect in another sequence reads Ramp's output — Ramp is sequence 0:
{ "kind": "Follow", "slots": { "Speed": { "ref": { "sequenceIndex": 0, "sequence": "Ramp", "type": "Float", "index": 1 } } } }
```

Output kinds: `Target`, `Velocity`, `Timescale`, `Result` (the animated value itself).

**Publish to an index nothing else uses.** Literals are allocated by *appending* to the sequence's value list, so their indices are implicit and do not appear in the dump; an output is addressed *explicitly*. If an output writes to an index some effect also reads, the output overwrites it every frame — a feedback loop, refused with JBX088. Each sequence's dump carries `valueSlotsUsed` (e.g. `{"Float": 3}`): **publish at or above that count** and you cannot collide. The `ref` is the third member of the slot union, addressed by the publishing sequence's **`sequenceIndex`** (its array position — the file order when authoring, or the dump's `index`); an optional `sequence` name may be echoed alongside (advisory, warns on mismatch). `type` + `index` must match its output exactly (copy them from a dump). A slow Float sequence driving a fast one's `Speed` is the canonical use.

## Instance parameters — per-run values from code

A sequence can declare named **instance parameters** so code reuses one animation with different values per run:

```csharp
anim.RunSequence("Dash", new InstanceParameter("speed", 12f),
                         new InstanceParameter("distance", new Vector3(4, 0, 0)));
```

Declare them on the sequence and read them from slots by name:

```json
{ "name": "Dash", "type": "Vector3",
  "instanceParams": [
    { "name": "speed", "type": "Float", "value": [8] },
    { "name": "distance", "type": "Vector3", "value": [2, 0, 0] }
  ],
  "effects": [ { "kind": "Follow",
    "target": { "instanceParam": "distance" },
    "slots": { "Speed": { "instanceParam": "speed" } },
    "endCondition": { "kind": "WithinRange" } } ] }
```

- `name` is **free-form text**, not a code identifier — **spaces are fine** (`"punch scale"`, `"rise height"`), as is mixed case. It is what code passes to `RunSequence` and what the graph shows verbatim as the parameter's label, so write it readably. The only rule is that names are **unique within the sequence, case-insensitively** (`"Speed"` and `"speed"` collide).
- `value` is the parameter's **default** — what a run uses when code passes nothing (arity per type, `text` for `String`). Types: `Float`, `Int`, `String`, `Vector2`, `Vector3`, `Vector4`, `Quaternion`, `Rect`.
- `instanceParam` is the **fourth member of the slot union** and reads the parameter by name (case-insensitive; spaces and case must still match a declared name). The parameter's type must match what the slot takes: a `Float` slot (`Duration`, `Speed`, ...) needs a `Float` parameter; a value slot (`target`, `amplitude`) needs the sequence's type. Callback slots cannot read one (JBX095) — bind a method there instead.
- A binding's constant argument can also be parameter-fed: `"param": {"instanceParam": "spin"}` instead of `"param": {"type": "Int", "value": [3]}`. Any parameter type is allowed there — the bound method's signature is what must match at runtime.
- **A parameter no slot consumes eventually goes away.** Persistence is granted by being
  attached to a sequence, never the other way round, so a declaration nothing reads is floating
  data. It is NOT dropped by your patch: it survives, exactly as a disconnected node survives a
  human's editing session, so declaring a parameter in one patch and wiring it in the next works
  fine. It is cleared when the animation is next loaded into the graph editor, along with its
  default -- so do not treat an unwired parameter as durable storage. The same rule retires a
  combiner no sequence is a member of.
- Parameters are **local to their sequence**; another sequence cannot read them (`instanceParam` never crosses sequences — use outputs + `ref` for that).
- In a dump, a slot wired to a parameter appears as `{"instanceParam": "name"}` and the parameter's current default appears in `instanceParams`. **Preserve both on re-import** — replacing the reference with a literal silently detaches the slot from the parameter.

**`Duration` and `EndConditionTime` are the same slot.** Use `Duration`; setting both is last-write-wins and warns.

**`evalOnce` — evaluate a binding once, or live every frame.** This matters and is easy to get silently wrong: a `Random` `Duration` with `evalOnce: false` re-rolls *every frame* instead of picking a value when the effect starts. **Omit it and you get the correct default for that slot** — the same one the graph editor picks:

| Slot | Default | Notes |
|---|---|---|
| `Duration`, `Speed`, `Stiffness`, `Resistance`, `Amplitude`, `Frequency`, `EndConditionRange`, `EndConditionVelocity` | `true` | evaluated once at start |
| `ModifyEffectState` | `false` | live each frame |
| `GetTargetValue` (`target`) | `true` for `Tween`, `false` otherwise | a Follow chases a moving target |
| `onUpdate`, `EvaluateCondition` | `false` | fixed, cannot be changed |
| `onStart`, `onDone`, `onComplete`, `finally`, `setStartingValue`, `SetStartingVelocity` | `true` | fixed, cannot be changed |

**`Instance` / `BoundStatic` need an `objectRef`, and CANNOT be used in a definition asset.** A ScriptableObject cannot store a scene reference — Unity serializes it as null and the binding fails at play time with *"Instance mode but object is missing"*. Import refuses these (JBX216). Use `RelativeInstance` with a `descriptor` instead; it resolves against the target's hierarchy at runtime and needs no stored reference. If a dump shows `brokenObjectRef`, that binding is already broken in the asset and needs rebinding in the editor.

**A descriptor is CHECKED when the target is a scene object.** Writing to a scene — an insert, or a patch on an inline animation — gives the importer a real hierarchy to walk, so it resolves your `descriptor` with the same walk the runtime uses and reports what it found: **JBX225** if the descriptor resolves to nothing, **JBX226** if it resolves but the object has no method of that name (`RelativeInstance`) or no component of that `class` (relative `FieldAccess`). Both are WARNINGS, not errors — a descriptor may legitimately point at an object spawned later — but if you did not mean that, treat one as a typo you can fix before it becomes a silent zero at play time.

Writing to a **definition asset** reports neither. A definition is deliberately shareable across many objects, so it has no one hierarchy to be right about, and a descriptor that fails against any particular object is not an error there. You get the older JBX212 note instead, saying the check was skipped and why.

**An UNSAVED scene has no object ids, so addresses there are hierarchy paths.** Unity only mints a `GlobalObjectId` once a scene has been saved to disk; before that every object in it reports the null id, which resolves to nothing. So for a scene that has never been saved, `objectId` is **absent** from the survey and from an inline dump, a binding's dead reference is reported as `unsavedObjectRef` instead of an unusable `objectRef`, and a write's `assetPath` is the `/hierarchy/path` with a **JBX224** warning. The path reads back fine, but it follows the object's position — a rename or reparent breaks it — and it matches the first hit across every open scene. Save the scene to get stable ids.

**`FieldAccess` reads or writes a FIELD, and it is three bindings wearing one mode name.** Which one you get is decided by the other fields, not by the mode, so give exactly the ones its shape needs:

| shape | give | resolves to |
| --- | --- | --- |
| relative | `class` + `method` + `descriptor` | an instance field on the `class` component of the object the descriptor names |
| static | `class` + `method`, no object, no descriptor | a **static** field on `class` |
| instance | `method` + an object (`objSlot`, or `objectRef` on an inline animation), no descriptor | an instance field on **that object's own type** |

Two consequences worth knowing. In the instance shape `class` is not needed and is ignored if given — the field is looked up on the object's own type. Import still checks it whenever that object resolves at author time (an `objectRef` on an inline animation): the field must exist AND be of the slot's type, or JBX227. Through an unfilled `objSlot` the object does not exist yet, so the check is impossible and JBX228 says so rather than passing quietly. And a descriptor beats an object: giving both is refused (**JBX059**) rather than silently ignoring the object, because the relative path never consults it. The definition-asset rule above applies to the instance shape too — a scene `objectRef` there is refused with JBX216; use an `objSlot`.

**`param` — a constant argument for the bound method.** A binding can pass one value, stored in the sequence's value lists: `{"binding": {"mode": "RelativeInstance", "method": "GetSpinTarget", "descriptor": "./", "param": {"type": "Int", "value": [3]}}}` calls `GetSpinTarget(3)`. `type` is one of `Float`, `Int`, `String`, `Vector2`, `Vector3`, `Vector4`, `Quaternion`, `Rect`; use `value` for numbers (`text` for `String`). This is how otherwise-identical effects differ — four tweens calling the same method with parameters 0..3.

**Unity properties via accessors.** `StandardFunctions` deliberately has no full-vector transform helpers (no `GetPosition`, no `SetLocalScale`) because C# properties on components are bound directly through their compiler-generated accessor methods: `method: "get_localPosition"` / `"set_localPosition"` on the object's `Transform`, `"get_position"`, `"set_localScale"`, `"set_localRotation"` (Quaternion), and likewise any property on any component (`"set_intensity"` on a Light, `"set_alpha"` on a CanvasGroup). Use mode `RelativeInstance`: it resolves the descriptor to a GameObject and searches that object's components for a public instance method with the matching name and signature — so `class` is not needed and is ignored (the validator warns if you set it). The accessor name is case-exact and keeps Unity's lowercase property spelling after the prefix: `set_localPosition`, never `set_LocalPosition` or `SetLocalPosition`. The signature must match the sequence type: a `Vector3` sequence binds `set_localPosition(Vector3)`; binding it from a `Float` sequence fails at runtime. Prefer `StandardFunctions` helpers when you want a single axis or a 2D pair of a vector property; use accessors for the full value. One caution: because resolution is first-component-match by name, a rare same-signature method-name collision across two components on one GameObject resolves to whichever component comes first.

**Descriptors** locate a GameObject relative to the animated object: grammar `[N ":"] ["./"] ("../")* [name]`. `./` = self, each `../` = one parent up, then an optional child name searched under that point, with an optional `N:` index prefix to disambiguate duplicate names. Examples: `./` (self), `../` (parent), `HealthBar` (child named HealthBar), `2:Slot` (third matching Slot).

**Easing** is a per-effect field, not a slot: `{"function": "BackOut"}` (an easing name from the closed vocabulary) or `{"curve": [{"t":0,"v":0},{"t":1,"v":1}]}`. Omit for linear. Only `Tween` and `Shake` take easing.

**End conditions**, one of four kinds:

- `{"kind":"Time","time":0.3}` — fixed duration in seconds.
- `{"kind":"WithinRange","range":0.01,"velocity":0.02}` — settles when the value is close enough to its target and (optionally) slow enough. **Both fields belong to one condition and are checked together**, not alternatives. `range` defaults to `0.01` and is always active; a negative value inverts the test. `velocity` defaults to `0`, which *disables* the velocity check. So `{"kind":"WithinRange","velocity":0.02}` still enforces the default range of 0.01 — name `range` explicitly if you want a different one. This is the natural end condition for `Swing`.
- `{"kind":"Condition","condition":<slot returning bool>}` — runs until your method returns true.
- `{"kind":"Forever"}` — runs until the sequence is stopped externally.

`Tween` and `Shake` accept `Time` only (the runtime hard-types it); `Follow`, `Swing`, and `Wait` accept any kind.

Sequence fields: `loop` (`None`/`Loop`/`Yoyo` — loops the whole effect run), `segment` (`Update` default), `triggers` (list; `Manual` means code starts it, `OnStart`/`OnEnable`/collision/visibility flags start it automatically).

**`arcMovement` — curved motion for the whole sequence.** `"arcMovement": true` makes tweens and follows travel along an *arc* rather than a straight line (the graph's "Arc" badge). It is per-sequence, not per-effect. Type rules: **`Float` cannot arc** (there is no plane to curve in — JBX098); **`Vector2`/`Vector3`/`Vector4` toggle it** (this is the one you actually set); **`Quaternion` is always arc**, so the field is implicit there — a dump normalizes a Quaternion sequence to `"arcMovement": true` and you never need to set it. Omit for straight-line.

**`onComplete` vs `finally` — two end-of-run hooks, and they fire on different things.** Both live in the `property` block, both are plain callbacks (a binding, never a literal), and both are evaluated once. The difference is WHEN:

- **`onComplete` fires only on a NATURAL finish** — the sequence ran its effects to the end. An abort does not fire it. (The one exception is an abort through the EndSequence signal, which still counts as finishing.)
- **`finally` fires whenever a run ENDS, however it ended** — natural completion, `Stop()`, the owner destroyed or deactivated, or user code throwing out of a callback.

So `onComplete` is "the animation succeeded" and `finally` is "the animation is no longer running." Use `onComplete` for anything that should only happen if the motion actually played through — awarding the pickup, advancing the state machine. Use `finally` for cleanup that must happen either way — releasing a pooled object, re-enabling input, clearing a "busy" flag. A cleanup hook on `onComplete` silently leaks when the object is destroyed mid-run, which is exactly the case `finally` exists for.

Both are optional and independent: bind either, both, or neither. A dump emits each only when it is bound.

**`timescale` — static per-sequence speed.** `"timescale": 0.5` runs the whole sequence at half speed (1 = normal, 0 = frozen; negative is rejected). This is the editor's per-sequence TIMESCALE slider, stored parallel to the sequence — **distinct from a `property.timescale` delegate slot**, which is a per-frame binding. A dump emits it only when it is not 1; author it as a plain sequence field. It is covered by the `sequenceToken`.

**A sequence's `name` is a shared-library link, not just a label — and it is optional.** In JuiceBox a name does real work: the editor keeps every sequence with the *same name* in sync across all loaded animations, propagating edits to shared fields (loop, triggers, arc movement, timescale, and the effect structure) to all same-named siblings. So naming a sequence `"Bounce"` on ten objects makes them one shared sequence you edit once. An **empty name (`""`) opts out** — an unnamed sequence is standalone, propagates nothing, and receives nothing. This is a deliberate, common choice: one-off sequences (e.g. the flight on a throwaway spawned object) are left unnamed so their edits stay local. When a dump shows `"name": ""`, that sequence is intentionally standalone — do not "fix" it by inventing a name, which would silently link it to any other sequence that happens to share that name. **A name is never an address.** Sequences are addressed by **array index** — the `"index": N` on every dumped sequence, and the position in the `sequences` array when authoring. So the tools fully support unnamed sequences (author one by leaving `name` empty or omitting it) and duplicate names (allowed, but they link via the shared library — the validator warns). `replaceSequence` targets `"index": N`; a cross-sequence `ref` targets `"sequenceIndex": N`; either may carry an advisory `sequence` name that only warns on mismatch. Because these tools never reorder or remove sequences, an index from a dump stays valid until a human reorders in the graph — which the `sequenceToken` catches.

## Shared values — several slots, one value

Two slots holding the same numbers are two independent values: change one later and the other does not follow. When slots must be **one source** — the twinned ValueNodes the graph draws — declare it once and cite it by name:

```json
{ "name": "Lob", "type": "Vector3",
  "sharedValues": [ { "name": "landing spot", "type": "Vector3", "value": [3, 4, 5] } ],
  "effects": [ { "kind": "Tween", "target": { "sharedValue": "landing spot" }, ... },
               { "kind": "Follow", "target": { "sharedValue": "landing spot" }, ... } ] }
```

`sharedValues` takes the same `{name, type, value}` (or `text` for String) as `instanceParams`, and `{"sharedValue": name}` is the fifth member of the slot union — exclusive with `literal`, `binding`, `ref` and `instanceParam` like the rest. Every slot naming it lands on ONE value slot, which is exactly what makes them twins.

**The name is local to the document, like a token.** A twin in the graph is nothing but two slots holding one value slot — there is no name on it, and a ValueNode has none to show. So nothing stores the name you write: it exists to wire up the file it appears in, and a dump derives its own from the slot, `"shared Vector3 0"`. Author with whatever name reads well; cite the dump's name when patching. (An instance parameter is the opposite and is the one truly named thing: code addresses it by name through `RunSequence`, and it draws a ParameterNode. Use `instanceParams` when code supplies the value; `sharedValues` when several slots must simply stay equal.)

**In a dump, preserve them verbatim.** A dump names every shared slot, including sharing a human made in the graph. Replacing a `{"sharedValue": ...}` with the literal it currently holds silently splits the twin — the animation looks identical until someone edits one of them.

Three rules follow from the value being shared rather than the slots being linked:

- **Pointing one slot somewhere else is ordinary.** Give it a `literal`, another `sharedValue`, an `instanceParam` — no flag, no ceremony. The group simply has one fewer member.
- **Clearing one slot is ordinary too.** It removes a reader, not the value; every other slot still reads the same thing.
- **There is no "unlink".** To give one effect its own value, point that slot at a new one — the same thing the graph asks of a human, who adds a fresh ValueNode rather than detaching an existing one.

A shared value nothing reads is dropped, and one that ends up with a single reader stops being shared and becomes an ordinary value — the graph stops drawing a twin badge at exactly the same moment.

## Example 1 — fade a material in over 0.3s with ease-out

```json
{
  "jbxVersion": 1,
  "vocabVersion": "pro-104",
  "name": "PanelFadeIn",
  "sequences": [
    {
      "name": "Alpha",
      "type": "Float",
      "triggers": ["OnEnable"],
      "property": {
        "setStartingValue": { "binding": { "mode": "RelativeStatic",
          "class": "JuiceBox.StandardFunctions", "method": "GetMaterialAlpha", "descriptor": "./" } },
        "onUpdate": { "binding": { "mode": "RelativeStatic",
          "class": "JuiceBox.StandardFunctions", "method": "SetMaterialAlpha", "descriptor": "./" } }
      },
      "effects": [
        {
          "kind": "Tween",
          "target": { "literal": [1.0] },
          "endCondition": { "kind": "Time", "time": 0.3 },
          "easing": { "function": "Pow2Out" }
        }
      ]
    }
  ]
}
```

## Example 2 — punch scale: overshoot then spring back, looping shake on a child

```json
{
  "jbxVersion": 1,
  "vocabVersion": "pro-104",
  "name": "ButtonPunch",
  "sequences": [
    {
      "name": "Scale",
      "type": "Vector2",
      "triggers": ["Manual"],
      "property": {
        "setStartingValue": { "binding": { "mode": "RelativeStatic",
          "class": "JuiceBox.StandardFunctions", "method": "GetLocalScaleXY", "descriptor": "./" } },
        "onUpdate": { "binding": { "mode": "RelativeStatic",
          "class": "JuiceBox.StandardFunctions", "method": "SetLocalScaleXY", "descriptor": "./" } }
      },
      "effects": [
        {
          "kind": "Tween",
          "target": { "literal": [1.25, 1.25] },
          "endCondition": { "kind": "Time", "time": 0.08 },
          "easing": { "function": "Pow2Out" }
        },
        {
          "kind": "Swing",
          "target": { "literal": [1.0, 1.0] },
          "endCondition": { "kind": "WithinRange", "range": 0.01 }
        }
      ]
    },
    {
      "name": "IconShake",
      "type": "Float",
      "triggers": ["Manual"],
      "loop": "Loop",
      "property": {
        "setStartingValue": { "binding": { "mode": "RelativeStatic",
          "class": "JuiceBox.StandardFunctions", "method": "GetRectTransformZRotation", "descriptor": "Icon" } },
        "onUpdate": { "binding": { "mode": "RelativeStatic",
          "class": "JuiceBox.StandardFunctions", "method": "SetRectTransformZRotation", "descriptor": "Icon" } }
      },
      "effects": [
        {
          "kind": "Shake",
          "amplitude": { "literal": [6.0] },
          "frequency": 12,
          "waveform": "Sine",
          "endCondition": { "kind": "Time", "time": 0.5 }
        }
      ]
    }
  ]
}
```

## Example 3 — follow a moving target in full 3D (property accessors)

```json
{
  "jbxVersion": 1,
  "vocabVersion": "pro-104",
  "name": "CompanionFollow",
  "sequences": [
    {
      "name": "Position",
      "type": "Vector3",
      "triggers": ["OnStart"],
      "property": {
        "setStartingValue": { "binding": { "mode": "RelativeInstance",
          "method": "get_position", "descriptor": "./" } },
        "onUpdate": { "binding": { "mode": "RelativeInstance",
          "method": "set_position", "descriptor": "./" } }
      },
      "effects": [
        {
          "kind": "Follow",
          "speed": 4.0,
          "target": { "binding": { "mode": "RelativeInstance",
            "method": "get_position", "descriptor": "../Player" } },
          "endCondition": { "kind": "Condition",
            "condition": { "binding": { "mode": "RelativeInstance",
              "method": "IsDismissed", "descriptor": "./" } } }
        }
      ]
    }
  ]
}
```

## Example 4 — invalid file and its report (learn the error format)

```json
{ "jbxVersion": 1, "name": "Broken", "sequences": [
  { "name": "S", "type": "Flaot", "effects": [
    { "kind": "Tween", "endCondition": { "kind": "WithinRange", "range": 0.1 },
      "easing": { "function": "EaseOutBack" } } ] } ] }
```

```json
{ "status": "fail", "errors": [
  { "code": "JBX012", "path": "$.sequences[0].type", "message": "Unknown value 'Flaot'.", "suggestion": "Did you mean 'Float'?",
    "valid": ["Float", "Vector2", "Vector3", "Vector4", "Quaternion"] },
  { "code": "JBX023", "path": "$.sequences[0].effects[0].endCondition.kind", "message": "Tween does not accept end condition kind 'WithinRange'.", "valid": ["Time"] },
  { "code": "JBX031", "path": "$.sequences[0].effects[0].easing.function", "message": "Unknown value 'EaseOutBack'.", "suggestion": "Did you mean 'BackOut'?",
    "valid": ["Step1", "Step2", "…the full easing list…"] } ] }
```

Note the easing naming style: `BackOut`, `Pow2InOut`, `SineIn` — not `EaseOutBack`. Every easing name used in this contract is a canonical spelling. When you miss, an unknown-name finding carries the whole set it was checked against in **`valid`** — including per-asset sets like a sequence's instance parameters or a file's combiner nodeIds, which no vocabulary read could tell you — and `suggestion` fires when the miss is a near-spelling. Correct from `valid` instead of re-reading.

## Scene references — objSlot

`Instance` and `BoundStatic` bindings resolve a real object at runtime. A definition asset
cannot store a scene object (it would serialize as null), so scene targets are held as
**ObjSlots**: the definition keeps a hint per slot (a resolve path, an optional component
type, and a label), and a binding cites a slot by id. At play time the animation's component
resolves each slot in its own scene — so one authored definition works in any scene, and an
instance can override where a slot points without touching the asset.

Put an `objSlot` on an `Instance`/`BoundStatic` binding one of two ways.

**Reference an existing slot** — an integer id taken from a dump's `objSlotHints` (retarget
or reuse a slot the definition already has):

```json
"target": { "binding": { "mode": "Instance", "method": "get_position", "objSlot": 3 } }
```

**Create a new slot** — an object; the importer mints the id and writes the hint:

```json
"target": { "binding": { "mode": "Instance", "method": "get_position",
  "objSlot": { "path": "/Enemies/Boss", "type": "UnityEngine.Transform", "label": "Boss" } } }
```

- `path` (required) is where the object resolves, in the same grammar as a descriptor:
  relative (`./`, `../Name`, `2:Name` to disambiguate duplicate names) or **absolute**
  (`/Root/Child/…` from the scene root). Use an absolute path for a non-sibling target that
  isn't in the animated object's ancestry.
- `type` (optional) is the component fetched at the path (e.g. `UnityEngine.Transform`); omit
  it to bind the GameObject itself. `label` (optional) is for humans.
- Never invent a numeric id for a *new* slot — the definition mints ids itself and a chosen
  number would collide. Use the object form; the id is assigned for you.

A fresh `.jbx` uses the object form for every scene target (all slots are new). A `.jbxpatch`
uses the integer form to reuse an existing slot, or the object form to add one. Give either
`objSlot` or `objectRef`, never both; an `objectRef` that points at a scene object is refused
(it can't be stored) — use an `objSlot` instead.

To discover slots, **dump the definition and read the root `objSlotHints`** (each entry:
`slot`, `label`, `path`, `type`); each binding that uses one shows `"objSlot": <slot>`. There
is no separate list-slots command.

An ObjSlot binding shows **red** in the graph on the bare definition (nothing resolved yet)
and turns **blue** once an object is assigned to it per-instance in the inspector, or the
runtime resolves the hint path in-scene. That is expected, not an error.

## Example 5 — follow a scene object by ObjSlot (absolute path, non-sibling target)

Like Example 3, but the target is a scene object the definition references natively, resolved
by an absolute path — so it works even though the boss is neither a sibling nor an ancestor of
the animated object.

```json
{
  "jbxVersion": 1,
  "vocabVersion": "pro-104",
  "name": "ChaseBoss",
  "sequences": [
    {
      "name": "Position",
      "type": "Vector3",
      "triggers": ["OnStart"],
      "property": {
        "setStartingValue": { "binding": { "mode": "RelativeInstance",
          "method": "get_position", "descriptor": "./" } },
        "onUpdate": { "binding": { "mode": "RelativeInstance",
          "method": "set_position", "descriptor": "./" } }
      },
      "effects": [
        {
          "kind": "Follow",
          "speed": 4.0,
          "target": { "binding": { "mode": "Instance", "method": "get_position",
            "objSlot": { "path": "/Enemies/Boss", "type": "UnityEngine.Transform",
              "label": "Boss" } } },
          "endCondition": { "kind": "Time", "time": 5.0 }
        }
      ]
    }
  ]
}
```

Importing this mints slot 1, writes its hint (`/Enemies/Boss`, Transform), and points the
Follow target at it. A later dump shows a root `objSlotHints` entry for slot 1 and the target
binding carrying `"objSlot": 1`. Note the target uses world `get_position` — see the next
section for why parented/following objects must use world space.

## Placing an animation in a scene — the insert payload

A third top-level payload (beside `.jbx` create and `.jbxpatch` modify) puts a `JuiceBoxAnimation` component onto an **existing** GameObject in an **open** scene:

```json
{ "jbxInsertVersion": 1,
  "target": "/Canvas/Menu/Panel",
  "definition": "Assets/Animations/Punch.asset" }
```

- `target` — the receiving GameObject: an absolute hierarchy path or a GlobalObjectId. The object must already exist (creating GameObjects is Unity's own scene tooling's job, not JuiceBox's) and its scene must be open (scenes are never loaded automatically).
- Exactly one of `definition` (asset path/GUID to **link** — the component references the shared asset, every instance of which stays in sync) or `inline` (a complete `.jbx` body to **embed** — the component owns the data). An embedded body follows the inline conventions: scene references via `objectRef`, never `objSlot` (JBX209); it cannot carry `outputAsset`/`overwrite` (JBX133).
- Refused if the target already has a JuiceBoxAnimation (JBX341) — patch that animation, or remove the component first; insert never overwrites.
- A **pure scene mutation**: undoable, marks the scene dirty, **never saves it**. On success the report's `assetPath` is the new component's GlobalObjectId — dump or patch it next.

Via the bridge, drop the payload as a `.jbxinsert` file; via MCP it is a `juicebox_writescene` shape.

## Data ownership — copy, makeUnique, linkDefinition, extractDefinition

A component's data is either **inline** (owned by that component) or **linked** (a reference to a shared definition asset that keeps every instance in sync). The ownership payload (root `"jbxOwnershipVersion": 1` + `"op"`) is the transitions between those states, plus copy:

```json
{ "jbxOwnershipVersion": 1, "op": "copy",              "from": "/A", "to": "/B" }
{ "jbxOwnershipVersion": 1, "op": "makeUnique",        "target": "/A" }
{ "jbxOwnershipVersion": 1, "op": "linkDefinition",    "target": "/A", "definition": "Assets/X.asset" }
{ "jbxOwnershipVersion": 1, "op": "extractDefinition", "from": "/A", "outputAsset": "Assets/X.asset" }
```

- **`copy`** — duplicate the JuiceBoxAnimation onto another existing GameObject (which must not already have one). Works on either data kind: a linked source yields a linked duplicate (both share the definition); an inline source yields an owned duplicate. If the inline copy carries **named** sequences the report warns (JBX343): names are the shared-library key, so those sequences stay in sync with every same-named sibling — rename or unname via a patch to truly diverge.
- **`makeUnique`** — detach a linked component: definition data becomes owned inline data and the reference clears. THE fork move: after it, edits touch only this instance.
- **`linkDefinition`** — point an existing component at a definition asset (the inverse of makeUnique; the "attach a SO" move for a component that already exists). Linking over inline data discards it in favor of the shared version — that is the point, and it is undoable. Refused only when already linked to that same asset.
- **`extractDefinition`** — inline data becomes a new shared definition asset and the component links to it. Scene references convert to per-instance objSlots (warned, JBX356, with their paths). The report's `assetPath` is the asset; `component` is the linked component's objectId.

**Buckets:** `copy`/`makeUnique` are scene mutations (`juicebox_writescene`, dirty-only); `extractDefinition` creates an asset (`juicebox_writeshared`); `linkDefinition` is accepted by **both** — it writes no asset, so it is under either ceiling. Via the bridge, drop the payload as a `.jbxownership` file.

**Fork before diverge.** Shared data changes ALL its users: a definition dump reports `"usedBy": N` (components in the open scenes), and a patch on a definition used by more than one warns (JBX303). To give one instance a variant: `copy` or `makeUnique` FIRST, then edit the fork. The same discipline applies to things outside JuiceBox's view — a template GameObject referenced by several scripts is shared via those plain references; check who references an object (Unity's tools) before editing it in place.

## Choosing bindings — pitfalls the validator cannot catch

These produce animations that validate and import cleanly but behave wrongly at play time. Check each one before you author.

**`position` vs `localPosition` — pick by hierarchy, not habit.**
`set_position` writes world space; `set_localPosition` writes in the PARENT's frame, so the value you animate moves whenever the parent moves.

- **If one animated object is a child of another animated object, or objects follow each other, use `position` (world) for all of them.** The shipped Demo 1 scene is the canonical case: `Squirrel` is a child of `Food`, the squirrel chases the food, and the food runs away — every binding is `get_position`/`set_position`. In `localPosition` the same graph breaks: the food's movement drags the squirrel's coordinate frame along with it, so the squirrel's animation and the food's animation feed into each other (motion double-applies, follows never settle).
- **If you genuinely need local space** (animating relative to a deliberately placed parent — e.g. the arc-around-a-point trick of parenting to an empty "zero" object), **the two animated objects must not be parented to each other.** Restructure the hierarchy first; do not try to compensate inside the animation.
- **Never mix spaces in one data flow.** A `Follow` whose `target` reads another object's `get_position` must write `set_position`, not `set_localPosition` — the value read is world, so the value written must be too. Same rule for `setStartingValue`: read the same member you write (`get_position` with `set_position`, `get_localPosition` with `set_localPosition`).
- The same frame rule applies to rotation: `rotation` is world, `localRotation` is the parent's frame. Scale is only ever local (`localScale`).

**UI moves on the RectTransform.** For UI, drive `anchoredPosition` (Vector2) and `sizeDelta`, never `transform.position`; fade one graphic through material/text alpha, a whole panel through `CanvasGroup.alpha`.

**Rotation is a `Quaternion` sequence,** not a Vector3 of euler angles — quaternions interpolate correctly through any orientation.

**Read the world you write.** `setStartingValue` should read the current value of the exact member `onUpdate` writes; otherwise the first frame snaps.

## Rules the validator enforces

- Every name (types, kinds, easings, slots, modes, enums) must be in the closed vocabulary. Unknown keys are errors, not ignored.
- Sequence names are **optional** and need not be unique — a name is the shared-library propagation link, not an address (sequences are addressed by array `index`). An empty name is a standalone sequence; a duplicate non-empty name links its siblings and only warns (see "A sequence's `name` is a shared-library link"). `endCondition` required on every effect; `Tween`/`Shake` take `Time` only.
- Slot union: exactly one of `literal`/`binding`/`ref`/`instanceParam`. Literal arity matches the sequence type. Callback slots (onUpdate, onComplete, finally, condition) cannot take literals or instance parameters.
- `arcMovement: true` is rejected on a `Float` sequence (JBX098); it is free on `Vector2`/`Vector3`/`Vector4` and implicit (always on) for `Quaternion`.
- `instanceParams` entries need a name (free-form — spaces/case allowed — but unique within the sequence, case-insensitively), a known type, and a default of the right arity (JBX090–JBX092). An `instanceParam` reference must name a parameter its own sequence declares (JBX094) with a type matching the slot (JBX095); combiner slots take none (JBX096).
- `Instance`/`BoundStatic` require `objectRef`; `Relative*` require a syntactically valid `descriptor`. `RelativeInstance` takes `method` only (`class` is ignored and warned about); `RelativeStatic` and the object-bound modes require `class` too.
- **A binding's SIGNATURE is checked, not just its name** (JBX227). What a slot needs falls out of the slot plus the sequence's `type` — write `T` for that type: `onUpdate` needs `void M(T)`; `setStartingValue`, `target`, `amplitude` and a combiner's `getInitialValue` need `T M()`; `setStartingVelocity` needs `T M()` too, except on a `Quaternion` sequence where it is `Vector3 M()`; `onStart`, `onDone`, `onComplete` and `finally` need `void M()`; `condition` needs `bool M()`; `ModifyEffectState` needs `EffectSignal M()`; `easing` needs `float M(float)`; and every scalar slot (`Duration`, `Speed`, `Stiffness`, `Resistance`, `EndConditionRange`, `EndConditionVelocity`, `Frequency`, `timescale`) needs `float M()`. A leading `GameObject` parameter is always accepted, and is REQUIRED by `BoundStatic` and `RelativeStatic`. Adding a `param` to the binding replaces the argument list with that parameter's type — `Speed` with a Float `param` needs `float M(float)`.
- **Some modes cannot produce some shapes at all** (JBX229), whatever method you name. `FlatStatic` binds only parameterless members, so it can never drive `onUpdate` or `easing`. `RelativeInstance`/`RelativeStatic` cannot drive a slot that both takes an argument and returns a value. `FieldAccess` reads or writes a field, so it cannot drive a plain callback. `Random` only ever produces a getter, so it cannot drive `onUpdate` or a hook.
- **A `ref` must cite one of the publisher's declared outputs** (JBX230) — its `type` and `index`, copied exactly from a dump. Any other index is rejected, including one that is merely inside the publisher's value list: that reads whatever else occupies the slot, usually one of its own effect literals, and does it silently at a plausible magnitude. An index past the end reads zero, equally silently. Neither is detectable at runtime, which is why this is refused at write time.
- **A sequence's value must go somewhere** (JBX018). Bind the property's `onUpdate`, publish the value as `outputs["Result"]`, or set `combiner` to a combiner nodeId — those three are the only ways a sequence's live per-frame value leaves it, so without one it animates and the result is discarded. Note the name: the port the graph editor labels **OnUpdate** is written here as **`Result`**. **A sequence whose effects are ALL `Wait` is exempt** — that is a timer, and its `onComplete`/`finally` hooks are the point. One `Tween` beside the `Wait`s ends the exemption, because that Tween animates a value nothing reads.
- **An effect that takes a target needs one.** `Tween` and `Swing` without a `target` are REJECTED (JBX073): this is a dead animation rather than a dead effect, because the runtime refuses to run a sequence holding one and nothing plays. A `Follow` without a target only warns (JBX074) — it moves toward zero, which does run; write `"target": {"literal": [0, ...]}` if that is what you mean. `Wait` and `Shake` take no target and are never flagged.
- **A magnitude of zero warns** (JBX075) when no slot supplies one: `speed: 0` on a Follow, `frequency: 0` or a zero/absent `amplitude` on a Shake, `stiffness: 0` on a Swing. OMITTING these is not the same as zeroing them — the defaults are alive (`speed` is infinite, `frequency` is 5, `stiffness` is 1), and only `amplitude` is zero when omitted. `resistance: 0` is never flagged: an undamped spring oscillates forever, which is a real choice. A zero field beside a bound `Speed`/`Amplitude`/`Frequency`/`Stiffness` slot is silent — the field is the fallback and the slot drives.
- **"Could not check" is reported, never passed silently.** `RelativeInstance` resolves its method against the target's components at runtime; when the animation lives on a scene component the descriptor is resolved and the signature checked against the scene as it stands now (JBX225/JBX226 — warnings, because the hierarchy may differ at play time). On a definition asset there is no hierarchy, and JBX212 says so. A binding whose object arrives through an unfilled `objSlot` gets JBX228, which states the signature it must have. Double-check spelling of user classes and methods against the project source.

## Appendix — relative runtime cost

Reference only; consult when a choice is performance-relevant (crowds, per-frame effects on many objects). Costs are qualitative tiers; setup happens once per play, per-frame recurs every frame the effect runs.

| Option | Setup cost | Per-frame cost | Notes |
|---|---|---|---|
| Literal slot | none | lowest | Compiled to a value-slot read; no delegate call. |
| Library easing (`function`) | none | low | Inlined static math. |
| Curve easing | low | low-mid | `AnimationCurve.Evaluate` per frame; costs more than library math. |
| `Static` / `FlatStatic` binding | reflection once | low | One cached delegate invocation per frame (per-frame slots only). |
| `Instance` / `BoundStatic` binding | object resolve + reflection once | low | Same per-frame cost as Static once bound. |
| `RelativeInstance` / `RelativeStatic` | hierarchy walk + reflection once | low | Descriptor resolution is setup-time; per-frame identical to bound. |
| `random` Range funcs | none | low | One RNG draw per evaluation. |
| `random` Perlin / Simplex | none | mid | Noise sampling per evaluation; Simplex slightly cheaper than Perlin at higher dimensions. |
| `Tween` | low | lowest of the kinds | Fixed-duration interpolation. |
| `Follow` | low | low-mid | Re-reads its target every frame (target binding invoked per frame). |
| `Swing` | low | mid | Spring integration per frame. |
| `Wait` | low | lowest | Condition check only. |
| `Shake` | low | mid | Waveform + easing envelope per frame; `Sine` cheapest, others comparable. |
| `useSmoothing` | low | +mid | Adds spring smoothing on top of the effect's own math. `Tween`, `Follow`, `Wait` only — `Swing` is already a spring, and its `stiffness`/`resistance` shape that spring rather than a smoothing pass. |
| `smoothing` | low | — | Shapes that spring: `{"stiffness": n, "resistance": n}`, both ≥ 0. Requires `"useSmoothing": true` beside it (JBX036); same three kinds. Omit either field to leave it alone. Defaults are stiffness 1, resistance 1. A dump emits `smoothing` whenever `useSmoothing` is on, so a read-modify-write preserves a human's tuning. |
| Value type | — | Float < Vector2 < Vector3 < Vector4 < Quaternion | Quaternion effects use slerp-class math; markedly heavier than float. |