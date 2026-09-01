---
name: juicebox-llm-integration
description: Use for procedural motion "juice" in Unity -- UI and gameplay feedback like 'make the button bounce', 'add some juice to this menu', 'make the health bar drain smoothly', 'punch the icon when it's clicked', 'this tween feels wrong'. Covers creating, retuning, inspecting and placing JuiceBox animations from text. Applies whether or not the user names JuiceBox: if the project has JuiceBox installed and the request is about making something move, spring, shake, fade, bounce or feel snappier, use this rather than hand-writing a coroutine or an Animator clip. NOT for character or skeletal animation (an Animator/Animation component, an imported rig, an AnimationClip) -- that only shares the word "animation".
required_packages:
  com.unity.nuget.newtonsoft-json: ">=3.0.0"
---

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


# JuiceBox animations from text (LLM Hooks)

Author, inspect, and modify JuiceBox animations from text -- no graph editor -- through
three tools the Unity editor registers while it is running.

**Read this where it is; do not copy it into a skills folder.** It ships with the add-on
and is versioned with it, so a copy stops matching the installed build the moment JuiceBox
updates -- and the names and payload shapes below are exactly what changes. Setup deletes
stray copies for that reason.

## Which transport you have

Unity must be running either way.

- **`juicebox_read` / `juicebox_writescene` / `juicebox_writeshared` visible?** Use them.
  That is the primary path and the loop below is written for it.
- **Not visible as tools, but you have a terminal?** They are also Unity CLI commands:
  `unity command juicebox_read`, `unity command juicebox_writescene --content '<json>'`,
  `unity command juicebox_writeshared --content '<json>'`, run from the project root.
  Same names, same payloads, same reports as the tools -- the loop below is unchanged.
  Check with `unity command juicebox_read`; if the command is unknown, the project needs
  `com.unity.pipeline` (`unity pipeline install`).
- **Neither?** Two things register these, and each fails differently. Unity 6 with
  `com.unity.pipeline` registers them with the CLI, and `unity mcp configure <client>`
  republishes them as MCP tools. Unity 6 with `com.unity.ai.assistant` registers them
  with Unity's older MCP server, where they arrive DISABLED -- so "absent" there can
  mean "installed but switched off", and the human enables them under
  Project Settings > AI > Unity MCP Server. If the human wants neither, use the
  **File bridge (fallback)** section near the end -- same payloads, delivered as files.

The two write tools are split by BLAST RADIUS, which is also what a human authorizes:
`juicebox_writescene` mutates only an open scene (marks it dirty, never saves it),
`juicebox_writeshared` writes shared definition asset FILES that every instance in every
scene follows. Each refuses a payload belonging to the other and names the right one, so a
misdirected write costs one retry and never lands somewhere unintended.

## The loop

1. **Survey** -- `juicebox_read` with no target: every definition asset, plus every
   JuiceBoxAnimation in the open scenes with its scene, hierarchy path, objectId, and
   whether its data is a shared `definition` or `inline`.
2. **Look names up** -- `juicebox_read` with target `vocabulary`: effect kinds, easings,
   triggers, slots, modes. Binding method names (`SetLocalScaleXY`, `get_position`, ...)
   are NOT in there -- they come from a dump of an existing animation.
3. **Read one** -- `juicebox_read` with an address: a definition asset path, an inline
   animation's `objectId` (preferred; unambiguous), or its `/scene/path`. The dump is
   ground truth: tokens, combiners, value slots, everything. Reading a definition-backed
   component redirects to its shared asset and says so (`addressedComponent`); patch the
   asset and every instance follows.
4. **Create** -- `juicebox_writeshared` with a `.jbx` body (root `jbxVersion`); add
   `"outputAsset": "Assets/..."` to choose where the definition is created.
