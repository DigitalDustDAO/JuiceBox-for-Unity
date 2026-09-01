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

# JBXPATCH Contract — modifying existing JuiceBox animations

Read JBX-CONTRACT.md first; the effect/slot/easing vocabulary is identical. This file covers only modification. Modification never re-imports a `.jbx` — it patches the live `JuiceBoxAnimationDefinition` in place so everything you don't touch (graph layout, combiners, cross-sequence wiring, human edits) survives untouched.

## Workflow

1. **Dump** the current asset: `Unity -batchmode -projectPath <p> -executeMethod JuiceBox.LLMHooks.JbxDump.Run -jbxDump Assets/Anim.asset -jbxOut anim.dump.jbx`. (With the agent tools available, `juicebox_read <address>` returns the same dump and is the better route.)
2. **Read** the dump. It is `.jbx`-shaped plus a `token` per effect and a `valueToken` per sequence. It is not importable; its only role is telling you what exists and minting tokens.
3. **Author** a `.jbxpatch` (schema below).
4. **Apply**: headless, `-executeMethod JuiceBox.LLMHooks.JbxPatchApply.Run -jbxPatch <file>`. Apply validates the patch — token match (compare-and-swap), closed vocabulary, slot collisions — and applies all-or-nothing; the result (pass, or a structured rejection that changes nothing) is written to `<file>.apply.json`.

## Addressing a sequence: by index, never by name

`replaceSequence` targets a sequence by its **`index`** — the array position shown as
`"index": N` on every sequence in a dump. A sequence's `name` is NOT an address: it is the
shared-library propagation link (same-named sequences sync to each other), it may be empty, and
it may repeat. Indices are stable — these tools cannot reorder or remove sequences (`appendSequence`
only adds at the end; a sequence is retired by emptying it), so an index from a dump stays valid
until a human reorders in the graph, which the `sequenceToken` then catches (JBX312).

You MAY also include `"sequence": "<name>"` as an advisory echo for the human reader; if it
disagrees with the sequence actually at `index`, the apply WARNS (JBX315) but proceeds — index
always wins. Cross-sequence `ref`s address their publisher the same way, by `"sequenceIndex": N`
(see JBX-CONTRACT.md).

## Tokens are compare-and-swap

Every effect token (`e<index>:<hash>`) and every `sequenceToken` (`s:<hash>`) is verified against the live asset before anything mutates. One mismatch rejects the **whole patch** and the asset is untouched. Tokens are opaque: copy them verbatim, never construct or edit them.

A token is a hash of content, not a version counter, so **identical content gives an identical token** — undo an edit, or clear a slot you just bound, and the token returns to what it was. Two tokens differing is the only signal that matters; two tokens matching never means "no edit happened", only "the content is the same now". Never cache a token across a write: re-read.

**The `sequenceToken` is the important one.** `replaceSequence` is declarative — the `effects` array you send becomes the *entire* effect list. So if a human added an effect after your dump, and your patch doesn't mention it, it would be **deleted**. The `sequenceToken` covers the whole sequence (effect count and every effect's content in order, the value lists, combiner membership, property bindings + arc, the instance-parameter table, output slots, and the sequence-level loop / segment / triggers), so any such change rejects the patch instead. If you get JBX312, someone edited the sequence after your dump: re-dump and re-author against the current state.

## Patch schema

```json
{
  "jbxPatchVersion": 1,
  "vocabVersion": "pro-104",
  "asset": "Assets/Animations/ButtonPunch.asset",
  "operations": [
    {
      "op": "replaceSequence",
      "index": 0,
      "sequence": "Scale",
      "type": "Vector2",
      "sequenceToken": "s:8f3a91bc",
      "set": { "loop": "Yoyo" },
      "effects": [
        { "keep": "e0:a1b2c3d4" },
        { "keep": "e1:e5f60718", "set": { "easing": { "function": "BackOut" } } },
        { "new": { "kind": "Wait", "endCondition": { "kind": "Time", "time": 0.2 } } }
      ]
    },
    {
      "op": "appendSequence",
      "newSequence": { "name": "Glow", "type": "Float", "effects": [ ... ] }
    }
  ]
}
```

