---
name: asset-forge
description: Generate video game sprites and assets via the aforge CLI. Use when the user wants to create pixel art characters, game sprites, or any 2D game asset. Triggers on requests like "create a warrior sprite", "generate pixel art for my game", "make a character sprite", or mentions of aforge/AssetForge.
version: 0.1.0
---

# Asset Forge — Agent Skill

AssetForge (`aforge`) is a CLI tool for generating game sprites via AI image providers. The CLI orchestrates calls to external image generation APIs (OpenRouter). The agent's job is to translate natural language requests into `aforge` commands.

## Prerequisites

- Node.js >= 18
- `aforge` installed globally (`npm install -g @pokoion/assetforge`)
- OpenRouter API key as `OPENROUTER_API_KEY` in `.env` (must be created manually or by agent)
- `.env` must be in `.gitignore`

## CLI Overview

```
aforge init                         # Set up project config (wizard or --from-json)
aforge sprite init                  # Set up sprite conventions (wizard or --from-json)
aforge sprite <name> [description]  # Generate a sprite
aforge config                       # Manage provider/model settings
aforge config --list                # List available providers and models
aforge pixelize <image> --size WxH  # Convert image to pixel art (local, no API)
aforge palette extract <image>      # Extract color palette from image (local)
aforge export                       # Export assets to folder (excludes raw/history)
aforge export --zip                 # Export assets as ZIP archive
aforge doctor                       # Check environment health (config, API, deps)
aforge --version                    # Show version
```

## Command Reference

### `aforge init` — Project Setup

Sets up `assetforge.config.json`. Does NOT create `.env`, `.gitignore`, or `assets/` folders — those must be created separately.

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
- `model` — model slug (e.g. `google/gemini-3.1-flash-image-preview`, `openai/gpt-5-image-mini`)

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
| `--view <view>` | Camera: `front`, `top-down`, `side-scroller`, `isometric` |
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
- `--list` — List all available providers and models (recommended: `google/gemini-3.1-flash-image-preview`)
- `--provider [name]` — Set provider: `openrouter`, `mock`
- `--model [name]` — Set model for current provider

Run without flags for interactive wizard. Run `aforge config --list` to see all options.

## Workflows

### MANDATORY Flow — Front First (Design Checkpoint)

**⚠️ RULE: The agent MUST ALWAYS generate the front view first, show it to the user, and WAIT for explicit approval before generating any other views (side-scroller, top-down, isometric, etc.). This is NOT optional.**

**The agent MUST NOT generate multiple views in one go. Each view requires user confirmation.**

#### Flow (strictly enforced):

```
Step 1: ALWAYS generate front view first — regardless of what view the user requested
aforge sprite knight "heavy plate armor, silver sword, blue cape" --view front
→ Generates knight_front.png
→ Show the result to the user
→ ASK: "Here's the front view. Are you happy with the design? Would you like any changes before I generate other views?"

Step 2: WAIT for user response
├── User wants changes → apply edits (--edit or --force) and show again
│   → Loop until user approves
├── User approves → proceed to Step 3
└── User says "skip other views" → STOP, done

Step 3: ASK which additional views they want
→ "Which other views would you like me to generate? Options: side-scroller, top-down, isometric"
→ WAIT for user response
→ Generate ONLY the views the user explicitly requests

Step 4: Generate each additional view ONE AT A TIME, showing each result
aforge sprite knight --view side-scroller
→ Show result, ask: "Happy with this view?"
→ Wait for confirmation before next view
```

#### Why this is mandatory:
- **Cost control** — prevents wasting API calls on a design the user doesn't like
- **Consistency** — all subsequent views use the approved front as reference
- **User control** — the user decides what gets generated, not the agent

#### What the agent must NEVER do:
- ❌ Generate front + side-scroller + top-down in one batch without asking
- ❌ Generate isometric views without showing front first
- ❌ Skip the approval step ("I'll generate all views now")
- ❌ Assume the user wants all views listed in a spec without confirming

---

### First Sprite (New Character)

```
aforge sprite knight "heavy plate armor, silver sword, blue cape"
```

This creates:
- `assets/sprites/knight/knight.json` (descriptor)
- `assets/sprites/knight/knight_front.png` (generated image, magenta background removed)
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

### Pixelize (Local, No API)

Convert any image to pixel art using nearest-neighbor downsampling:

```
aforge pixelize concept.png --size 32x32
aforge pixelize sprite.png --size 16x16 --upscale 4    # 16x16 pixels → 64x64 display
aforge pixelize photo.jpg --size 64x64 --output out.png
```

Supported formats: `.png`, `.jpg`, `.jpeg`, `.webp`

### Palette Extract (Local, No API)

Extract dominant colors from a reference image and update config:

```
aforge palette extract reference.png                    # Extract 8 colors (default)
aforge palette extract reference.png --num-colors 16   # Extract 16 colors
aforge palette extract ref.png --output-palette pal.json  # Also save as JSON
```

Updates `color_palette_description` in `assetforge.config.json` automatically.

### Doctor (Health Check)

```
aforge doctor
```

Checks: config exists, API key present, provider reachable, sharp installed, output directory valid.

### Export (Local, No API)

Export all game-ready assets to a folder or ZIP, preserving the sprite folder structure. Excludes raw provider output, JSON descriptors, and history files.

```
aforge export                              # Export to ./export/ folder
aforge export --output dist/sprites        # Export to custom folder
aforge export --zip                        # Export as ./export.zip
aforge export --zip --output game.zip      # Export as custom ZIP path
```

**Flags:**

| Flag | Default | Purpose |
|------|---------|---------|
| `--output <path>` | `./export` | Output directory or ZIP file path |
| `--zip` | false | Create ZIP archive instead of folder |

**What gets exported:**
- All sprite PNGs (front, side, top-down, isometric)
- All variant PNGs
- Portraits (if generated)

**What is excluded:**
- `*_raw.png` — raw provider output before post-processing
- `*.json` — descriptor files (internal metadata)
- `history/` — old versions from edits/regenerations

**Output structure** (mirrors assets/ without excluded files):
```
export/
└── sprites/
    ├── knight/
    │   ├── knight_front.png
    │   ├── knight_side-right.png
    │   ├── knight_side-left.png
    │   └── variants/
    │       └── golden/
    │           └── knight_golden_front.png
    └── goblin/
        └── goblin_front.png
```

## Troubleshooting

| Error | Fix |
|-------|-----|
| "No config found" | `asset-forge-init` skill handles full setup |
| "No sprites config found" | `asset-forge-init` skill handles full setup |
| "No descriptor found for 'X'" | Provide description: `aforge sprite X "description"` |
| "File already exists" | Add `--force` to overwrite |
| "Provider returned error" | Check `OPENROUTER_API_KEY` in `.env`, check model availability |
| "JSON cannot be passed inline" | Write JSON to a file first, then use `--from-json @filename.json` |
| "Register aforge" | Run `npm link` in assetforge directory if using local dev |
| "aforge not recognized" | Run `npm run build` then `npm link` again |

## Setup

If no `assetforge.config.json` exists yet, the `asset-forge-init` skill handles full project setup including API key, .env, and sprite conventions.

## Environment

Create a `.env` file in your project root. The agent MUST do this during setup — `aforge init` does NOT create `.env`.

```
OPENROUTER_API_KEY=sk-or-...
```

Also add `.env` and `.aforge-debug/` to `.gitignore`.

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