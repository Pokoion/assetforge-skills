---
name: sprite-spec
description: Define multiple game sprites in a batch JSON spec file for AssetForge generation. Use when the user wants to create many sprites at once, define a sprite roster, or batch-generate characters. Triggers on "create all my game sprites", "generate the character roster", "batch generate sprites", or mentions of sprites.spec.json.
version: 0.1.0
---

# Sprite Spec — Batch Sprite Definition

Define multiple sprites in a single JSON file (`sprites.spec.json`) and generate them all with one command: `aforge sprite --from-spec sprites.spec.json`.

## Spec File Format

```json
{
  "version": "1",
  "sprites": [
    {
      "name": "knight",
      "description": "heavily armored medieval knight, silver sword, blue cape",
      "views": ["front", "side-scroller"],
      "variants": [
        { "slug": "golden", "description": "golden armor variant, same knight" }
      ],
      "size": "64x64",
      "based_on": null,
      "style_notes": null,
      "status": "pending"
    },
    {
      "name": "goblin",
      "description": "small green goblin, ragged clothes, holding a dagger",
      "views": ["front"],
      "size": "32x32",
      "status": "pending"
    }
  ]
}
```

## Schema Reference

### Top Level

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `version` | `"1"` | Yes | — | Schema version (must be `"1"`) |
| `sprites` | Array | Yes | — | List of sprite definitions |

### Sprite Object

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `name` | string | **Yes** | — | Asset identifier (slug, lowercase, no spaces) |
| `description` | string | **Yes** | — | Character description sent to image model |
| `views` | string[] | No | `["front"]` | Views to generate: `front`, `top-down`, `side-scroller`, `isometric` |
| `variants` | Array | No | `[]` | Named variations of this sprite |
| `size` | string | No | (config default) | Override size: `WxH` format (e.g. `64x64`) |
| `based_on` | string \| null | No | `null` | Name of existing sprite to base visuals on |
| `style_notes` | string \| null | No | `null` | Per-sprite style override notes |
| `status` | string | No | `"pending"` | Generation status (managed automatically) |

### Variant Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `slug` | string | **Yes** | Variant identifier (lowercase, no spaces) |
| `description` | string | **Yes** | Variant description appended to base description |

### Status Values

| Value | Meaning |
|-------|---------|
| `pending` | Not yet generated — will be processed |
| `generated` | Successfully generated |
| `failed` | Generation failed — check error output |

## Agent Workflow

### You are a sprite description expert

Users will give you brief, simple descriptions. Your job is to enrich them into detailed, vivid descriptions that produce high-quality results from the image model. The `description` field is the actual text sent to the image generation model — make every word count.

#### Enrichment Guidelines

**Always include these details** (the model needs them):
- Physical build (height, body type, posture, stance, facial expression if visible)
- Equipment & clothing (armor type, weapon specifics, fabric, accessories, colors)
- Color specifics (not just "red" but "crimson", "scarlet", "rusty iron", "charcoal grey")
- Material & texture (leather, chainmail, plate, fur, cloth, wood, crystal, bone)
- Distinguishing features (scars, horns, glowing eyes, tattered cape, missing horns, etc.)
- Art style cues consistent with the project config (pixel art, clean silhouette, etc.)

**Examples of enrichment:**

| User says | Enriched description |
|-----------|---------------------|
| "a goblin" | "small hunched goblin, green warty skin, large pointed ears, tattered brown loincloth, holding a rusty dagger, mischievous glare, short stature, barefoot" |
| "a knight" | "heavily armored medieval knight, polished steel plate armor with gold trim, full helm with narrow visor slit, royal blue cape flowing behind, two-handed broadsword resting point-down, broad shouldered stance, battle-worn gauntlets" |
| "a skeleton" | "undead skeleton warrior, cracked bones visible through gaps in rusted chainmail, single glowing red eye in cracked skull, broken sword held loosely, hunched posture, dust particles around feet, tattered remnants of a burial shroud" |
| "ice mage" | "tall slender ice mage, pale blue skin with frost patterns, crystalline robes shimmering with frozen edges, staff topped with a glowing blue gem, snowflakes swirling around hands, long white hair flowing, cold blue eyes, floating slightly above ground" |

**What NOT to include in description** (handled elsewhere in the pipeline):
- Background/ground plane description (magenta screen is added automatically)
- Camera angle or composition (handled by `views` field)
- "pixel art" or "game sprite" qualifiers (added from config `base_prompt`)
- Resolution or size requirements (handled by `size` field or config)

**Proactive enrichment:** When a user says "I need 10 enemy sprites" without details, ask clarifying questions or infer reasonable details from the game genre. For example, in a fantasy RPG, a "wolf" enemy could be enriched to "large grey wolf, snarling with bared fangs, matted fur with battle scars, yellow eyes, crouched hunting stance, leather collar with iron spikes".

