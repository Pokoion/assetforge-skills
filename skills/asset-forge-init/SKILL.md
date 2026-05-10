---
name: asset-forge-init
description: Set up AssetForge project and sprite configuration. Use when the user asks to initialize aforge, set up asset generation for a game project, or configure sprite conventions. Triggers on "init aforge", "setup assetforge", "configure game assets", or "initialize sprite settings".
version: 0.1.0
---

# Asset Forge — Init Skill

Sets up project configuration (`assetforge.config.json`) and sprite conventions (`assets/sprites/sprites.json`) for AssetForge.

## Decision Tree

```
Is the user setting up a brand new project?
├── YES → aforge init (project config)
│         Then: aforge sprite init (sprite conventions)
├── NO, just adjusting config → write JSON to file → aforge init --from-json @file
└── NO, just adjusting sprite notes → write JSON to file → aforge sprite init --from-json @file
```

## Wizard Mode (Interactive)

### `aforge init`

Interactive wizard asks 7 questions:

1. **Project name** — Your game's name (e.g. "Dungeon Quest")
2. **Art style** — Choose from:
   - `pixel-art-retro` — 8/16-bit era (e.g. NES, SNES, Game Boy)
   - `pixel-art-modern` — Modern pixel art (e.g. Celeste, Stardew Valley, Dead Cells)
   - `flat-vector` — Clean vector/flat illustration style
   - `painterly` — Hand-painted look
   - `other` — Custom prompt
3. **Base prompt** — Core art direction text sent to image models (e.g. "pixel art, game sprite, dark fantasy")
4. **Game context** — Narrative description of your game world (optional). Helps the AI generate thematically consistent assets.
5. **Game genre** — Genre tag (optional): "RPG", "Platformer", "Roguelike", "Metroidvania", "Action", "Strategy", etc.
6. **Default view** — Camera angle used when `--view` not specified: `front`, `top-down`, `side-scroller`
7. **Default size** — Output resolution in `WxH` format: `32x32`, `64x64`, `128x128`, etc.

### `aforge sprite init`

Interactive wizard asks 1 question:

1. **Sprite conventions** — Notes applied to ALL sprites in this project. Examples:
   - "dark outlines, single figure centered in frame"
   - "no weapons shown, clean silhouette"
   - "fantasy RPG style, glowing elements"

These notes appear in every sprite generation prompts under `## SPRITE NOTES`.

## `--from-json` Mode (Programmatic / LLM-Guided)

### CRITICAL: How to pass JSON

**NEVER pass JSON inline on the command line.** Shell escaping will corrupt it. Always write JSON to a file first, then use `@filepath`.

```
Step 1: Write JSON to a file
Step 2: Run: aforge init --from-json @filename.json
```

### `aforge init --from-json`

#### PowerShell (Windows)
```powershell
Set-Content -Path .aforge-init.json -Value '{"project":{"name":"my-game"},"game":{"context":"A dark fantasy RPG","genre":"RPG"},"style":{"visual":"pixel-art-retro","base_prompt":"pixel art, game character sprite, black pixel outline","color_palette_description":"warm earthy tones with gold accents","lighting":"top-down dramatic lighting","detail_level":"high detail, 16-bit era"},"view":"front","size":"32x32","provider":"openrouter","model":"openai/gpt-5-image-mini"}'
aforge init --from-json @.aforge-init.json
```

#### Bash (Linux / macOS)
```bash
cat > .aforge-init.json << 'EOF'
{"project":{"name":"my-game"},"game":{"context":"A dark fantasy RPG","genre":"RPG"},"style":{"visual":"pixel-art-retro","base_prompt":"pixel art, game character sprite, black pixel outline","color_palette_description":"warm earthy tones with gold accents","lighting":"top-down dramatic lighting","detail_level":"high detail, 16-bit era"},"view":"front","size":"32x32","provider":"openrouter","model":"openai/gpt-5-image-mini"}
EOF
aforge init --from-json @.aforge-init.json
```

**Required fields:** `project.name`, `style.visual`, `style.base_prompt`, `view`, `size`

**Defaulted fields (can omit):** `game`, `provider`, `model`, `style.color_palette_description`, `style.color_palette_reference`, `style.lighting`, `style.detail_level`

### `aforge sprite init --from-json`

#### PowerShell (Windows)
```powershell
Set-Content -Path .aforge-sprite-init.json -Value '{"notes":"dark outlines, single figure centered"}'
aforge sprite init --from-json @.aforge-sprite-init.json
```

#### Bash (Linux / macOS)
```bash
echo '{"notes":"dark outlines, single figure centered"}' > .aforge-sprite-init.json
aforge sprite init --from-json @.aforge-sprite-init.json
```

**Required fields:** none (notes defaults to `""`)

## Field Reference

### `style.visual` (enum)

| Value | Description | Best For |
|-------|------------|----------|
| `pixel-art-retro` | 8/16-bit era, constrained palette | Platformers, retro RPGs |
| `pixel-art-modern` | High-res pixel art, smooth shading | Modern indie games |
| `flat-vector` | Clean lines, flat colors | Mobile games, UI avatars |
| `painterly` | Hand-painted, textured | Fantasy RPGs, card games |
| `other` | Custom base_prompt drives style | Experimental, mixed styles |

### `view` (enum)

| Value | Description |
|-------|-------------|
| `front` | Subject facing viewer, full body, symmetrical |
| `top-down` | Overhead perspective, bird's eye view |
| `side-scroller` | Profile facing right, platformer framing |
| `isometric` | 26° angle, 2:1 pixel ratio, 4 directions (sw/ne/nw/se) |

### `size` format

Always `WxH` with lowercase x: `32x32`, `64x64`, `128x128`. Represents the final output resolution after post-processing.

### `provider` (enum)

| Value | Description |
|-------|-------------|
| `openrouter` | Uses OpenRouter API (recommended, multi-model BYOK) |
| `mock` | Test provider — returns fixed test images |

### Common Models

| Model | Notes |
|-------|-------|
| `openai/gpt-5-image-mini` | Fast, OpenRouter routed (recommended) |

## Agent Workflow

### Setting up from scratch

```
1. Ask user about their game (genre, art style, resolution)
2. Build config JSON
3. Write JSON to .aforge-init.json
4. Run: aforge init --from-json @.aforge-init.json
5. Ask about sprite conventions (outlines, silhouette, etc.)
6. Write JSON to .aforge-sprite-init.json
7. Run: aforge sprite init --from-json @.aforge-sprite-init.json
8. Confirm: assetforge.config.json and assets/sprites/sprites.json created
9. Ready: user can now run aforge sprite <name> "description"
```

### Quick setup (defaults)

When user says "set up aforge with defaults" or "just use pixel art":

```powershell
Set-Content -Path .aforge-init.json -Value '{"project":{"name":"my-game"},"style":{"visual":"pixel-art-retro","base_prompt":"pixel art, game sprite, black pixel outline"},"view":"front","size":"32x32"}'
aforge init --from-json @.aforge-init.json
```

### Edge cases

- **Config already exists**: Tell user. Use `--from-json` to overwrite only if explicitly requested (delete config first, then re-run init).
- **User wants custom resolution**: Ask for WxH. Common sizes: 16x16, 32x32, 64x64, 128x128.
- **User unsure about style**: Default to `pixel-art-retro` — it's the most common for game sprites.
- **No API key yet**: Config can still be created. User needs to set `OPENROUTER_API_KEY` in `.env` before any generation.

## Provider Configuration

After init, the user can change provider/model later:

```
aforge config --provider openrouter
aforge config --model openai/gpt-5-image-mini
aforge config                                      # Show current settings
```
