---
name: asset-forge
description: Generate video game sprites and assets via the aforge CLI. Use when the user wants to create pixel art characters, game sprites, or any 2D game asset. Triggers on requests like "create a warrior sprite", "generate pixel art for my game", "make a character sprite", or mentions of aforge/AssetForge.
version: 0.1.0
---

# Asset Forge — Agent Skill

AssetForge (`aforge`) is a CLI tool for generating game sprites via AI image providers. The CLI orchestrates calls to external image generation APIs (OpenRouter). The agent's job is to translate natural language requests into `aforge` commands.

## Prerequisites

- Node.js >= 18
- `aforge` installed globally (`npm install -g assetforge`) or locally (`npx aforge`)
- OpenRouter API key set as `OPENROUTER_API_KEY` env var

## CLI Overview

```
aforge init                         # Set up project config (wizard or --from-json)
aforge sprite init                  # Set up sprite conventions (wizard or --from-json)
aforge sprite <name> [description]  # Generate a sprite
aforge config                       # Manage provider/model settings
aforge config --list                # List available providers and models
aforge --version                    # Show version
```

## Command Reference

### `aforge init` — Project Setup

Sets up `assetforge.config.json` and `assets/sprites/` folder structure.

**Flags:**
- `--from-json @file` — Skip wizard, provide config via `@filepath`. Write JSON to a file first, then pass `@filename`. **Inline JSON is not supported.**
- `--llm` — Print LLM-guided setup prompt (for IDE agent consumption, not CLI use).

**Config fields:**
- `project.name` — game project name
- `game.context` — narrative/world context for the game
- `game.genre` — genre (RPG, platformer, etc.)
- `style.visual` — one of: `pixel-art-retro`, `pixel-art-modern`, `flat-vector`, `painterly`, `other`
- `style.base_prompt` — core art direction (e.g. "pixel art, game sprite, black pixel outline")
- `style.color_palette_description` — palette description
- `style.color_palette_reference` — path to reference palette image
- `style.lighting` — lighting style
- `style.detail_level` — detail level
- `view` — default camera: `front`, `top-down`, `side-scroller`
- `size` — default output size: `WxH` (e.g. `32x32`, `64x64`)
- `provider` — `openrouter` or `mock`
- `model` — model slug (e.g. `openai/gpt-5-image-mini`)

### `aforge sprite init` — Sprite Conventions

Creates `assets/sprites/sprites.json` with global sprite notes.

**Flags:**
- `--from-json @file` — Provide sprite notes via `@filepath`. Write JSON to a file first.
- `--llm` — Print LLM-guided prompt.

If no flags: runs interactive wizard asking for sprite conventions notes.

### `aforge sprite <name> [description]` — Generate Sprite

The core command. Generates sprite images from a description.

**Arguments:**
- `<name>` — Asset identifier (slug, lowercase, e.g. `warrior`, `orc-grunt`)
- `[description]` — Character description (only needed for first generation)

**Flags:**

| Flag | Purpose |
|------|---------|
| `--view <view>` | Camera: `front`, `top-down`, `side-scroller` |
| `--variant <slug>` | Named variation of same character |
| `--based-on <name>` | New character based on existing one's visuals |
| `--reference <path>` | Explicit reference image path |
| `--update-desc <text>` | Update description only, no generation |
| `--edit <text>` | Re-generate using existing image as reference + edit instruction |
| `--force` | Overwrite existing files (moves old to `history/`) |
| `--from-spec <path>` | Batch-generate sprites from `sprites.spec.json` |

**Important:** If `<name>` is `"init"`, runs `aforge sprite init` instead.

### `aforge config` — Provider Configuration

Manage image provider and model without editing JSON.

**Flags:**
- `--list` — List all available providers and models (recommended: `openai/gpt-5-image-mini`)
- `--provider [name]` — Set provider: `openrouter`, `mock`
- `--model [name]` — Set model for current provider

Run without flags for interactive wizard. Run `aforge config --list` to see all options.

## Workflows

### First Sprite (New Character)

```
aforge sprite knight "heavy plate armor, silver sword, blue cape"
```

This creates:
- `assets/sprites/knight/knight.json` (descriptor)
- `assets/sprites/knight/knight_front.png` (generated image, magenta background)
- Adds entry to `assets/manifest.json`

After first generation, the descriptor has `reference_image` set. All subsequent views/variants use it as reference.

### Additional Views

```
aforge sprite knight --view side-scroller
aforge sprite knight --view top-down
```

Each generates new view using first generation as reference. Side-scroller auto-mirrors (right + left).

### Variants (Same Character, Different Gear)

```
aforge sprite knight --variant golden "golden armor, same knight"
```

Creates: `assets/sprites/knight/variants/golden/knight_golden_front.png`

Multiple variants:
```
aforge sprite knight --variant golden "golden armor" --variant shadow "dark armor, spectral glow"
```

### Based-On (New Character, Similar Style)

```
aforge sprite captain "elite captain, silver armor" --based-on knight
```

New character inherits knight's reference image as visual anchor but uses its own description. Requires `--force` if descriptor exists.

### Editing (Regenerate with Modification)

```
aforge sprite knight --view side-scroller --edit "wider stance"
```

Uses existing image as reference, adds edit instruction. Old files (sprite, portrait, raw) moved to `history/`.

### Batch Generation (Spec File)

```
aforge sprite --from-spec sprites.spec.json
```

See `sprite-spec` skill for the spec format.

## Troubleshooting

| Error | Fix |
|-------|-----|
| "No config found" | Run `aforge init` first |
| "No sprites config found" | Run `aforge sprite init` first |
| "No descriptor found for 'X'" | Provide description: `aforge sprite X "description"` |
| "File already exists" | Add `--force` to overwrite |
| "Provider returned error" | Check API key in `.env`, check model availability |
| "Register aforge" | Run `npm link` in the assetforge directory if using local dev |
| "aforge not recognized" | Run `npm run build` then `npm link` again |
| "JSON cannot be passed inline" | Write JSON to a file first, then use `--from-json @filename.json` |

## Environment

Create a `.env` file in your project root:

```
OPENROUTER_API_KEY=sk-or-...
```

## Output Structure

```
assets/
├── manifest.json
├── sprites/
│   ├── sprites.json
│   └── {name}/
│       ├── {name}.json
│       ├── {name}_front.png
│       ├── {name}_top-down.png
│       ├── {name}_side-right.png
│       ├── {name}_side-left.png
│       ├── history/
│       │   ├── {name}_{view}_{timestamp}.png
│       │   ├── {name}_{view}_{timestamp}_portrait.png
│       │   └── {name}_{view}_{timestamp}_raw.png
│       └── variants/
│           └── {slug}/
│               └── {name}_{slug}_{view}.png
```

## Notes

- Sprites are generated on solid magenta (`#FF00FF`) background. Post-processing removes the background automatically.
- Side views auto-generate a mirrored left copy (no extra API call).
- Descriptors auto-update `reference_image` and `views_generated` after each generation.
- Debug mode: set `AFORGE_DEBUG=1` to save intermediate images and API request dumps to `.aforge-debug/`.
- All sprites include a black pixel outline by default (specified in `base_prompt` and technical requirements).
