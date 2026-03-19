# Textpillar — Design Specification

**Date:** 2026-03-19
**Status:** Draft
**Author:** Felipe Venturini + Claude

---

## 1. Overview

Textpillar is a Claude Code plugin that orchestrates multi-agent content generation from web research. It coordinates specialized agents through a research-curate-validate-generate pipeline, producing blog posts, podcast scripts, and image generation prompts.

The plugin is designed to be:
- **Generic** — works for any topic, any client, any language
- **Portable** — runs as Claude Code skill (CLI) and will support API via Agent SDK
- **Published** — open-source on GitHub as a public Claude Code plugin

### 1.1 Use Cases

| Use Case | Freshness | Depth | Output |
|---|---|---|---|
| Daily news blog | Last 24h | Shallow (5-10 sources) | Blog + image prompt |
| Weekly podcast | Last 7 days | Medium (10-20 sources) | Script + thumbnail + blog |
| Documentary content | No limit | Deep (20-50+ sources) | Blog + script + images |
| Series (daily, Sun-Sun) | Last 24h per edition | Shallow-medium | Blog per day (manual invocation per day in v1) |

### 1.2 Key Principles

- **Source integrity** — URLs must be real articles (not homepages, not category pages), HTTP status warned by hook and validated by validator agent
- **Bias mitigation** — Cross-reference facts across 2+ independent sources
- **No repetition** — Agent memory prevents duplicate content across sessions
- **SEO without spam** — Keyword density controlled at 1.5-2.5%
- **Natural dialogue** — Podcast scripts sound human, not robotic
- **Visual consistency** — Blog covers and podcast thumbnails share visual language

---

## 2. Architecture

### 2.1 Component Map

```
textpillar/                              (plugin root)
├── .claude-plugin/
│   └── plugin.json                      (manifest)
├── skills/
│   └── textpillar/
│       ├── SKILL.md                     (orchestrator — single user entry point)
│       └── templates/
│           ├── blog-template.md
│           ├── script-template.md
│           └── image-prompt-template.md
├── agents/
│   ├── researcher.md
│   ├── curator.md
│   ├── validator.md
│   ├── blog-writer.md
│   ├── script-writer.md
│   └── image-prompt-writer.md
├── hooks/
│   └── hooks.json                       (HTTP 200 validation on WebFetch)
└── README.md
```

### 2.2 Runtime Directories

| Directory | Purpose | Lifecycle |
|---|---|---|
| `.textpillar-session/` | Intermediate session data (research, curated, validated JSONs) | Cleaned up by SessionEnd hook; gitignored |
| `./textpillar-output/` | Final outputs with date-based naming (e.g., `2026-03-19-blog.md`, `2026-03-19-script.json`) | Persists in user's working directory, no overwrites |
| `.claude/agent-memory-local/<agent-name>/` | Agent persistent memory (anti-repetition, visual consistency) | Persists across sessions |

### 2.3 Data Flow

```
User
  │
  ▼
/textpillar (SKILL.md — main context, interactive)
  │
  ├─ Phase 0: Input Parser
  │   ├─ JSON provided? → extract params, proceed
  │   └─ Missing params? → adaptive questions, one at a time
  │
  ├─ Phase 1: researcher agent (subagent, model: haiku)
  │   ├─ WebSearch (parallel queries, reformulated)
  │   ├─ WebFetch (user-provided URLs, RSS, APIs)
  │   └─ Writes .textpillar-session/01-research.json
  │
  ├─ Phase 2: curator agent (subagent, model: sonnet)
  │   ├─ Reads 01-research.json
  │   ├─ Checks agent memory for past content
  │   ├─ Cross-references, deduplicates, ranks by credibility
  │   └─ Writes .textpillar-session/02-curated.json
  │
  ├─ Phase 3: validator agent (subagent, model: haiku)
  │   ├─ Reads 02-curated.json
  │   ├─ Verifies URL patterns (rejects homepages, category pages)
  │   ├─ Confirms content matches extracted summaries
  │   └─ Writes .textpillar-session/03-validated.json
  │   (HTTP status warned by PreToolUse hook; validator agent makes final accept/reject decision)
  │
  ├─ Phase 4: Generation (PARALLEL subagents)
  │   ├─ blog-writer agent (model: opus) → ./textpillar-output/<date>-blog.md
  │   ├─ script-writer agent (model: opus) → ./textpillar-output/<date>-script.json
  │   └─ image-prompt-writer agent (model: sonnet) → ./textpillar-output/<date>-image-prompt.json
  │
  └─ Phase 5: Present results summary to user
```

