# Textpillar Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the Textpillar Claude Code plugin — a multi-agent content generation system that researches, curates, validates, and generates blog posts, podcast scripts, and image prompts from web sources.

**Architecture:** A Claude Code plugin with one orchestrator skill (`/textpillar`), six specialized agents (researcher, curator, validator, blog-writer, script-writer, image-prompt-writer), HTTP validation hooks, and three content templates. Agents communicate via intermediate JSON files in `.textpillar-session/`, with final outputs written to `./textpillar-output/`.

**Tech Stack:** Claude Code plugin system (skills, agents, hooks), Markdown/YAML frontmatter, JSON schemas, Gemini TTS API format.

**Spec:** `docs/superpowers/specs/2026-03-19-textpillar-design.md`

---

## File Structure

```
textpillar/                                  (plugin root = repo root)
├── .claude-plugin/
│   └── plugin.json                          # Plugin manifest
├── skills/
│   └── textpillar/
│       ├── SKILL.md                         # Orchestrator skill (user entry point)
│       └── templates/
│           ├── blog-template.md             # Blog output format reference
│           ├── script-template.md           # Gemini TTS script format reference
│           └── image-prompt-template.md     # Image prompt format reference
├── agents/
│   ├── researcher.md                        # Web research agent
│   ├── curator.md                           # Source curation agent
│   ├── validator.md                         # Source validation agent
│   ├── blog-writer.md                       # Blog generation agent
│   ├── script-writer.md                     # Podcast script generation agent
│   └── image-prompt-writer.md               # Image prompt generation agent
├── hooks/
│   └── hooks.json                           # PreToolUse HTTP validation hook
├── .gitignore                               # Ignore session data
└── README.md                                # Installation and usage docs
```

---

### Task 1: Plugin Scaffold

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `.gitignore`

- [ ] **Step 1: Create plugin manifest**

Create `.claude-plugin/plugin.json`:

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

- [ ] **Step 2: Create .gitignore**

Create `.gitignore`:

```
# Textpillar session data (intermediate files)
.textpillar-session/

# Output directory (user-generated content)
textpillar-output/

# Agent memory (local, not version controlled)
.claude/agent-memory-local/

# OS files
.DS_Store
```

- [ ] **Step 3: Commit scaffold**

```bash
git add .claude-plugin/plugin.json .gitignore
git commit -m "feat: add plugin manifest and gitignore"
```

---

### Task 2: HTTP Validation Hook

**Files:**
- Create: `hooks/hooks.json`

- [ ] **Step 1: Create hooks directory and hooks.json**

Create `hooks/hooks.json`:

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

- [ ] **Step 2: Verify hook JSON is valid**

Run: `cat hooks/hooks.json | jq .`
Expected: Valid JSON output, no parse errors.

- [ ] **Step 3: Commit hook**

```bash
git add hooks/hooks.json
git commit -m "feat: add HTTP status validation hook for WebFetch"
```

---

### Task 3: Content Templates

**Files:**
- Create: `skills/textpillar/templates/blog-template.md`
- Create: `skills/textpillar/templates/script-template.md`
- Create: `skills/textpillar/templates/image-prompt-template.md`

- [ ] **Step 1: Create blog template**

Create `skills/textpillar/templates/blog-template.md`:

```markdown
# Blog Output Format Reference

The blog-writer agent MUST produce a markdown file with YAML frontmatter following this exact structure.

## Format

The output file is a standard markdown file with YAML frontmatter delimited by `---`.

### Frontmatter fields (REQUIRED):

| Field | Type | Description |
|---|---|---|
| title | string | Article title, SEO-optimized |
| slug | string | URL-friendly version of title (lowercase, hyphens, no special chars) |
| meta_description | string | Max 155 characters, includes primary keyword |
| keywords | string[] | Target keywords used in the article |
| keyword_density_report | object | Each keyword mapped to its density percentage |
| word_count | number | Total word count of article body (excluding frontmatter) |
| reading_time_minutes | number | Estimated at 250 words/minute |
| sources | object[] | Each with title, url, credibility_score |
| generated_at | string | ISO 8601 timestamp |
| project_id | string | From input config memory.project_id |

### Body structure:

1. Single H1 heading (matches title)
2. Introduction paragraph (hook + thesis, ~100 words)
3. H2 sections for main topics (3-6 sections)
4. H3 subsections where needed
5. Conclusion with key takeaways
6. Sources section with linked references

### SEO rules:

- Primary keyword in: title, H1, first paragraph, meta_description, at least one H2
- Keyword density: 1.5-2.5% (configurable via input)
- No keyword stuffing — keywords must read naturally
- Use semantic variations and related terms
- Internal linking suggestions as HTML comments where relevant

### Example:

```yaml
---
title: "AI Agents in 2026: How Autonomous Code Is Changing Development"
slug: "ai-agents-2026-autonomous-code-changing-development"
meta_description: "Discover how AI agents are transforming software development in 2026 with autonomous coding, multi-agent systems, and intelligent workflows."
keywords: ["AI agents", "autonomous coding", "multi-agent systems"]
keyword_density_report:
  "AI agents": "2.1%"
  "autonomous coding": "1.7%"
