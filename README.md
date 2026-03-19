# Textpillar

Multi-agent research-driven content generation plugin for [Claude Code](https://claude.com/claude-code).

Textpillar researches, curates, validates web sources, and generates SEO-optimized blog posts, Gemini TTS-ready podcast scripts, and image generation prompts — all from a single command.

## Installation

Inside a Claude Code session, add the marketplace and install the plugin:

```
/plugin marketplace add felipe-venturini/textpillar
/plugin install textpillar@textpillar
```

Or use the interactive plugin browser:

```
/plugin
```

Then navigate to **Marketplaces** tab → add `felipe-venturini/textpillar` → **Discover** tab → install Textpillar.

## Quick Start

### Interactive mode (recommended for first use)

```
/textpillar
```

Textpillar will ask you questions one at a time to understand what you need.

### With a topic

```
/textpillar AI agents and autonomous coding in 2026
```

### With full JSON configuration

```
/textpillar {"topic": {"subject": "AI agents", "keywords": ["autonomous coding", "multi-agent systems"], "freshness": "7d"}, "outputs": {"blog": {"enabled": true}, "script": {"enabled": true, "hosts": [{"name": "Alex", "voice": "Kore", "personality": "Enthusiastic tech expert", "role": "lead"}, {"name": "Sam", "voice": "Puck", "personality": "Skeptical journalist", "role": "co-host"}]}}}
```

## What It Does

Textpillar runs a 5-phase pipeline:

1. **Research** — Searches the web with multiple reformulated queries, fetches content from URLs, RSS feeds, and APIs
2. **Curate** — Filters sources for bias, cross-references facts across outlets, deduplicates, ranks by credibility
3. **Validate** — Checks every URL is a real article (not a homepage or category page), verifies HTTP accessibility, confirms content matches claims
4. **Generate** — Produces requested outputs in parallel:
   - **Blog post** — SEO-optimized markdown with keyword density control
   - **Podcast script** — Natural dialogue in Gemini TTS API-ready JSON format
   - **Image prompts** — Detailed prompts for blog covers and podcast thumbnails
5. **Present** — Shows summary with file paths and statistics

## Outputs

All outputs are written to `./textpillar-output/` with date-based naming:

| File | Description |
|---|---|
| `2026-03-19-blog.md` | Markdown with YAML frontmatter (SEO metadata, sources, keyword density report) |
| `2026-03-19-script.json` | Gemini TTS API-ready JSON (send directly to `gemini-2.5-flash-preview-tts`) |
| `2026-03-19-image-prompt.json` | Image generation prompts with brand colors, dimensions, negative prompts |

## Configuration

See the [full input schema](docs/superpowers/specs/2026-03-19-textpillar-design.md#4-input-schema) in the design spec.

### Key options

| Option | Description | Default |
|---|---|---|
| `topic.subject` | What to research (REQUIRED) | — |
| `topic.freshness` | How recent sources should be | `24h` for news, `unlimited` for documentary |
| `topic.depth` | How many sources to find | `medium` (10-20) |
| `outputs.blog.enabled` | Generate a blog post | `true` |
| `outputs.script.enabled` | Generate a podcast script | `false` |
| `outputs.image_prompt.enabled` | Generate image prompts | `false` |
| `tone.language` | Output language (ISO code) | Conversation language |
| `tone.formality` | Writing style | `semi-formal` |
| `memory.project_id` | Group content under a project | `default` |

## Agent Architecture

| Agent | Model | Purpose |
|---|---|---|
| researcher | Haiku | Fast web search, content extraction |
| curator | Sonnet | Source quality analysis, bias detection |
| validator | Haiku | URL verification, content matching |
| blog-writer | Opus | SEO-optimized article generation |
| script-writer | Opus | Natural podcast dialogue generation |
| image-prompt-writer | Sonnet | Visual prompt crafting |

## Memory

Textpillar uses agent-level persistent memory to:
- **Avoid content repetition** — The curator remembers past topics and key facts
- **Reference past episodes** — The script writer naturally callbacks to previous podcasts
- **Maintain visual consistency** — The image prompt writer preserves brand style across runs

## Requirements

- [Claude Code](https://claude.com/claude-code) CLI
- `jq` installed (for HTTP validation hook)
- `curl` installed (for HTTP validation hook)

## License

MIT