`asset` names the patch target: a definition **asset path or GUID** (above), or an **inline
animation** in an open scene, addressed by its `objectId` (a GlobalObjectId, preferred) or
`/scene/path` — both come from the bridge's `{"action":"scene"}` survey. Inline specifics:

- A **definition-backed component** is refused (`JBX302`) with the shared asset's path —
  patch that instead; every instance follows. Only truly inline data is patched in place.
- Inline bindings hold scene references **directly**: use `objectRef` (a GlobalObjectId
  from the dump). `objSlot` is refused on inline targets (`JBX209`) — slots exist only on
  definition assets.
- A successful inline apply edits the component and marks its scene **dirty** — like any
  editor edit, it stays undoable and discardable, and the human saves it (Ctrl+S) or Unity's
  own save-scene command does. JuiceBox never writes the scene file itself: a scene mutation
  only ever marks dirty (so `saved` is always `false` for inline). Definition/asset applies
  always save — the asset file *is* the change.
- Scenes are never loaded automatically: an address into an unopened scene is refused.

## replaceSequence semantics

`effects` is the **complete new node series** for that sequence, in order. It is not a diff:

- `{"keep": token}` — the live effect is reused as-is. Kept effects come back byte-identical; nothing is rebuilt from this file's (deliberately lossy) vocabulary.
- `{"keep": token, "set": {...}}` — reused, then the named fields are changed on top. `set` takes any effect field except `kind`. Everything you don't name keeps its current value — this is how you change one thing without respecifying the effect.
  - **A slot you DO name is REPLACED, not merged.** Naming a slot (`target`, `amplitude`, an entry in `slots`, …) makes the shape you give it what that slot becomes: a `literal` / `ref` / `instanceParam` replaces a binding that was there, and a `binding` replaces a value. There is no way to edit part of a slot — restate the whole slot, copying from the dump what you want to keep.
  - **`evalOnce` is the one thing carried across a slot replacement.** Re-specifying a slot that already had a binding keeps its current `evalOnce` unless your new binding names one, because `evalOnce` is not something you can see from the field you were changing and losing it is silent (a run-once `Duration` that starts re-rolling every frame looks like the animation, not like the patch). Set it explicitly to change it. A slot that had no binding takes the per-slot default, same as generation.
- `{"new": {...}}` — a full effect in `.jbx` vocabulary, exactly as in generation.
- **Omitting** a token removes that effect. **Repeating** a token duplicates the effect (the second occurrence is a clone). **Reordering** entries reorders the effects. An empty `effects` array empties the sequence — that is also how you retire a sequence, since sequence removal is not supported.
- Cross-sequence edges targeting kept effects follow them to their new position automatically; edges targeting removed effects are dropped.

**Sequences fed into a combiner** (`combinerNumber` non-zero in the dump) mix their output with other sequences rather than driving a property directly — such a sequence may legitimately have no `onUpdate`. You can retune its effects, but do not assume its output goes where an ordinary sequence's would.

`type` is required. Declaring a **different type** than the sequence currently has is a full rebuild: no entry may be `keep` (typed effects cannot cross a type change — author every effect as `new`), the old property and all its slots are discarded, and all cross-sequence edges into the sequence are dropped. Respecify `set.property` or the sequence will animate nothing visible (the apply warns with JBX331 if you don't). The `valueToken` is still required — value-slot lists survive a type change. `set` at the op level updates `loop`, `segment`, `triggers`, `arcMovement`, `timescale`, `instanceParams`, or `property` slots; absent fields keep current values. (`timescale` is the static per-sequence speed multiplier — the editor's slider, distinct from a `property.timescale` delegate; 1 = normal, 0 = frozen, negative rejected. Covered by the `sequenceToken`.) (`arcMovement` is the per-sequence arc toggle — Vector2/3/4 only; restate it after a type change, since that rebuilds the property. The `sequenceToken` covers it, so a graph toggle invalidates a stale patch.) Renaming a sequence is not supported by `replaceSequence` (there is no `name` in `set`) — rename in the graph editor; addressing is by `index`, so a rename never breaks a patch.

