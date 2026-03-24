# Azure TTS Per-Model Voice Override Fix

## Summary
Fixed the Azure TTS backend to respect per-model voice overrides and implemented a voice selection UI in the Model Editor.

## Changes Made

### 1. Backend: `backend/open_webui/routers/audio.py`
**Issue**: The Azure TTS endpoint was always using the global `TTS_VOICE` configuration and ignoring any voice override in the request payload.

**Fix**: Modified the Azure TTS section in the `/speech` endpoint (lines 485-498) to:
- Extract the `voice` from the request payload using `payload.get("voice", ...)`
- Use the payload voice if provided, otherwise fall back to the global `TTS_VOICE` config
- Derive the `locale` from the selected voice ID (e.g., `en-US` from `en-US-JennyNeural`)

```python
# Use voice from payload if provided, otherwise fall back to global config
language = payload.get("voice", request.app.state.config.TTS_VOICE)
# Derive locale from voice ID (e.g., "en-US-JennyNeural" -> "en-US")
locale = "-".join(language.split("-")[:2])
```

### 2. Frontend: `src/lib/components/workspace/Models/ModelEditor.svelte`
**Issue**: The Model Editor only provided a simple text input for TTS voice, making it difficult for users to know which voices are available.

**Fix**: Implemented voice selection UI similar to the Audio Settings:

#### Added imports (line 10):
```typescript
import { getVoices as _getVoices } from '$lib/apis/audio';
```

#### Added state variables (lines 106-116):
```typescript
let voices = [];

const getVoices = async () => {
    const res = await _getVoices(localStorage.token).catch((e) => {
        console.error('Failed to fetch voices:', e);
    });

    if (res) {
        voices = res.voices;
    }
};
```

#### Called `getVoices()` in onMount (line 253):
```typescript
await getVoices();
```

#### Updated TTS Voice UI (lines 832-854):
Added a `datalist` element that provides auto-completion from the fetched voices:
```svelte
<div class="flex w-full">
    <div class="flex-1">
        <input
            list="voice-list"
            class="w-full text-sm bg-transparent outline-hidden"
            type="text"
            bind:value={tts.voice}
            placeholder={$i18n.t('e.g. alloy, echo, shimmer')}
        />
        <datalist id="voice-list">
            {#each voices as voice}
                <option value={voice.id}>{voice.name}</option>
            {/each}
        </datalist>
    </div>
</div>
```

## Behavior

### Provider Voice Selection
- The `/voices` endpoint returns voices for the currently configured TTS engine
- If the administrator changes the TTS engine, the voice list will update after a page refresh
- If no voices are returned (e.g., engine not configured), the datalist will be empty, but users can still manually enter a voice ID

### Voice Override Storage
- The Model Editor correctly saves the selected voice to `info.meta.tts.voice`
- `ResponseMessage.svelte` and `CallOverlay.svelte` already retrieve this value via `getVoiceId()` helper functions
- The TTS voice is sent in the request payload to the `/speech` endpoint

### Azure Voice ID Format
- Azure voice IDs (ShortNames) contain the locale (e.g., `en-US-AvaMultilingualNeural`)
- The locale is derived by splitting the voice ID on `-` and taking the first two parts
- This logic is robust for standard Azure naming conventions

## Testing Notes
1. Test with Azure TTS engine configured with different voices
2. Create/edit a model and select a voice from the dropdown
3. Verify that TTS uses the model-specific voice when reading aloud or during voice calls
4. Test that the global default voice is used when no model-specific voice is set
5. Test with different TTS engines (OpenAI, ElevenLabs) to ensure the UI works correctly
