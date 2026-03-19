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
| blog_cover (16:9) | 16:9 | 1792x1024 |
| podcast_thumbnail (1:1) | 1:1 | 1024x1024 |
| blog_cover (4:3) | 4:3 | 1365x1024 |