### Filling out the spec

1. Ask user what sprites they need (characters, enemies, NPCs).
2. For each sprite, determine:
   - Name (slug)
   - Detailed description (visual appearance, equipment, colors)
   - Views needed (front only? side too? top-down? isometric?)
   - Variants (color swaps, equipment changes)
   - Size overrides (if different from project default)
   - Based-on (if this sprite should visually match an existing one)
3. Write the spec JSON to `sprites.spec.json` in the project root.
4. **DO NOT run `--from-spec` directly.** Instead, generate each sprite following the mandatory front-first checkpoint flow:
   - `aforge sprite <name> "<description>" --view front`
   - Show result → ask for approval → then generate remaining views one by one

### Processing behavior

**⚠️ MANDATORY: Front-First Design Checkpoint (even in batch mode)**

When processing a spec file, the agent MUST NOT blindly generate all views. The flow is:

1. For EACH sprite in the spec, generate ONLY the `front` view first.
2. Show the front result to the user and ASK: "Happy with this design? Any changes before generating other views?"
3. WAIT for user approval before generating additional views for that sprite.
4. Only after the user approves the front, generate the remaining views listed in the spec (one at a time, showing each result).
5. Then move to the next sprite and repeat.

**The agent MUST NOT run `aforge sprite --from-spec` directly** if the spec contains multiple views. Instead, generate sprites one by one following the checkpoint flow above.

**Exception:** If the user explicitly says "generate everything without asking" or "batch all, no checkpoints", the agent may use `--from-spec` directly.

Additional rules:
- Only sprites with `status: "pending"` are generated.
- Already-generated sprites (`status: "generated"`) are skipped.
- Failed sprites keep `status: "failed"` for the agent to review.
- The spec file is updated after each sprite completes (so progress is preserved if interrupted).
- Side-scroller views auto-generate mirrored left copies.
- Isometric views generate 4 directional sprites (sw/ne/nw/se) from 1 API call via dual-sprite template + split + mirror.

### After generation

1. Read the updated spec to check for failures.
2. For any `"failed"` sprites, review the error messages in the CLI output.
3. To retry failed sprites, change their status back to `"pending"` and re-run.

## Examples

### Minimal — Single Character

```json
{
  "version": "1",
  "sprites": [
    { "name": "hero", "description": "brave hero with sword and shield", "status": "pending" }
  ]
}
```

Generates: `hero_front.png` (uses config's default view and size).

### Full Roster — Enemies + Bosses

```json
{
  "version": "1",
  "sprites": [
    {
      "name": "skeleton",
      "description": "undead skeleton warrior, cracked bones, rusty sword",
      "views": ["front"],
      "size": "32x32",
      "status": "pending"
    },
    {
      "name": "slime",
      "description": "translucent green slime, gelatinous blob with bubbles",
      "views": ["front"],
      "size": "32x32",
      "status": "pending"
    },
    {
      "name": "dragon-boss",
      "description": "massive red dragon, leathery wings, breathing fire",
      "views": ["front", "side-scroller"],
      "size": "128x128",
      "status": "pending"
    }
  ]
}
```

### Variant Example — Color Swaps

```json
{
  "version": "1",
  "sprites": [
    {
      "name": "knight",
      "description": "medieval knight in plate armor",
      "views": ["front", "side-scroller"],
      "size": "64x64",
      "variants": [
        { "slug": "golden", "description": "golden armor" },
        { "slug": "shadow", "description": "dark corrupted armor with purple glow" },
        { "slug": "ice", "description": "frost-covered armor with blue crystals" }
      ],
      "status": "pending"
    }
  ]
}
```

### Based-On Example — Matching Style

```json
{
  "version": "1",
  "sprites": [
    {
      "name": "knight",
      "description": "medieval knight, plate armor, longsword",
      "status": "pending"
    },
    {
      "name": "captain",
      "description": "elite captain, decorated silver armor, cape",
      "based_on": "knight",
      "status": "pending"
    },
    {
      "name": "squire",
      "description": "young squire, light leather armor, short sword",
      "based_on": "knight",
      "status": "pending"
    }
  ]
}
```

Place `knight` first so it generates before `captain` and `squire` (which depend on knight's reference image). Order matters! Dependencies must appear before dependents.

## Constraints

- Sprite names must be unique within the spec.
- `based_on` sprites must exist in the spec and appear before dependent sprites.
- Spec file is overwritten in-place during generation (to update statuses).
- Large batches (roughly 30+ sprites) may hit provider rate limits; consider splitting into multiple spec files if you see `429`/rate-limit errors.
