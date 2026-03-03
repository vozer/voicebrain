# VoiceBrain

Personal AI voice assistant that runs on Telegram. Send voice notes, text messages, or photos — VoiceBrain understands your intent and routes to the right service automatically.

Built as an [n8n](https://n8n.io) workflow using a conversational AI Agent (GPT-4o-mini) with 9 integrated tools.

## What It Does

| You say... | VoiceBrain does... |
|---|---|
| "Buy milk and eggs" | Adds items to your [Bring!](https://www.getbring.com/) shopping list |
| "Create an issue about the font bug in rollersite" | Creates a GitHub issue in the right repo (fuzzy-matches project names) |
| "Remember to call Juan tomorrow" | Saves a note in [Memos](https://www.usememos.com/) |
| "Show my open tasks" | Lists all memos tagged `#task #open` |
| "Search my notes about meeting" | Full-text search across all memos |
| "Write this as an email in my style" | Uses Pinecone style memory to match your writing voice |
| Voice note with multiple topics | Handles each topic separately (multi-task) |

## Architecture

```
Telegram Bot (@your_bot)
  ↓
n8n: voicebrain-v2 (23 nodes)
  ↓
Message Trigger → Route Chat Input → [Voice/Photo/Text]
  → Parse Input (dedup + chat history)
  → VoiceBrain Agent (GPT-4o-mini, 9 tools)
  → Format Reply → Send Reply

Tools:
  ├── create_memo      → Memos API
  ├── add_to_bring     → Bring! Shopping API
  ├── create_github_issue → GitHub API (private repo support)
  ├── search_memos     → Memos API (full-text search)
  ├── web_search       → Brave Search API
  ├── update_memo      → Memos API (append/replace/tag ops)
  ├── get_open_tasks   → Memos API (#task #open filter)
  ├── send_buttons     → Telegram inline keyboard
  └── Pinecone Vector Store → Writing style retrieval
```

## Prerequisites

### Services to Install

| Service | Purpose | Install |
|---------|---------|---------|
| [n8n](https://n8n.io) | Workflow automation platform | `docker run -d --name n8n -p 5678:5678 n8nio/n8n` |
| [Memos](https://www.usememos.com/) | Self-hosted note/memo server | `docker run -d --name memos -p 5230:5230 neosmemo/memos` |
| [Bring!](https://www.getbring.com/) | Shared shopping list app | Free account at getbring.com |
| [Pinecone](https://www.pinecone.io/) | Vector database for style memory | Free tier at pinecone.io |

### API Keys Required

| Key | Where to get it |
|-----|----------------|
| Telegram Bot Token | Create bot via [@BotFather](https://t.me/BotFather) |
| OpenAI API Key | [platform.openai.com](https://platform.openai.com) |
| GitHub PAT | [github.com/settings/tokens](https://github.com/settings/tokens) (fine-grained, Issues R/W) |
| Bring! Email/Password | Your Bring! account credentials |
| Brave Search API Key | [brave.com/search/api](https://brave.com/search/api/) (free 2K queries/month, optional) |
| Pinecone API Key | [pinecone.io](https://www.pinecone.io/) (optional, for style memory) |

## Installation

### 1. Set Up Services

Run n8n and Memos on the same Docker network so they can communicate:

```bash
docker network create voicebrain-net

docker run -d --name memos --network voicebrain-net \
  -p 5230:5230 -v memos-data:/var/opt/memos \
  neosmemo/memos

docker run -d --name n8n --network voicebrain-net \
  -p 5678:5678 -v n8n-data:/home/node/.n8n \
  -e MEMOS_URL=http://memos:5230 \
  -e MEMOS_TOKEN=your_memos_token \
  -e GITHUB_PAT=your_github_pat \
  -e BRING_EMAIL=your@email.com \
  -e BRING_PASSWORD=your_password \
  -e BRAVE_SEARCH_KEY=your_brave_key \
  n8nio/n8n
```

### 2. Import the Workflow

1. Open n8n at `http://localhost:5678`
2. Go to **Workflows** → **Import from File**
3. Import `workflow/voicebrain-v2.json`
4. Configure credentials:
   - **Telegram Bot API**: your bot token from BotFather
   - **OpenAI API**: your API key
   - **Pinecone API**: your API key (optional)
5. Update credential references in each node that shows a warning

### 3. Set Up Memos

1. Open Memos at `http://localhost:5230`
2. Create an account
3. Go to **Settings** → **Access Tokens** → create a token
4. Add the token as `MEMOS_TOKEN` environment variable in n8n

### 4. Set Up Pinecone (Optional)

For writing style memory:

1. Create a Pinecone index named `voicebraincommand`
   - Dimensions: 1024
   - Metric: cosine
2. Load your writing samples using the included data loader workflow
3. Add your Pinecone API key as a credential in n8n

### 5. Activate

1. Activate the workflow in n8n
2. Send `/start` to your Telegram bot
3. Start talking!

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `MEMOS_URL` | Yes | Memos API base URL (e.g., `http://memos:5230`) |
| `MEMOS_TOKEN` | Yes | Memos personal access token |
| `GITHUB_PAT` | Yes | GitHub fine-grained PAT (needs Issues R/W) |
| `BRING_EMAIL` | Yes | Bring! account email |
| `BRING_PASSWORD` | Yes | Bring! account password |
| `BRAVE_SEARCH_KEY` | No | Brave Search API key (free tier) |
| `OPENAI_API_KEY` | Yes | OpenAI API key (used by n8n AI nodes) |

## How the Tools Work

### Memos (create_memo, search_memos, update_memo, get_open_tasks)
[Memos](https://www.usememos.com/) is a self-hosted note server. VoiceBrain uses it as its primary storage — every note, task, idea, and reminder gets saved as a Memos entry with appropriate tags.

### Bring! (add_to_bring)
[Bring!](https://www.getbring.com/) is a shared shopping list app. When you mention buying something, VoiceBrain authenticates with the Bring! API and adds items to your default list. Shared with family members who also use the app.

### GitHub Issues (create_github_issue)
Creates issues in your GitHub repositories. Fuzzy-matches project names — say "rollersite" and it finds `madrid-rollers`. Supports private repos via fine-grained PAT.

### Brave Search (web_search)
Real-time web search via [Brave Search API](https://brave.com/search/api/). Free tier gives 2,000 queries/month.

### Pinecone (style memory)
[Pinecone](https://www.pinecone.io/) stores vector embeddings of your writing samples. When VoiceBrain needs to write something in your style (emails, posts), it retrieves similar text and uses it as context.

### Inline Buttons (send_buttons)
When VoiceBrain is unsure (e.g., which project for a GitHub issue), it sends Telegram inline keyboard buttons for disambiguation.

## Conversation Memory

VoiceBrain maintains a rolling conversation history (last 40 messages) in n8n workflow static data. This enables multi-turn conversations:

```
You: "Create an issue about the login bug"
Bot: "Which project?" [buttons: madrid-alive, rollersite, abonoteatro]
You: [clicks madrid-alive]
Bot: "Created Issue #5 in madrid-alive: Login bug"
```

## Telegram Commands

| Command | Description |
|---------|-------------|
| `/start` | Welcome message + clear chat history |
| `/help` | List all capabilities |
| `/tasks` | List open tasks (shortcut for natural language) |
| `/search <term>` | Search notes (shortcut for natural language) |

## Key Technical Notes

- **toolCode `query` variable**: n8n Code Tool nodes receive AI Agent input via `query`, not `$input`
- **Input schema**: Tools needing structured JSON must have `specifyInputSchema: true` with a `jsonSchemaExample`
- **Bring! API headers**: Requires `X-BRING-API-KEY`, `X-BRING-CLIENT: android`, `X-BRING-APPLICATION: bring`
- **Telegram formatting**: HTML mode (`<b>bold</b>`, `<i>italic</i>`), no markdown links
- **n8n partial updates**: `n8n_update_partial_workflow` replaces the entire `parameters` object — always include all fields

## License

MIT