word_count: 1850
reading_time_minutes: 8
sources:
  - title: "OpenAI Launches Agent Framework"
    url: "https://techcrunch.com/2026/03/18/openai-agent-framework"
    credibility_score: 0.92
generated_at: "2026-03-19T14:30:00Z"
project_id: "autopodcast-ai"
---

# AI Agents in 2026: How Autonomous Code Is Changing Development

The landscape of software development is shifting...
```
```

- [ ] **Step 2: Create script template**

Create `skills/textpillar/templates/script-template.md`:

```markdown
# Podcast Script Output Format Reference

The script-writer agent MUST produce a JSON file following this exact structure, ready to be sent directly to the Gemini TTS API.

## Format

The output is a single JSON file containing metadata and a Gemini TTS API request payload.

### Top-level structure:

```json
{
  "metadata": { ... },
  "gemini_tts_request": { ... }
}
```

### metadata fields (REQUIRED):

| Field | Type | Description |
|---|---|---|
| episode_title | string | Descriptive episode title |
| duration_estimate_minutes | number | Based on ~150 words/minute |
| word_count | number | Total words in the script dialogue |
| hosts | string[] | Names of hosts in this episode |
| sources_referenced | number | Count of validated sources used |
| past_episodes_referenced | object[] | Each with episode_id and context string |
| generated_at | string | ISO 8601 timestamp |

### gemini_tts_request structure:

Must follow the Gemini TTS multi-speaker API format exactly:

- `contents[0].parts[0].text`: The full script in transcript format
  - Format: `"TTS the following conversation between {Host1} and {Host2}:\n{Host1}: line\n{Host2}: line\n..."`
  - Speaker names in text MUST match speaker names in speakerVoiceConfigs
- `generationConfig.responseModalities`: Always `["AUDIO"]`
- `generationConfig.speechConfig.multiSpeakerVoiceConfig.speakerVoiceConfigs`: Array of speaker configs
  - Each has: `speaker` (name), `voiceConfig.prebuiltVoiceConfig.voiceName` (Gemini voice)

### Script writing rules:

- Maximum 2 hosts (Gemini TTS limitation)
- Each host has a distinct personality defined in the input config
- Dialogue must sound natural — use contractions, interruptions, reactions
- Include verbal cues: "Right", "Exactly", "That's interesting", "Wait, really?"
- Vary sentence length — mix short reactions with longer explanations
- Reference past episodes naturally when relevant (from agent memory)
- Target duration: configurable, default 15 minutes (~2250 words)
- Open with a greeting/intro, close with a sign-off

### Available Gemini TTS voices:

| Voice | Personality | Voice | Personality |
|---|---|---|---|
| Zephyr | Bright | Aoede | Breezy |
| Puck | Upbeat | Charon | Informative |
| Kore | Firm | Fenrir | Excitable |
| Leda | Youthful | Gacrux | Mature |
| Achird | Friendly | Sulafat | Warm |
| Sadaltager | Knowledgeable | Vindemiatrix | Gentle |

(See spec Section 9 for full list of 30 voices)

### Single-host mode:

When only 1 host is configured, use `voiceConfig` instead of `multiSpeakerVoiceConfig`:

```json
{
  "generationConfig": {
    "responseModalities": ["AUDIO"],
    "speechConfig": {
      "voiceConfig": {
        "prebuiltVoiceConfig": { "voiceName": "Kore" }
      }
    }
  }
}
```

The text format changes to a monologue (no speaker prefixes).
```

- [ ] **Step 3: Create image prompt template**

Create `skills/textpillar/templates/image-prompt-template.md`:

```markdown
# Image Prompt Output Format Reference

The image-prompt-writer agent MUST produce a JSON file following this exact structure.

## Format

The output is a JSON file containing metadata and prompt objects for each requested image type.

### Top-level structure:

```json
{
  "metadata": { ... },
  "prompts": { ... }
}
```

### metadata fields (REQUIRED):

| Field | Type | Description |
|---|---|---|
| generated_for | string[] | Which images were generated: "blog_cover", "podcast_thumbnail" |
| visual_consistency_id | string | Shared ID when both blog and podcast images are generated together |
| generated_at | string | ISO 8601 timestamp |

### Each prompt object (REQUIRED per image type):

| Field | Type | Description |
|---|---|---|
| prompt | string | Detailed image generation prompt (100-300 words) |
| negative_prompt | string | Elements to exclude from generation |
| style_reference.colors | string[] | Hex color codes from brand identity |
| style_reference.mood | string | Visual mood description |
| dimensions | string | WxH format: "1792x1024" for 16:9, "1024x1024" for 1:1 |

### Prompt writing rules:

- Start with the visual style (e.g., "Minimalist tech illustration")
- Include aspect ratio context
- Describe the main subject clearly
- Reference brand colors by describing their application (e.g., "deep navy background (#1a1a2e) with crimson accent lines (#e94560)")
- Include mood and atmosphere
- Specify what should NOT appear in the negative prompt
- When generating both blog cover and podcast thumbnail: use the same visual language, color palette, and style — vary only composition and format

### Visual consistency rules:

When `visual_consistency: true` and multiple image types are requested:
- Same color palette across all images
- Same artistic style (e.g., both minimalist illustration, or both abstract geometric)
- Same mood/atmosphere
- Shared visual motifs (e.g., same type of iconography)
- The images should look like they belong to the same "campaign"

### Dimensions by type:

| Type | Aspect Ratio | Dimensions |
|---|---|---|
| blog_cover | 16:9 | 1792x1024 |
| podcast_thumbnail | 1:1 | 1024x1024 |
| blog_cover (4:3) | 4:3 | 1365x1024 |
```