**`set.instanceParams` is a declarative FULL replacement** of the sequence's runtime parameter table (see the instance-parameters section of JBX-CONTRACT.md), like `effects` is for effects. Absent = keep the live table untouched. When present: an entry whose name matches a live parameter of the same type keeps that parameter's value slot — every slot already reading it stays wired — and rewrites its default; a new name (or a type change, which warns with JBX097) allocates fresh; live parameters not re-listed are removed. Copy the entries you want to keep from the dump's `instanceParams` verbatim. The `sequenceToken` covers the parameter table, so a patch cannot silently rewire parameters a human added after the dump.

### `set.sharedValues` — and the one write that needs confirming

Retunes a shared value in place. Cite the name **from a fresh dump** (`"shared Vector3 0"` and the like — those names are derived from the slot, not stored, so they are what the applier resolves against) and give the new value; a name that matches nothing live declares a new shared value for this patch's own slot references. See the shared-values section of JBX-CONTRACT.md.

Restating a shared value rewrites the ONE slot every twin reads, so it changes motion you may not have been looking at. When the named value has **two or more readers**, the apply is refused with **JBX344** naming the count, and you re-send with `"updateAllTwins": true` beside `sharedValues` in the same `set`. A value with one reader has a blast radius of exactly one effect and needs no flag.

Nothing else can reach a shared value. Pointing a slot at something else, or clearing it, allocates a fresh value and re-points — it removes a reader without touching what the others read, so those need no confirmation.

## configureCombiner

```json
{ "op": "configureCombiner", "combinerType": "Vector3", "combinerToken": "c:9a1b2c3d",
  "enabled": true, "nodeId": 3,
  "onUpdate": { "binding": { "mode": "RelativeInstance", "method": "set_localPosition", "descriptor": "./" } } }
```

Declarative per-type combiner state. `combinerToken` comes from the dump's `combiners` entry; use `c:00000000` if the dump shows no combiner of that type yet. Disable with `"enabled": false` (the config stays; members detach by setting their `combiner` to 0 via `replaceSequence` `set`). Membership is per sequence: `set.combiner` in a `replaceSequence`. Published outputs likewise: `set.outputs`.

A sequence's `sequenceToken` covers its combiner membership and outputs too, so a patch cannot silently detach or rewire what a human set up after the dump.

## configureAnimation

```json
{ "op": "configureAnimation", "animationToken": "a:9a1b2c3d",
  "animation": { "whenDisabled": "Pause", "timescale": 0.8 } }
```

Declarative update of the animation-level settings (see the `animation` block in JBX-CONTRACT.md). Null/absent fields keep current values. `animationToken` comes from the dump root (always present there, even when the block itself is omitted for being all-default); a mismatch rejects with JBX323 — the settings changed since the dump, re-dump and re-author. On a definition the write lands on the shared asset, so every instance follows; on an inline animation it edits the component.

## appendSequence

`newSequence` is a full `.jbx` sequence object; it is appended at the end (its index becomes the old sequence count). A NON-empty `name` must not already exist in the asset (a name collision would silently link the two via the shared library — JBX320). An **empty name is allowed** — that is a deliberate standalone sequence, and any number may coexist.

## Failure modes worth knowing

- `JBX31x` — CAS failures (unknown sequence, `keep` across a type change, stale token, stale valueToken). Always fix by re-dumping; never by editing tokens.
- `JBX330` — a build error occurred *after* verification (e.g. a class in a `new` effect didn't resolve). The asset is deliberately **not saved**; tell the human to revert/reload before retrying. Avoid this by validating bindings carefully up front.
- Overriding a `target` with a new literal allocates a new value slot; the old value stays in the list harmlessly (slots are grow-only).