5. **Modify** -- author a `.jbxpatch` (root `jbxPatchVersion`) from a FRESH read's tokens.
   A definition asset goes to `juicebox_writeshared`; an inline animation goes to
   `juicebox_writescene` and marks the scene **dirty** (the human saves with Ctrl+S, or
   Unity's own save-scene command does). JuiceBox never writes the scene file itself.
6. **Place in a scene** -- `juicebox_writescene` with
   `{"jbxInsertVersion": 1, "target": "/Root/Child" or a GlobalObjectId, "definition":
   "Assets/X.asset"}` to LINK a shared definition, or `"inline": { ...full .jbx body... }`
   to EMBED component-owned data. Adds a JuiceBoxAnimation to that EXISTING GameObject
   (create the object first with Unity's own scene tooling); refuses if one is already
   there. The report's `assetPath` is the new component's objectId -- read/patch it next.
   Scene marked dirty, never saved.
7. **Change data ownership** -- `{"jbxOwnershipVersion": 1, "op": ...}` with op `copy`
   (`from`/`to`: duplicate an animation onto another object) or `makeUnique` (`target`:
   detach an instance from its shared definition into owned inline data) via
   `juicebox_writescene`; `extractDefinition` (`from` + `outputAsset`: turn inline data
   into a new shared definition and link it) via `juicebox_writeshared`; `linkDefinition`
   (`target` + `definition`: point a component at a definition asset) via either.

Inline animations hold scene references directly: use `objectRef` (a GlobalObjectId from a
dump) on their bindings, never `objSlot` -- slots exist only on definition assets. Scenes
are never loaded automatically: an address into an unopened scene is refused, so survey
first to see what is reachable.

Every successful write changes every token in that animation, and the report hands you the
new state so you need no second call -- read its `dump` field. On an animation too large to
carry inline, `dumpOmitted` says so and you re-read the address in `assetPath`. Never
author from tokens minted before a write.

## Rules that matter

- Tokens are opaque: copy them verbatim from a read, never construct or edit one. A
  rejected patch (stale token) means the animation changed -- re-read and re-author. Apply
  is all-or-nothing and undoable in the editor.
- A write report with `"status": "fail"` is a verdict, not a fault: `errors` carries a
  code, a JSON path, a message, and for an unknown name the whole `valid` set it was
  checked against. Fix from that and retry; report what the report actually says, and
  never claim success without a `pass`.
- Names are a closed vocabulary; unknown names are errors. Do not guess -- look them up.
- A sequence's `name` is a shared-library LINK, not an address: the editor keeps all same-named
  sequences in sync across loaded animations (loop, triggers, arc, timescale, structure). An
  empty name (`""`) is a deliberate STANDALONE sequence -- don't invent a name for it. Names are
  OPTIONAL and need not be unique (a duplicate just warns). **Address sequences by INDEX**, not
  name: every dumped sequence carries `"index": N`; `replaceSequence` targets `"index": N` and a
  cross-sequence `ref` targets `"sequenceIndex": N` (an advisory `sequence` name may ride along,
  warns on mismatch). Full rules in `../../../JBX-CONTRACT.md`.
- **Fork before diverge.** Shared data changes ALL its users: a definition dump reports
  `usedBy` (linked components in the open scenes), and patching a definition used by more
  than one WARNS. To give ONE instance a variant, `copy` or `makeUnique` it FIRST, then edit
  the fork. Same-named sequences are shared the same way (rename/unname to diverge), and a
  template object referenced by several scripts is shared OUTSIDE JuiceBox's view -- check
  who references an object before editing it in place.
- Position bindings: if animated objects are parented to or follow each other, bind
  `position` (world), NOT `localPosition`. See the pitfalls in `../../../JBX-CONTRACT.md`.

## Verification -- check your own work

You do not need a human to tell you whether a write landed. Every write answers with a
report, and a `"status": "pass"` with an empty `errors` array means the apply happened in
full -- applies are all-or-nothing, so there is no partial state to inspect.

Then read the result rather than assuming it:

- The report carries the fresh `dump` of what you just wrote. Compare it against what you
  intended -- effect count, the values in the slots you set, the bindings you expected.
- If `dumpOmitted` appears instead, the animation was too large to inline: read the
  address in `assetPath` and check that.
- A scene write reports `"saved": false` by design. That is correct, not a failure: the
  scene is marked dirty and the human saves it.
- `warnings` are not failures, but read them -- they are how the importer tells you it did
  something you did not ask for, such as falling back to a hierarchy path (JBX224) or
  finding a descriptor that resolves to nothing (JBX225).

## Common issues

| Symptom | Cause | Fix |
| --- | --- | --- |
| A patch is refused with a stale-token error | The animation changed after your dump; every write changes every token | Read it again and re-author against the fresh tokens. Never edit a token by hand. |
| An unknown-name error naming a field you were sure of | The vocabulary is CLOSED and the name was invented | The error's `valid` array lists the whole accepted set -- pick from it. Binding method names come from a dump, not the vocabulary. |
| JBX302 on a patch | You addressed a component whose data lives on a shared definition | Patch the definition the error names; every instance follows. |
| JBX401 / JBX402 | The payload's blast radius belongs to the other write tool | The error names the right tool. Re-send unchanged to that one. |
| Files dropped in `JuiceBoxAgent/in/` produce no report | The bridge ships OFF; `in/BRIDGE-OFF.txt` says so | Do not wait on `out/`. Ask the human to enable it (Tools > JuiceBox > LLM Hooks). |
| An address from a write does not read back | The scene has never been saved, so its objects have no stable id | The report warned with JBX224 and gave a hierarchy path -- use that, and suggest saving the scene. |
| A binding reads zero at play time | A relative `descriptor` names an object that is not there | On scene targets the importer warns (JBX225/JBX226) with what it resolved against; fix the path or create the object first. |

## Scene references (objSlot)

A definition asset can't hold a scene object directly, so scene targets are **ObjSlots**.
On an `Instance` / `BoundStatic` binding:

- **Reuse/retarget an existing slot:** `"objSlot": <id>` -- the id comes from the dump.
- **Create a new one:** `"objSlot": { "path": "/Root/Child", "type": "...", "label": "..." }`.
  `path` is relative (`./`, `../Name`, `2:Name`) or absolute (`/Root/Child`); `type`/`label`
  optional. Resolved per-scene at play time, so it works when the definition is dropped into
  a different scene than it was authored in.

To find a slot's id, **dump the definition and read `objSlotHints`** (each entry: id, label,
path, type). There is no separate "list slots" command -- the dump is the source of truth.

An ObjSlot-backed binding shows **red** in the graph on the bare definition (no object resolved
yet) and turns **blue** once an object is assigned per-instance in the inspector, or the runtime
resolves the hint path in-scene. That is expected -- the definition holds the slot + path, and
each instance resolves it locally without touching the authored asset.

## Animation-level settings

A root `"animation"` block holds settings of the animation itself (not any one
sequence): `timescale` (global speed multiplier), `whenDisabled` (Quit/Pause/
KeepRunning on GameObject deactivate), `whenOffscreen` (NoHandling/Pause/Quit).
Stored on the shared definition -- every instance follows. Dumps always carry an
`animationToken` (a:xxxxxxxx); change the block via the `configureAnimation`
patch op citing that token. Details in `../../../JBX-CONTRACT.md`.

## Instance parameters (per-run values from code)

A sequence can declare named runtime parameters that code overrides per run:
`anim.RunSequence("Dash", new InstanceParameter("speed", 12f))`. In `.jbx`, declare
them on the sequence (`"instanceParams": [{"name": "speed", "type": "Float",
"value": [8]}]` -- `value` is the default) and read them from slots with the fourth
slot-union member, `{"instanceParam": "speed"}`. Names are free-form (spaces and
mixed case are fine, e.g. `"punch scale"`); they need only be unique within the
sequence, case-insensitively. In a dump, preserve both the
declaration and every `instanceParam` reference verbatim on re-import -- swapping
one for a literal silently detaches the slot from its parameter. Full rules and
patch semantics: the "Instance parameters" sections of the two contracts.

## File bridge (fallback)

Only when the `juicebox_*` tools are absent. A folder the editor watches: drop a payload
into `JuiceBoxAgent/in/`, read the answer from `JuiceBoxAgent/out/`. Paths are relative to
the project root.

It **ships off**. Ask the human to enable it in Unity (Tools > JuiceBox > LLM Hooks), or
create `JuiceBoxAgent/in/` yourself -- the watcher starts once the folder exists. If
`JuiceBoxAgent/in/BRIDGE-OFF.txt` is present the watcher is OFF: dropped files sit
unprocessed and no `out/` report ever appears, so don't wait on it. That file vanishes
once it's watching.

Everything above still holds -- tokens, compare-and-swap against a fresh dump,
all-or-nothing applies, the closed vocabulary, fork-before-diverge. Only delivery changes:

- **Writes** are the same payloads as files: `.jbx`, `.jbxpatch`, `.jbxinsert`,
  `.jbxownership` into `in/`; verdict in `out/<n>.report.json`. There is no scene/shared
  split here -- the file's own root key routes it.
- **Reads** are request files: `{"action":"list"}`, `{"action":"scene"}` (the survey, split
  in two here), or `{"action":"dump","asset":"Assets/Path/Thing.asset"}` ->
  `out/Thing.dump.jbx`.
- **After a successful write** the asset is re-dumped to a FILE and the report's `dumpFile`
  field names it. Read that file instead of requesting another dump.
- **Write working files ONLY under `JuiceBoxAgent/`**; never into `Assets/`. Anything
  inside `Assets/` gets imported by Unity and gains a `.meta` file, churning the asset
  database and the user's version control.
- Dropping a file validates it and writes the verdict to `out/`; no external tool is needed.

## References

In the add-on itself, three folders up from this skill:

- `../../../JBX-CONTRACT.md` -- full `.jbx` schema, worked examples, and pitfalls the
  validator cannot catch (including position vs localPosition on parented objects).
- `../../../JBXPATCH-CONTRACT.md` -- the `.jbxpatch` modification format and token rules.

And one generated file, which is the ONLY authority on names for the installed build:

- `.agents/skills/juicebox-llm-hooks/references/manifest.json` at the project root --
  the complete machine-readable vocabulary (effect kinds, slots, easings, delegate
  modes). It is regenerated from the installed version every time agent access is set
  up, so it cannot disagree with the code the way a hand-written list would.