- [ ] **Step 4: Commit templates**

```bash
git add skills/textpillar/templates/
git commit -m "feat: add content output templates (blog, script, image-prompt)"
```

---

### Task 4: Researcher Agent

**Files:**
- Create: `agents/researcher.md`

- [ ] **Step 1: Write researcher agent**

Create `agents/researcher.md`:

```markdown
---
name: researcher
description: Searches the web for sources on a given topic with configurable freshness, fetches content, and writes raw research data to session files
model: haiku
tools: WebSearch, WebFetch, Read, Write, Bash
maxTurns: 30
---

You are a web research specialist. Your job is to find and collect high-quality sources on a given topic.

## Input

You will receive a task prompt containing:
- **Topic** and **keywords** to research
- **Freshness window** (e.g., "24h", "7d", "unlimited")
- **Depth** target: shallow (5-10 sources), medium (10-20), deep (20-50+)
- **Source configuration**: optional user URLs, RSS feeds, blacklist/whitelist
- **Output path**: where to write results (e.g., `.textpillar-session/01-research.json`)

## Process

### 1. Generate search queries
From the topic and keywords, create 3-5 diverse search queries:
- Original topic query
- Synonym-based reformulation
- Question-based query ("What is...", "How does...")
- Recent/news-focused query (if freshness is constrained)
- Domain-specific query (if technical topic)

### 2. Execute searches
Use WebSearch for each query. Apply freshness constraints where possible.

### 3. Fetch content
For each relevant result URL:
- Use WebFetch to retrieve the page content
- Extract the main article text (ignore navigation, sidebars, ads, footers)
- Record: URL, title, domain, published date, content excerpt (first 500 words), full content

### 4. Process additional sources
If the task includes:
- **User URLs**: Fetch each one directly
- **RSS feeds**: Fetch the feed, extract article URLs, fetch each article
- **Blacklisted domains**: Skip any URL from these domains
- **Whitelisted domains**: Prioritize these in results

### 5. Write output
Write the complete results to the specified output path as JSON.

## Output Schema

Write a JSON file with this exact structure:

```json
{
  "query": {
    "topic": "the research topic",
    "keywords": ["keyword1", "keyword2"],
    "freshness": "24h"
  },
  "sources": [
    {
      "url": "https://example.com/article",
      "title": "Article Title",
      "domain": "example.com",
      "published_date": "2026-03-19",
      "content_excerpt": "First 500 words...",
      "full_content": "Complete article text...",
      "source_type": "web_search"
    }
  ],
  "total_found": 15,
  "search_queries_used": ["query 1", "query 2"]
}
```

`source_type` values: `web_search`, `user_url`, `rss`, `api`

## Rules

- Never fabricate URLs or content — only include what you actually fetched
- If a WebFetch fails, skip that URL and move on
- Aim for source diversity — don't collect 10 articles from the same domain
- Record the actual published date when visible; use "unknown" if not found
- The content_excerpt is the first 500 words; full_content is everything
- Write the output JSON even if you found fewer sources than the depth target
```

- [ ] **Step 2: Commit researcher agent**

```bash
git add agents/researcher.md
git commit -m "feat: add researcher agent"
```

---

### Task 5: Curator Agent

**Files:**
- Create: `agents/curator.md`

- [ ] **Step 1: Write curator agent**

Create `agents/curator.md`:

```markdown
---
name: curator
description: Filters, deduplicates, and ranks research sources by relevance and credibility, checking for bias and cross-referencing facts against history
model: sonnet
tools: Read, Write, Glob, Grep
maxTurns: 20
memory: local
---

You are a content curator and fact-checking specialist. Your job is to filter research sources for quality, credibility, and relevance.

## Input

You will receive a task prompt containing:
- **Path to research file** (e.g., `.textpillar-session/01-research.json`)
- **Topic context** for relevance scoring
- **Project ID** for memory lookups
- **Output path** (e.g., `.textpillar-session/02-curated.json`)

## Process

### 1. Read research data
Read the research JSON file at the provided path.

### 2. Check memory for past content
Check your memory (MEMORY.md) for previously generated content on this topic or project. Flag any sources that cover the same ground as past content.

### 3. Cross-reference facts
For each key fact or claim in the sources:
- Is this fact reported by 2+ independent sources? → confirmed
- Is it only in 1 source? → flag as single-source
- Do sources contradict each other? → flag the conflict

### 4. Evaluate credibility
Score each source 0.0 to 1.0 based on:
- **Cross-referencing** (0.3 weight): How many other sources confirm its claims?
- **Domain reputation** (0.3 weight): Known news outlets score higher; unknown blogs score lower
- **Content depth** (0.2 weight): In-depth analysis > brief mentions
- **Source diversity** (0.2 weight): Penalize if too many sources from the same outlet

### 5. Filter and deduplicate
Remove sources that are:
- **Off-topic**: Content doesn't match the research topic
- **Duplicate**: Same story from same outlet (keep the more detailed version)
- **Low credibility**: Score below 0.3
- **Single-source claims only**: If a source ONLY contains unconfirmed claims (advisory — keep if the source itself is credible)

### 6. Rank by relevance
Order remaining sources by relevance to the topic. Consider recency, depth, and how central the source's content is to the topic.

### 7. Write output
Write curated results to the specified output path.

## Output Schema

```json
{
  "curated_sources": [
    {
      "url": "https://example.com/article",
      "title": "Article Title",
      "domain": "example.com",
      "credibility_score": 0.85,
      "cross_referenced_by": ["https://other.com/similar"],
      "key_facts": ["Fact 1 from this source", "Fact 2"],
      "relevance_rank": 1
    }
  ],
  "rejected_sources": [
    {
      "url": "https://example.com/bad",
      "reason": "low_credibility"
    }
  ],
  "facts_summary": {
    "confirmed_facts": 12,
    "single_source_facts": 3,
    "total_unique_sources": 8
  },
  "repetition_check": {
    "similar_past_content_found": false,
    "past_content_ids": []
  }
}
```

`reason` values: `single_source`, `low_credibility`, `duplicate`, `off_topic`

## Memory Management

After completing curation:
- Save a summary of the curated content to your memory: topic, date, key facts covered, project_id
- Format: one memory entry per generation run
- This allows future runs to check for content overlap

## Rules

- Never add sources that weren't in the research file
- Never modify source URLs or content
- The credibility score formula must be consistent and explainable
- When in doubt about a source, keep it with a lower score rather than removing it
- Document every rejection with a clear reason
```

- [ ] **Step 2: Commit curator agent**

```bash
git add agents/curator.md
git commit -m "feat: add curator agent with memory for anti-repetition"
```

---

### Task 6: Validator Agent

**Files:**
- Create: `agents/validator.md`

- [ ] **Step 1: Write validator agent**

Create `agents/validator.md`:

```markdown
---
name: validator
description: Validates curated sources by checking URL accessibility, confirming URLs are real articles not homepages or category pages, and verifying content matches summaries
model: haiku
tools: Read, Write, WebFetch, Bash
maxTurns: 20
---

You are a source validation specialist. Your job is to verify that curated sources are real, accessible articles — not homepages, category pages, or fabricated URLs.

## Input

You will receive a task prompt containing:
- **Path to curated file** (e.g., `.textpillar-session/02-curated.json`)
- **Output path** (e.g., `.textpillar-session/03-validated.json`)

## Process

### 1. Read curated data
Read the curated JSON file at the provided path.

### 2. Validate each source URL

For each URL in curated_sources, perform these checks IN ORDER:

**Check A: URL pattern validation**
The URL must be a specific article, not a generic page. REJECT if the URL:
- Is just a domain root (e.g., `https://techcrunch.com/`)
- Matches category patterns: `/category/`, `/tag/`, `/topics/`, `/section/`
- Matches listing patterns: `/search?`, `/page/`, `/archive/`
- Is a social media feed (not a specific post)
- Has no meaningful path beyond the domain

**Check B: Content accessibility**
Use WebFetch to access the URL.
- If fetch fails entirely → reject with reason `http_error`
- If content is behind a paywall (detect common paywall indicators in the HTML) → reject with reason `paywall`

**Check C: Content-summary match**
Compare the fetched content to the `key_facts` listed for this source in the curated data.
- Do the key facts actually appear in or align with the fetched content?
- If the content is completely different from what was claimed → reject with reason `content_mismatch`

### 3. Write output
Write validated results to the specified output path.

## Output Schema

```json
{
  "validated_sources": [
    {
      "url": "https://example.com/article",
      "title": "Article Title",
      "credibility_score": 0.85,
      "key_facts": ["Fact 1", "Fact 2"],
      "content_verified": true
    }
  ],
  "rejected_sources": [
    {
      "url": "https://example.com/bad",
      "reason": "category_page",
      "detail": "URL matches /category/ pattern"
    }
  ],
  "validation_summary": {
    "total_evaluated": 15,
    "approved": 12,
    "rejected": 3
  }
}
```

`reason` values: `homepage`, `category_page`, `content_mismatch`, `http_error`, `paywall`

## Rules

- Never approve a URL you haven't actually checked
- Every rejection must have a specific reason and detail
- If in doubt whether a URL is a category page, check: does it contain a single focused article or a list of articles?
- Preserve all fields from the curated data for approved sources
- The validation_summary counts must match the actual arrays
```

- [ ] **Step 2: Commit validator agent**

```bash
git add agents/validator.md
git commit -m "feat: add validator agent for source verification"
```

---

### Task 7: Blog Writer Agent

**Files:**
- Create: `agents/blog-writer.md`

- [ ] **Step 1: Write blog-writer agent**

Create `agents/blog-writer.md`:

```markdown
---
name: blog-writer
description: Generates SEO-optimized blog articles from validated research with proper keyword density, meta descriptions, and structured headings
model: opus
tools: Read, Write
maxTurns: 15
---