---

## 3. Plugin Manifest

```json
{
  "name": "textpillar",
  "description": "Multi-agent research-driven content generation. Produces SEO-optimized blog posts, Gemini TTS-ready podcast scripts, and image generation prompts from curated web research.",
  "version": "1.0.0",
  "author": {
    "name": "Felipe Venturini"
  }
}
```

---

## 4. Input Schema

The user can provide a full JSON config via `$ARGUMENTS` or answer adaptive questions interactively.

Only `topic.subject` is required. Everything else has smart defaults or is asked interactively.

```json
{
  "topic": {
    "subject": "string (REQUIRED)",
    "keywords": ["string"],
    "scope": "news | documentary",
    "freshness": "24h | 7d | 30d | unlimited",
    "depth": "shallow | medium | deep"
  },

  "sources": {
    "urls": ["string (optional user-provided URLs)"],
    "local_files": ["string (optional local file paths)"],
    "rss_feeds": ["string (optional RSS feed URLs)"],
    "apis": ["string (optional API endpoints)"],
    "blacklist": ["string (domains to exclude)"],
    "whitelist": ["string (preferred domains)"]
  },

  "outputs": {
    "blog": {
      "enabled": true,
      "seo": {
        "target_keywords": ["string"],
        "keyword_density": "string (default: 1.5-2.5%)",
        "meta_description_length": 155,
        "custom_rules": ["string"]
      }
    },
    "script": {
      "enabled": false,
      "hosts": [
        {
          "name": "string",
          "voice": "string (Gemini voice name: Kore, Puck, Charon, etc.)",
          "personality": "string (description of personality and speaking style)",
          "role": "lead | co-host"
        }
      ],
      "tts_model": "gemini-2.5-flash-preview-tts | gemini-2.5-pro-preview-tts",
      "duration_target_minutes": 15,
      "reference_episodes": 5
    },
    "image_prompt": {
      "enabled": false,
      "style": {
        "brand_colors": ["string (hex colors)"],
        "visual_style": "string",
        "mood": "string",
        "format": "16:9 | 1:1 | 4:3"
      },
      "generate_for": ["blog_cover", "podcast_thumbnail"],
      "visual_consistency": true
    }
  },

  "tone": {
    "voice": "string (default: expert but accessible)",
    "formality": "formal | semi-formal | casual",
    "persona": "string",
    "audience": "string",
    "language": "string (ISO code, default: conversation language)",
    "complexity": "beginner | intermediate | advanced",
    "words_to_avoid": ["string"],
    "jargon_allowed": true,
    "reference_content": ["string (URLs of content the user likes)"]
  },

  "memory": {
    "project_id": "string (groups content under a project)",
    "avoid_repetition": true,
    "reference_past_episodes": true,
    "max_past_references": 5
  }
}
```

### 4.1 Smart Defaults

When parameters are missing, the orchestrator suggests contextual defaults:

| Parameter | Default Logic |
|---|---|
| `scope` | "news" if topic contains current events keywords, "documentary" otherwise |
| `freshness` | "24h" for news, "unlimited" for documentary |
| `depth` | "shallow" for daily content, "medium" for weekly, "deep" for documentary |
| `language` | Same as conversation language |
| `tone.voice` | "expert but accessible" |
| `tone.formality` | "semi-formal" |
| `keyword_density` | "1.5-2.5%" |
| `tts_model` | "gemini-2.5-flash-preview-tts" |

### 4.2 Adaptive Question Flow

When $ARGUMENTS is empty or incomplete:

1. "What topic should I research?" (always ask)
2. "What would you like me to generate? A) Blog B) Podcast C) Both D) All three" (always ask)
3. "How fresh should the sources be?" — suggest based on topic (contextual)
4. "Who is the target audience and what tone?" — suggest based on topic (contextual)
5. "What language for the final content?" — default to conversation language (contextual)
6. SEO keywords — only if blog enabled, suggest based on topic
7. Hosts config — only if podcast enabled
8. Visual identity — only if image prompts enabled

Each question is asked ONE AT A TIME with multiple choice when possible.

