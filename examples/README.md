# Worked example: teaching an agent to punch a health bar

This walks the full editing loop an agent runs against JuiceBox for Unity:
**read → author a patch → apply → re-read**. Every file in this folder is a real
payload shipped with the add-on; the token values below are illustrative, because
tokens are minted by the read step against your project's actual asset.

The task: *"make the health bar punch when the player takes damage."*

---

## 1. Read the vocabulary

Names in a `.jbx` come from a closed vocabulary. Inventing one is a validation
error, not a silent no-op, so the first call is always:

```
unity command juicebox_read --content 'vocabulary'
```

This returns the legal effect kinds (`Tween`, `Follow`, `Swing`, `Shake`, `Wait`,
`Yoyo`, …), easing function names, binding modes, and end-condition kinds.

## 2. Read what already exists

```
unity command juicebox_read
```

With no argument this returns an overview: every animation definition asset in
the project and every `JuiceBoxAnimation` in the open scenes, each with an
`objectId` and a note of whether it is a shared definition or inline. Pick the
address you want and read it:

```
unity command juicebox_read --content 'Assets/Animations/HealthBar.asset'
```

That dump is ground truth. It is `.jbx`-shaped with two additions: a `token` on
every effect and a `valueToken`/`sequenceToken` per sequence. Those tokens are
what make the next step safe.

## 3. Author the change

If the animation does not exist yet, you write a `.jbx` and create it — see
[`02_ButtonPunch.jbx`](02_ButtonPunch.jbx), which is exactly this punch shape: a
fast `Tween` up to 1.25 scale over 0.08s eased `Pow2Out`, then a `Swing` back to
1.0 that ends when it settles `WithinRange` 0.01. That second effect is why it
reads as a punch rather than a pulse — it overshoots and oscillates down instead
of easing back.

If it *does* exist, you never regenerate it. You patch it:

```json
{
  "jbxPatchVersion": 1,
  "vocabVersion": "pro-104",
  "asset": "Assets/Animations/HealthBar.asset",
  "operations": [
    {
      "op": "replaceSequence",
      "index": 0,
      "sequence": "Scale",
      "type": "Vector2",
      "sequenceToken": "s:c026c0a8",
      "effects": [
        { "keep": "e0:e805c9eb" },
        {
          "kind": "Swing",
          "target": { "literal": [1.0, 1.0] },
          "endCondition": { "kind": "WithinRange", "range": 0.01 }
        }
      ]
    }
  ]
}
```

Three things in that payload are the whole point of the format:

- **`index`, not `name`, is the address.** A sequence name is the shared-library
  propagation link; it can be empty and it can repeat. `sequence` is an advisory
  echo for the human reading the diff — if it disagrees with the sequence at
  `index`, the apply warns and proceeds, index winning.
- **`keep` preserves an effect you are not touching.** `replaceSequence` is
  declarative: the `effects` array becomes the *entire* effect list. Omitting an
  effect deletes it. `keep` cites the existing effect by token so you can append
  to a sequence without restating it.
- **`sequenceToken` is compare-and-swap.** It hashes the whole sequence — effect
  count, every effect's content in order, value lists, combiner membership,
  property bindings, the instance-parameter table, output slots, loop and trigger
  settings. If a human touched any of it since your dump, the patch is refused
  (`JBX312`) and **nothing** is written. That is the guarantee that lets an agent
  edit a hand-tuned animation at all.

## 4. Apply and read the report

```
unity command juicebox_writeshared --content "$(cat healthbar-punch.jbxpatch)"
```

Applies are all-or-nothing and register a single editor Undo step, so the human
can revert with Ctrl+Z. The call returns a JSON report:

```json
{ "status": "pass" }
```

or, on failure, a structured rejection that changed nothing:

```json
{
  "status": "fail",
  "errors": [
    {
      "code": "JBX312",
      "path": "operations[0].sequenceToken",
      "message": "Sequence token does not match the current asset.",
      "suggestion": "Re-read the asset and re-author against the current state."
    }
  ]
}
```

Never report success without a `pass`. On `JBX312`, go back to step 2 — someone
edited the sequence after your dump.

## 5. Re-read

Tokens are content hashes, not version counters, so identical content yields an
identical token. Two tokens matching never means "no edit happened", only "the
content is the same now". Never cache a token across a write — re-read.

---

## A note on `vocabVersion`

`vocabVersion` is **advisory**. A mismatch against the installed build raises a
`JBX104` warning and the payload still applies — it is a drift signal, not a gate.

The example files in this folder carry `"vocabVersion": "pro-104"`, which is what
they shipped with; current builds carry `pro-106`. Rather than copying a version
string out of any document, read the one your build actually carries:

```
unity command juicebox_read --content 'vocabulary'
```

## The files here

| file | what it demonstrates |
|---|---|
| `01_PanelFadeIn.jbx` | Minimal single-sequence animation; a `Tween` on a Float property |
| `02_ButtonPunch.jbx` | Two sequences; the `Tween`-then-`Swing` punch, plus a looping `Shake` on a child object |
| `03_CompanionFollow.jbx` | `Follow` effect tracking a moving target |
| `04_InstanceParams.jbx` | Instance parameters — values supplied per run at runtime |
| `05_Combiner.jbx` | Combining several sequences onto one property |
| `06_AnimationSettings.jbx` | Animation-level settings and triggers |
| `07_PlaceInScene.jbxinsert` | Placing a definition onto an existing scene GameObject |
| `08_ForkForVariant.jbxownership` | `copy` / `makeUnique` / `extractDefinition` ownership operations |
| `09_Invalid_ShouldFail.jbx` | A deliberately invalid file, to show the shape of a rejection |

Full format reference: [`../contracts/JBX-CONTRACT.md`](../contracts/JBX-CONTRACT.md)
(authoring) and [`../contracts/JBXPATCH-CONTRACT.md`](../contracts/JBXPATCH-CONTRACT.md)
(modification).