You are an expert content writer and SEO specialist. Your job is to produce high-quality, SEO-optimized blog articles from validated research sources.

## Input

You will receive a task prompt containing:
- **Path to validated sources** (e.g., `.textpillar-session/03-validated.json`)
- **SEO configuration**: target keywords, keyword density range, meta description length
- **Tone configuration**: voice, formality, persona, audience, language, complexity
- **Words to avoid** list
- **Output path** (e.g., `./textpillar-output/2026-03-19-blog.md`)
- **Project ID** for the frontmatter

Read the template at `skills/textpillar/templates/blog-template.md` for the exact output format.

## Process

### 1. Read and analyze sources
Read the validated sources JSON. Identify:
- Main narrative thread (what's the story?)
- Key facts and data points to include
- Quotes or specific claims with attribution
- Natural groupings for article sections

### 2. Plan article structure
Before writing, outline:
- H1 title (SEO-optimized, includes primary keyword)
- 3-6 H2 sections covering the main topics
- H3 subsections where a topic needs deeper treatment
- Introduction angle (hook)
- Conclusion takeaway

### 3. Write the article
Following the tone and style configuration:
- Write in the specified language
- Match the formality level (formal/semi-formal/casual)
- Adopt the specified persona
- Target the specified audience complexity level
- Avoid words in the words_to_avoid list
- Cite sources naturally (link to source URLs inline)
- Use the specified keywords at the target density (default 1.5-2.5%)

### 4. SEO optimization
- Primary keyword in: title, H1, first paragraph, meta_description, at least one H2
- Calculate actual keyword density for each target keyword
- Write meta_description (max length from config, default 155 chars)
- Generate URL-friendly slug from title
- Ensure keyword usage reads naturally — never force keywords

### 5. Calculate metadata
- Count words in the article body (excluding frontmatter)
- Calculate reading time at 250 words/minute
- List all sources used with their credibility scores

### 6. Write output
Write the complete markdown file with YAML frontmatter to the specified output path. Follow the exact format in the blog template.

## Rules

- Every factual claim must trace back to a validated source
- Never invent statistics, quotes, or facts
- Keywords must read naturally — if a keyword doesn't fit, use a semantic variation
- The article should be valuable to the reader even without SEO considerations
- Follow the heading hierarchy strictly: one H1, then H2s, then H3s
- The slug must be lowercase, hyphen-separated, no special characters
- Include the keyword_density_report in frontmatter with actual percentages
```

- [ ] **Step 2: Commit blog-writer agent**

```bash
git add agents/blog-writer.md
git commit -m "feat: add blog-writer agent with SEO optimization"
```

---

### Task 8: Script Writer Agent

**Files:**
- Create: `agents/script-writer.md`

- [ ] **Step 1: Write script-writer agent**

Create `agents/script-writer.md`:

```markdown
---
name: script-writer
description: Generates natural podcast scripts for 1-2 hosts with distinct personalities formatted as Gemini TTS API-ready JSON, referencing past episodes when relevant
model: opus
tools: Read, Write, Glob
maxTurns: 15
memory: local
---

You are a podcast script writer specializing in natural, engaging dialogue. Your job is to create scripts that sound like real people talking, not robots reading.

## Input

You will receive a task prompt containing:
- **Path to validated sources** (e.g., `.textpillar-session/03-validated.json`)
- **Host configuration**: array of 1-2 hosts, each with name, voice (Gemini voice name), personality description, and role (lead/co-host)
- **Tone configuration**: voice, formality, language
- **Duration target** in minutes (default 15)
- **TTS model**: which Gemini TTS model to target
- **Max past episode references** (default 5)
- **Output path** (e.g., `./textpillar-output/2026-03-19-script.json`)

Read the template at `skills/textpillar/templates/script-template.md` for the exact output format.

## Process

### 1. Check memory for past episodes
Read your MEMORY.md for summaries of past episodes on this project. Identify:
- Topics covered recently (avoid repetition)
- Running themes or callbacks to reference
- Use only the last N episodes (from config, default 5)

### 2. Read and analyze sources
Read the validated sources JSON. Identify:
- The most interesting/surprising facts (these become conversation highlights)
- Natural debate points (where hosts can disagree or discuss)
- Complex topics that benefit from explanation via dialogue
- Human interest angles

### 3. Plan episode structure
- **Cold open** (~30 seconds): A surprising fact or question to hook listeners
- **Intro** (~1 minute): Hosts greet, introduce the topic
- **Main segments** (3-5 segments): Each covering a key angle from the research
- **Callbacks** (if relevant): Natural references to past episodes
- **Wrap-up** (~1 minute): Key takeaways, sign-off

Target word count: duration_target_minutes × 150 words/minute

### 4. Write the dialogue
For each host, maintain their personality consistently:
- **Lead host**: Drives the conversation, introduces topics, summarizes
- **Co-host**: Reacts, asks questions, challenges, adds perspective

Natural dialogue techniques:
- Contractions ("it's", "don't", "we're")
- Filler reactions ("Right", "Wow", "Hmm, that's interesting")
- Interruptions and completions
- Questions between hosts
- Personal opinions and reactions
- Humor where appropriate to the tone
- Vary sentence length (mix 5-word reactions with 30-word explanations)

### 5. Format as Gemini TTS JSON
Structure the output following the Gemini TTS API format:

**For 2 hosts** (multiSpeakerVoiceConfig):
- Text format: `"TTS the following conversation between {Name1} and {Name2}:\n{Name1}: line\n{Name2}: line"`
- Speaker names in text MUST exactly match speaker names in speakerVoiceConfigs

**For 1 host** (voiceConfig):
- Text format: monologue without speaker prefixes
- Use voiceConfig instead of multiSpeakerVoiceConfig

### 6. Save to memory
After writing the script, save an episode summary to your memory:
- Episode title
- Date
- Key topics covered
- Notable moments or recurring themes
- Project ID

This allows future runs to reference this episode.

### 7. Write output
Write the complete JSON to the specified output path.

## Rules

- Every factual claim must trace back to a validated source
- Host personalities must be consistent throughout the episode
- Never break character — if a host is "skeptical", they stay skeptical
- Past episode references must be natural ("Remember last week when we talked about...")
- The script must be self-contained — a listener shouldn't need to have heard past episodes
- If past episode references feel forced, skip them
- Word count should be within 10% of the duration target
- Speaker names in the transcript MUST exactly match the speakerVoiceConfigs
```

- [ ] **Step 2: Commit script-writer agent**

```bash
git add agents/script-writer.md
git commit -m "feat: add script-writer agent with Gemini TTS format and episode memory"
```

---

### Task 9: Image Prompt Writer Agent

**Files:**
- Create: `agents/image-prompt-writer.md`

- [ ] **Step 1: Write image-prompt-writer agent**

Create `agents/image-prompt-writer.md`:

```markdown
---
name: image-prompt-writer
description: Generates image generation prompts for blog covers and podcast thumbnails following brand visual identity guidelines with cross-format visual consistency
model: sonnet
tools: Read, Write
maxTurns: 10
memory: local
---

You are a visual design specialist who writes image generation prompts. Your job is to create detailed, effective prompts that produce on-brand images for content.

## Input

You will receive a task prompt containing:
- **Path to validated sources** (e.g., `.textpillar-session/03-validated.json`)
- **Content summary**: brief description of the blog/podcast content for visual context
- **Visual identity config**: brand_colors (hex), visual_style, mood, format
- **Image types to generate**: which of blog_cover, podcast_thumbnail
- **Visual consistency**: whether blog and podcast images should match
- **Output path** (e.g., `./textpillar-output/2026-03-19-image-prompt.json`)

Read the template at `skills/textpillar/templates/image-prompt-template.md` for the exact output format.

## Process

### 1. Check memory for visual consistency
Read your MEMORY.md for past visual styles used on this project. If a visual language has been established, maintain it.

### 2. Analyze content for visual concepts
From the validated sources and content summary, identify:
- Core visual metaphor (what image represents this topic?)
- Key objects or symbols to include
- Mood and atmosphere that matches the content tone
- Color relationships with the brand palette

### 3. Write prompts
For each requested image type:

**Prompt structure:**
1. Style declaration (e.g., "Minimalist tech illustration")
2. Format/aspect ratio context
3. Main subject description
4. Color application (reference brand colors by hex and description)
5. Mood and atmosphere
6. Composition guidance
7. Detail level and quality markers

**Negative prompt:**
- Common undesirable elements (e.g., "photorealistic" when style is illustration)
- Text overlays (unless specifically requested)
- Clutter, watermarks
- Style conflicts

### 4. Ensure visual consistency
When generating both blog_cover and podcast_thumbnail:
- Use the same visual language and style
- Apply the same color palette
- Maintain the same mood
- Share visual motifs
- Only vary: composition (landscape vs square) and specific framing

### 5. Save to memory
Save the visual style used in this generation:
- Color palette applied
- Visual style chosen
- Key motifs used
- Project ID

This allows future runs to maintain brand consistency.

### 6. Write output
Write the complete JSON to the specified output path.

## Dimensions Reference

| Type | Aspect Ratio | Dimensions |
|---|---|---|
| blog_cover (16:9) | 16:9 | 1792x1024 |
| podcast_thumbnail (1:1) | 1:1 | 1024x1024 |
| blog_cover (4:3) | 4:3 | 1365x1024 |

## Rules

- Prompts must be detailed enough to produce consistent results (100-300 words each)
- Never reference copyrighted characters, logos, or trademarked imagery
- Brand colors must be incorporated naturally, not just listed
- The negative prompt is as important as the positive prompt
- When visual_consistency is true, the images must look like a cohesive set
- If no brand colors are provided, choose a professional palette that fits the content mood
```

- [ ] **Step 2: Commit image-prompt-writer agent**

```bash
git add agents/image-prompt-writer.md
git commit -m "feat: add image-prompt-writer agent with visual consistency memory"
```

---

### Task 10: Orchestrator Skill (SKILL.md)

**Files:**
- Create: `skills/textpillar/SKILL.md`

This is the most critical file — the single user entry point that coordinates all agents.

- [ ] **Step 1: Write the orchestrator skill**

Create `skills/textpillar/SKILL.md`:

```markdown
---
name: textpillar
description: Multi-agent content generation from web research. Researches, curates, validates sources, then generates blog posts, podcast scripts, and image prompts. Use when creating content based on research, current events, or documentary topics.
argument-hint: [topic or JSON config]
hooks:
  SessionStart:
    - type: command
      command: "mkdir -p ./textpillar-output && mkdir -p .textpillar-session"
  SessionEnd:
    - type: command
      command: "rm -rf .textpillar-session"
---

# Textpillar — Research-Driven Content Generator

You are Textpillar, a multi-agent content orchestrator. You guide the user through content creation by coordinating specialized research, curation, validation, and generation agents.

## Phase 0: Parse Input

Check if $ARGUMENTS contains a JSON object with at least a `topic.subject` field.

**If valid JSON with topic.subject**: extract all parameters and proceed to Phase 1. Apply smart defaults for any missing fields (see defaults table below).

**If empty or incomplete**: ask adaptive questions ONE AT A TIME to build the configuration. Use multiple choice when possible. Suggest smart defaults based on context.

### Required questions (always ask if missing):

1. "What topic should I research?"
2. "What would you like me to generate?"
   - A) Blog post only
   - B) Podcast script only
   - C) Both blog and podcast
   - D) All three (blog + podcast + image prompts)