---

## 5. Agent Definitions

### 5.1 researcher.md

```yaml
---
name: researcher
description: Searches the web for sources on a given topic with configurable freshness, fetches content, and writes raw research data to session files
model: haiku
tools: WebSearch, WebFetch, Read, Write, Bash
maxTurns: 30
---
```

**System prompt responsibilities:**
- Receive topic, keywords, freshness window, source config
- Execute parallel web searches with reformulated queries (synonyms, related terms)
- Fetch and extract main content from each URL (ignore navigation, ads, footers)
- For user-provided URLs, RSS feeds, and APIs: fetch and extract
- Write structured output to `01-research.json`

**Output schema (01-research.json):**
```json
{
  "query": { "topic": "...", "keywords": [...], "freshness": "..." },
  "sources": [
    {
      "url": "string",
      "title": "string",
      "domain": "string",
      "published_date": "string (ISO 8601)",
      "content_excerpt": "string (first 500 words)",
      "full_content": "string",
      "source_type": "web_search | user_url | rss | api"
    }
  ],
  "total_found": 0,
  "search_queries_used": ["string"]
}
```

### 5.2 curator.md

```yaml
---
name: curator
description: Filters, deduplicates, and ranks research sources by relevance and credibility, checking for bias and cross-referencing facts against history
model: sonnet
tools: Read, Write, Glob, Grep
maxTurns: 20
memory: local
---
```

**System prompt responsibilities:**
- Read `01-research.json`
- Check agent memory for previously generated content on this topic/project
- Cross-reference: same fact reported by 2+ independent sources?
- Source diversity: not over-relying on a single outlet
- Deduplication: merge sources covering the same event/fact
- Assign credibility score (0-1) per source based on: cross-referencing count, domain reputation, content depth
- Flag sources that only appear once (single-source facts)
- Write `02-curated.json`

**Output schema (02-curated.json):**
```json
{
  "curated_sources": [
    {
      "url": "string",
      "title": "string",
      "domain": "string",
      "credibility_score": 0.0,
      "cross_referenced_by": ["string (URLs)"],
      "key_facts": ["string"],
      "relevance_rank": 0
    }
  ],
  "rejected_sources": [
    {
      "url": "string",
      "reason": "single_source | low_credibility | duplicate | off_topic"
    }
  ],
  "facts_summary": {
    "confirmed_facts": 0,
    "single_source_facts": 0,
    "total_unique_sources": 0
  },
  "repetition_check": {
    "similar_past_content_found": false,
    "past_content_ids": ["string"]
  }
}
```

### 5.3 validator.md

```yaml
---
name: validator
description: Validates curated sources by checking URL accessibility, confirming URLs are real articles not homepages or category pages, and verifying content matches summaries
model: haiku
tools: Read, Write, WebFetch, Bash
maxTurns: 20
---
```

**System prompt responsibilities:**
- Read `02-curated.json`
- For each URL: verify it's a specific article (not homepage, not category, not search results page)
- URL pattern validation: must have a path beyond the domain root, must not match category patterns (`/category/`, `/tag/`, `/topics/`)
- Content-summary match: fetch URL content and confirm it matches the extracted summary
- Generate rejection report with specific reasons
- Write `03-validated.json`

**Output schema (03-validated.json):**
```json
{
  "validated_sources": [
    {
      "url": "string",
      "title": "string",
      "credibility_score": 0.0,
      "key_facts": ["string"],
      "content_verified": true
    }
  ],
  "rejected_sources": [
    {
      "url": "string",
      "reason": "homepage | category_page | content_mismatch | http_error | paywall",
      "detail": "string"
    }
  ],
  "validation_summary": {
    "total_evaluated": 0,
    "approved": 0,
    "rejected": 0
  }
}
```

### 5.4 blog-writer.md

```yaml
---
name: blog-writer
description: Generates SEO-optimized blog articles from validated research with proper keyword density, meta descriptions, and structured headings
model: opus
tools: Read, Write
maxTurns: 15
---
```

**System prompt responsibilities:**
- Read `03-validated.json` + tone/SEO config from orchestrator prompt
- Generate article with proper heading hierarchy (H1, H2, H3)
- Control keyword density within configured range (default 1.5-2.5%)
- Write meta description within configured length (default 155 chars)
- Generate URL-friendly slug
- Calculate word count and reading time
- Cite sources naturally within the text
- Follow tone, persona, audience, and language settings
- Respect words_to_avoid list
- Output blog.md (content) with embedded YAML frontmatter (metadata)

