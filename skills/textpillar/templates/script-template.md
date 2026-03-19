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