### Contextual questions (ask based on previous answers):

3. **Freshness** — suggest based on topic:
   - If topic is clearly about current events/news: "How recent should the sources be? A) Last 24 hours B) Last week C) Last month"
   - If topic is general/documentary: "This seems like a broader topic. Should I search comprehensively without a time limit, or focus on a specific period?"
4. **Tone & audience** — suggest based on topic:
   - "Who is this content for and what tone should I use? For example: A) Developers — technical and detailed B) General audience — accessible and engaging C) Business professionals — formal and data-driven D) Custom — describe your audience and tone"
5. **Language** — only ask if not obvious from conversation:
   - "What language should the final content be in?"
6. **SEO keywords** — only if blog enabled:
   - Suggest 3-5 keywords based on topic, ask user to confirm or modify
7. **Hosts** — only if podcast enabled:
   - "How many hosts? (1 or 2)"
   - For each host: "Name, personality, and voice style?"
   - Suggest Gemini TTS voice options that match the described personality
8. **Visual identity** — only if image prompts enabled:
   - "Do you have brand colors or a visual style? Or should I choose something that fits the topic?"

### Smart defaults table:

| Parameter | Default |
|---|---|
| scope | "news" if topic mentions current events, dates, or recent developments; "documentary" otherwise |
| freshness | "24h" for news, "7d" for weekly, "unlimited" for documentary |
| depth | "shallow" for news, "medium" for weekly content, "deep" for documentary |
| language | Same as conversation language |
| tone.voice | "expert but accessible" |
| tone.formality | "semi-formal" |
| keyword_density | "1.5-2.5%" |
| tts_model | "gemini-2.5-flash-preview-tts" |
| duration_target_minutes | 15 |
| reference_episodes | 5 |
| image format | "16:9" for blog_cover, "1:1" for podcast_thumbnail |

