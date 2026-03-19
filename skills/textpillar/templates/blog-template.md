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