**Output format (blog.md):**
```yaml
---
title: "Article Title"
slug: "article-title"
meta_description: "155 char description..."
keywords: ["keyword1", "keyword2"]
keyword_density_report:
  keyword1: "2.1%"
  keyword2: "1.8%"
word_count: 1850
reading_time_minutes: 8
sources:
  - title: "Source Title"
    url: "https://..."
    credibility_score: 0.92
generated_at: "2026-03-19T14:30:00Z"
project_id: "autopodcast-ai"
---

# Article Title

Article content here...
```

### 5.5 script-writer.md

```yaml
---
name: script-writer
description: Generates natural podcast scripts for 1-2 hosts with distinct personalities formatted as Gemini TTS API-ready JSON, referencing past episodes when relevant
model: opus
tools: Read, Write, Glob
maxTurns: 15
memory: local
---
```

**System prompt responsibilities:**
- Read `03-validated.json` + host config + tone config from orchestrator prompt
- Check agent memory for past episodes (last N, configurable)
- Generate natural dialogue between hosts respecting their personalities
- Reference past episodes naturally when relevant ("Last week we talked about...")
- Format output as Gemini TTS API-ready JSON (multiSpeakerVoiceConfig)
- Keep within duration target (estimate ~150 words/minute)
- Max 2 hosts (Gemini TTS limitation)
- Save episode summary to agent memory for future reference

**Output format (script.json):**
```json
{
  "metadata": {
    "episode_title": "string",
    "duration_estimate_minutes": 0,
    "word_count": 0,
    "hosts": ["string"],
    "sources_referenced": 0,
    "past_episodes_referenced": [
      {
        "episode_id": "string",
        "context": "string"
      }
    ],
    "generated_at": "string (ISO 8601)"
  },
  "gemini_tts_request": {
    "contents": [{
      "parts": [{
        "text": "TTS the following conversation between Host1 and Host2:\nHost1: ...\nHost2: ..."
      }]
    }],
    "generationConfig": {
      "responseModalities": ["AUDIO"],
      "speechConfig": {
        "multiSpeakerVoiceConfig": {
          "speakerVoiceConfigs": [
            {
              "speaker": "Host1",
              "voiceConfig": {
                "prebuiltVoiceConfig": { "voiceName": "Kore" }
              }
            },
            {
              "speaker": "Host2",
              "voiceConfig": {
                "prebuiltVoiceConfig": { "voiceName": "Puck" }
              }
            }
          ]
        }
      }
    }
  }
}
```

### 5.6 image-prompt-writer.md

```yaml
---
name: image-prompt-writer
description: Generates image generation prompts for blog covers and podcast thumbnails following brand visual identity guidelines with cross-format visual consistency
model: sonnet
tools: Read, Write
maxTurns: 10
memory: local
---
```

**System prompt responsibilities:**
- Read `03-validated.json` + visual identity config from orchestrator prompt
- Generate prompts for each requested output type (blog_cover, podcast_thumbnail)
- When both blog and podcast are requested, ensure visual consistency (same style, colors, mood)
- Include negative prompts to avoid unwanted elements
- Specify dimensions based on format
- Check agent memory for past visual styles (maintain brand consistency across sessions)
- Save visual style reference to agent memory

**Output format (image-prompt.json):**
```json
{
  "metadata": {
    "generated_for": ["blog_cover", "podcast_thumbnail"],
    "visual_consistency_id": "string",
    "generated_at": "string (ISO 8601)"
  },
  "prompts": {
    "blog_cover": {
      "prompt": "string (detailed generation prompt)",
      "negative_prompt": "string",
      "style_reference": {
        "colors": ["string (hex)"],
        "mood": "string"
      },
      "dimensions": "1792x1024"
    },
    "podcast_thumbnail": {
      "prompt": "string (detailed generation prompt)",
      "negative_prompt": "string",
      "style_reference": {
        "colors": ["string (hex)"],
        "mood": "string"
      },
      "dimensions": "1024x1024"
    }
  }
}
```

---

## 6. Orchestrator Skill (SKILL.md)