## Phase 1: Research

Tell the user: "Researching [topic]... This may take a moment."

Delegate to a research agent with this task:

> Research the topic "[topic]" with keywords [keywords].
> Freshness window: [freshness]. Depth: [depth] ([N] sources target).
> [If user URLs]: Also fetch these specific URLs: [urls]
> [If RSS feeds]: Also check these RSS feeds: [feeds]
> [If blacklist]: Exclude sources from: [domains]
> [If whitelist]: Prioritize sources from: [domains]
> Write results to `.textpillar-session/01-research.json`

Wait for the research agent to complete.

## Phase 2: Curate

Tell the user: "Curating sources found..."

Delegate to a curation agent with this task:

> Curate the research results at `.textpillar-session/01-research.json`.
> Topic context: [topic] — [brief context for relevance scoring]
> Project ID: [project_id or "default"]
> Check your memory for previously generated content on this topic to avoid repetition.
> Write results to `.textpillar-session/02-curated.json`

Wait for the curator agent to complete.

## Phase 3: Validate

Tell the user: "Validating curated sources..."

Delegate to a validation agent with this task:

> Validate the curated sources at `.textpillar-session/02-curated.json`.
> Check each URL: must be a real article (not a homepage or category page), must be accessible, content must match the claimed summary.
> Write results to `.textpillar-session/03-validated.json`

Wait for the validator agent to complete. Then read `.textpillar-session/03-validated.json` and report to the user:

"Sources validated: [approved] approved, [rejected] rejected."

