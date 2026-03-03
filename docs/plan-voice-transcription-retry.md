# Plan: Save Transcribed Voice Notes for Retry

**GitHub Issue:** vozer/voicebrain#2  
**Status:** Approved, not yet implemented  
**Priority:** Medium  
**Estimated Effort:** ~30 minutes

## Executive Summary

Store raw voice transcriptions in n8n workflow static data (last 20 entries). Add a `get_last_transcription` tool so the AI can retrieve the original text when the user says "retry", "that was wrong", or asks to redo a failed action.

**Success criteria:** User sends voice note → AI misunderstands → user says "retry" → AI retrieves raw transcription and reprocesses correctly.

## Current State

- Voice notes are transcribed via Whisper and passed to the AI Agent as `$json.text`
- If the AI misinterprets, the transcription is lost — only the chat history summary remains
- No way for the AI to access the original transcription for retry

## Implementation Details

### 1. Update `parse-input` node (ID: `parse-input`)

**What:** When `messageType === 'voice'`, store the raw transcription in `staticData.voiceTranscriptions[]`.

**Add after the existing chat history storage block:**

```javascript
// Store voice transcriptions for retry
if (messageType === 'voice' && text) {
  if (!staticData.voiceTranscriptions) staticData.voiceTranscriptions = [];
  staticData.voiceTranscriptions.push({
    text: text,
    messageId: messageId,
    ts: Date.now()
  });
  if (staticData.voiceTranscriptions.length > 20) {
    staticData.voiceTranscriptions = staticData.voiceTranscriptions.slice(-20);
  }
}
```

**Also add to `/start` handler in `cmd-handler` node:**

```javascript
if (cmd === '/start') {
  staticData.chatHistory = [];
  staticData.voiceTranscriptions = [];  // ADD THIS LINE
  // ... rest of /start handler
}
```

**Risk:** None — additive change, no existing behavior affected.

### 2. New Code Tool node: `get_last_transcription` (ID: `tool-transcription`)

**Type:** `@n8n/n8n-nodes-langchain.toolCode`  
**specifyInputSchema:** `false` (plain string input — optional index)

**Tool description:**
```
Retrieve the original transcription of a recent voice note. Call this when the user says the result was wrong, asks to retry, or wants to redo something. Pass nothing for the most recent, or a number (e.g. "2") for the second most recent.
```

**Code:**
```javascript
try {
  const staticData = $getWorkflowStaticData('global');
  const transcriptions = staticData.voiceTranscriptions || [];
  if (transcriptions.length === 0) return 'No voice transcriptions stored.';
  
  const indexStr = typeof query === 'string' ? query.trim() : '';
  const index = indexStr && !isNaN(Number(indexStr)) ? Number(indexStr) : 1;
  
  const item = transcriptions[transcriptions.length - index];
  if (!item) return `No transcription at position ${index}. Have ${transcriptions.length} stored.`;
  
  const ago = Math.round((Date.now() - item.ts) / 60000);
  return `Voice transcription (${ago} min ago):\n${item.text}`;
} catch(e) { return `Transcription error: ${e.message}`; }
```

**Connection:** Connect output to `vb-agent` AI Agent node (ai_tool connection).

### 3. Update system prompt (node `vb-agent`)

**Add to RULES section:**

```
12. When the user says the output was wrong, asks to retry, or wants to redo — use get_last_transcription to retrieve the original voice note text and reprocess it
```

## Execution Order

1. Update `parse-input` — add voice transcription storage (~5 lines)
2. Update `cmd-handler` — clear transcriptions on `/start` (~1 line)
3. Create `tool-transcription` — new Code Tool node
4. Connect `tool-transcription` to `vb-agent` (ai_tool connection)
5. Update `vb-agent` system prompt — add rule #12
6. Test: send voice note → say "retry" → verify AI retrieves original text
7. Update sanitized workflow in GitHub repo

## Checklists

### Senior Code Review
- [x] Follows existing patterns (static data storage, Code Tool)
- [x] No breaking changes
- [x] Error handling in tool (try-catch, empty state)
- [x] Performance: 20-item cap, no API calls

### Code Simplification
- [x] Reuses existing static data pattern
- [x] Single responsibility (store vs retrieve)
- [x] No duplicate code

### Security
- [x] No sensitive data exposure
- [x] No external API calls
- [x] Static data scoped to workflow

## Testing

1. Send voice note → verify transcription stored in static data
2. Send "retry" or "that was wrong" → verify AI calls get_last_transcription
3. Send multiple voice notes → verify FIFO (oldest dropped after 20)
4. Send `/start` → verify transcriptions cleared
5. Ask for transcription when none exists → verify graceful "No transcriptions" response

## Rollback

Remove the tool node and revert parse-input/cmd-handler changes. No data migration needed — static data clears automatically.