```markdown
---
name: textpillar
description: Multi-agent content generation from web research. Researches, curates, validates sources, then generates blog posts, podcast scripts, and image prompts.
argument-hint: [topic or JSON config]
hooks:
  SessionStart:
    - type: command
      command: "mkdir -p ./textpillar-output && mkdir -p .textpillar-session"
  SessionEnd:
    - type: command
      command: "rm -rf .textpillar-session"
---
```

The full SKILL.md body content will be written during implementation based on the data flow in Section 2.3 and the behavioral rules below. No separate reference document exists — this spec is the single source of truth.

**Agent invocation:** The orchestrator describes what needs to be done in each phase. Claude automatically delegates to the correct agent based on the agent's `description` field matching the task. No special syntax is needed — Claude reads agent descriptions and decides which to invoke.

**Key behaviors:**
- Runs in main conversation context (no `context: fork`) to support interactive questions
- Describes tasks naturally; Claude matches to agents via description field
- Writes intermediate data to `.textpillar-session/` (project-local, gitignored)
- Writes final outputs to `./textpillar-output/`
- Presents summary with file paths and stats at completion

---

## 7. Hooks

### 7.1 HTTP 200 Validation (hooks/hooks.json)

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "WebFetch",
        "hooks": [
          {
            "type": "command",
            "command": "URL=$(jq -r '.tool_input.url // empty'); if [ -n \"$URL\" ]; then STATUS=$(curl -sI -o /dev/null -w '%{http_code}' --max-time 5 \"$URL\"); if [ \"$STATUS\" -lt 200 ] || [ \"$STATUS\" -ge 400 ]; then echo \"WARNING: URL $URL returned HTTP $STATUS\" >&2; fi; fi; exit 0"
          }
        ]
      }
    ]
  }
}
```

### 7.2 Skill-scoped hooks (in SKILL.md frontmatter)

- `SessionStart`: Creates `./textpillar-output/` and `.textpillar-session/` directories
- `SessionEnd`: Cleans up `.textpillar-session/`

---

## 8. Agent Memory Strategy

| Agent | Memory | Purpose |
|---|---|---|
| researcher | none | Stateless — each search is independent |
| curator | `memory: local` | Remembers past content to prevent repetition |
| validator | none | Stateless — each validation is independent |
| blog-writer | none | Stateless — tone/style come from config each time |
| script-writer | `memory: local` | Remembers past episodes for natural references |
| image-prompt-writer | `memory: local` | Remembers visual styles for brand consistency |

Memory is stored at `.claude/agent-memory-local/<agent-name>/` and managed automatically by Claude Code. Each agent with memory enabled has a `MEMORY.md` index (first 200 lines auto-loaded into context).

---

## 9. Gemini TTS Voice Reference

Available voices for podcast script configuration:

| Voice | Personality | Voice | Personality |
|---|---|---|---|
| Zephyr | Bright | Aoede | Breezy |
| Puck | Upbeat | Callirrhoe | Easy-going |
| Charon | Informative | Umbriel | Easy-going |
| Kore | Firm | Algieba | Smooth |
| Fenrir | Excitable | Despina | Smooth |
| Leda | Youthful | Erinome | Clear |
| Orus | Firm | Algenib | Gravelly |
| Autonoe | Bright | Rasalgethi | Informative |
| Enceladus | Breathy | Laomedeia | Upbeat |
| Iapetus | Clear | Achernar | Soft |
| Alnilam | Firm | Schedar | Even |
| Gacrux | Mature | Pulcherrima | Forward |
| Achird | Friendly | Zubenelgenubi | Casual |
| Vindemiatrix | Gentle | Sadachbia | Lively |
| Sadaltager | Knowledgeable | Sulafat | Warm |

**Constraint:** Maximum 2 speakers in multi-speaker mode.
**Supported languages:** 73+ including pt-BR, en, es, fr, de, ja, ko, zh.

---

## 10. Future Considerations (not in v1)

- **Agent SDK port** — Core logic extracted for programmatic API access
- **3+ podcast hosts** — Segmented TTS requests when Gemini lifts the 2-speaker limit
- **Real-time fact checking** — Integration with fact-checking APIs
- **Content scheduling** — Cron-based generation (daily, weekly)
- **Analytics feedback loop** — SEO performance data feeding back into content strategy
- **Custom MCP servers** — Specialized research tools (academic papers, patents)