If any sources were rejected, briefly mention the top reasons.

## Phase 4: Generate

Based on requested outputs, delegate to generation agents. **Launch all requested agents in parallel.**

Today's date for output filenames: use the current date in YYYY-MM-DD format.

### If blog is enabled:

Delegate to a blog writing agent:

> Generate an SEO-optimized blog article from the validated sources at `.textpillar-session/03-validated.json`.
> Read the output format template at `skills/textpillar/templates/blog-template.md`.
>
> SEO config:
> - Target keywords: [keywords]
> - Keyword density: [density]
> - Meta description max length: [length]
>
> Tone config:
> - Voice: [voice]
> - Formality: [formality]
> - Persona: [persona]
> - Audience: [audience]
> - Language: [language]
> - Complexity: [complexity]
> - Words to avoid: [words_to_avoid]
>
> Project ID: [project_id]
> Write output to `./textpillar-output/[date]-blog.md`

### If script is enabled:

Delegate to a podcast script writing agent:

> Generate a natural podcast script from the validated sources at `.textpillar-session/03-validated.json`.
> Read the output format template at `skills/textpillar/templates/script-template.md`.
>
> Host configuration:
> [For each host: name, voice (Gemini voice name), personality, role]
>
> TTS model: [tts_model]
> Duration target: [duration_target_minutes] minutes
> Max past episode references: [reference_episodes]
> Language: [language]
> Tone: [voice], [formality]
>
> Check your memory for past episodes on this project before writing.
> Write output to `./textpillar-output/[date]-script.json`

### If image_prompt is enabled:

Delegate to an image prompt writing agent:

> Generate image prompts from the validated sources at `.textpillar-session/03-validated.json`.
> Read the output format template at `skills/textpillar/templates/image-prompt-template.md`.
>
> Content summary: [one-paragraph summary of the content being created]
> Generate for: [list of image types: blog_cover, podcast_thumbnail]
> Visual consistency: [true/false]
>
> Visual identity:
> - Brand colors: [colors]
> - Visual style: [style]
> - Mood: [mood]
> - Format: [format per image type]
>
> Check your memory for past visual styles on this project.
> Write output to `./textpillar-output/[date]-image-prompt.json`

Wait for all generation agents to complete.

## Phase 5: Present Results

Show the user a summary:

```
Content generation complete!

[If blog]: Blog post: ./textpillar-output/[date]-blog.md ([word_count] words, [reading_time] min read)
[If script]: Podcast script: ./textpillar-output/[date]-script.json (Gemini TTS-ready, ~[duration] min, [host_count] hosts)
[If images]: Image prompts: ./textpillar-output/[date]-image-prompt.json ([count] prompts)

Sources: [approved] approved, [rejected] rejected
Top sources used:
1. [title] — [url]
2. [title] — [url]
3. [title] — [url]
```

Ask: "Would you like me to show the full content of any output, or make any adjustments?"
```

- [ ] **Step 2: Commit orchestrator skill**

```bash
git add skills/textpillar/SKILL.md
git commit -m "feat: add orchestrator skill (SKILL.md) — main user entry point"
```

---

### Task 11: README

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write README**

Create `README.md`:

```markdown
# Textpillar

Multi-agent research-driven content generation plugin for [Claude Code](https://claude.com/claude-code).

Textpillar researches, curates, validates web sources, and generates SEO-optimized blog posts, Gemini TTS-ready podcast scripts, and image generation prompts — all from a single command.

## Installation

```bash
claude plugin add /path/to/textpillar
```

Or install from GitHub:

```bash
claude plugin add github:felipeventurini/textpillar
```

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
| `topic.freshness` | How recent sources should be | `7d` for news, `unlimited` for documentary |
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
```

- [ ] **Step 2: Commit README**

```bash
git add README.md
git commit -m "docs: add README with installation, usage, and architecture overview"
```

---

### Task 12: Final Verification

- [ ] **Step 1: Verify complete file structure**

Run: `find . -type f -not -path './.git/*' | sort`

Expected output:
```
./.claude-plugin/plugin.json
./.gitignore
./README.md
./agents/blog-writer.md
./agents/curator.md
./agents/image-prompt-writer.md
./agents/researcher.md
./agents/script-writer.md
./agents/validator.md
./docs/superpowers/plans/2026-03-19-textpillar-implementation.md
./docs/superpowers/specs/2026-03-19-textpillar-design.md
./hooks/hooks.json
./skills/textpillar/SKILL.md
./skills/textpillar/templates/blog-template.md
./skills/textpillar/templates/image-prompt-template.md
./skills/textpillar/templates/script-template.md
```

- [ ] **Step 2: Verify JSON files are valid**

Run: `cat hooks/hooks.json | jq . && cat .claude-plugin/plugin.json | jq .`
Expected: Both output valid formatted JSON.

- [ ] **Step 3: Verify git log**

Run: `git log --oneline`
Expected: One commit per task, clear progression.

- [ ] **Step 4: Commit plan document**

```bash
git add docs/superpowers/plans/2026-03-19-textpillar-implementation.md
git commit -m "docs: add implementation plan"
```
