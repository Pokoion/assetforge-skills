# AssetForge Skills

Agent skills for [AssetForge](https://github.com/Pokoion/assetforge) — the CLI for generating game assets from your IDE agent.

## Install

```bash
npx skills add Pokoion/assetforge-skills
```

## Skills

| Skill | Description |
|-------|-------------|
| `assetforge` | Core skill — how to use the aforge CLI to generate sprites and game assets |
| `assetforge-init` | Guides the agent through `aforge init --llm` setup |
| `assetforge-image-prompt` | Teaches the agent how to build effective image prompts for each asset type |

## Requirements

- [AssetForge CLI](https://github.com/Pokoion/assetforge) installed: `npm install -g assetforge`
- An OpenRouter API key in your `.env`

## Usage

Once installed, your IDE agent (Claude Code, Cursor, Copilot...) will automatically use these skills when you ask it to create game assets.

```bash
# Example — agent picks up the skill and runs:
aforge sprite warrior "heavily armored medieval warrior, red cape"
```