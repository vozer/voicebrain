# Changelog

All notable changes to VoiceBrain are documented here.
Format based on [Keep a Changelog](https://keepachangelog.com/).

## [2.1.0] - 2026-03-03

### Added
- **mark_done mode** for update_memo — replaces `#open` with `#done` in memo content
- **replace_tag mode** for update_memo — replace any tag with another (`old_tag` → `new_tag`)
- **voicebrain GitHub repo** — public repo with sanitized workflow, README, CHANGELOG, AGENTS.md

### Fixed
- **update_memo always reads memo first** — all modes now GET current content before PATCHing
- **GitHub tool filters archived repos** — no more permission errors on familias-enlazadas, madrid-alive, etc.
- **System prompt repo list** — updated to non-archived repos: voicebrain, madrid-alive-vercel, rollersite-vercel, abonoteatro, madrid-rollers, community-orchestrator, Review-Gate

## [2.0.0] - 2026-03-03

### Added
- **Conversational AI Agent** — replaced classify-and-route pipeline (v1) with a single GPT-4o-mini AI Agent that decides actions naturally
- **9 integrated tools**: create_memo, add_to_bring, create_github_issue, search_memos, web_search, update_memo, get_open_tasks, send_buttons, Pinecone style retrieval
- **Conversation memory** — rolling history of last 40 messages stored in workflow static data, injected as context for every AI call
- **Inline keyboard buttons** — AI can ask disambiguation questions with clickable Telegram buttons (e.g., "Which project?")
- **Web search** — Brave Search API tool (free tier, 2K queries/month)
- **Memo editing** — update_memo tool supports append and replace modes
- **Task listing** — get_open_tasks retrieves memos tagged `#task #open`
- **Multi-task from voice** — single voice note mentioning multiple topics triggers multiple tool calls
- **Input schemas** — tools needing structured JSON use `specifyInputSchema: true` with `jsonSchemaExample` for reliable AI-to-tool communication
- **Telegram formatting rules** — system prompt enforces plain URLs (no markdown links) and HTML tags for bold/italic

### Fixed
- **toolCode input variable** — all tools use `query` variable (not `$input`) for AI Agent parameters
- **Bring! API auth** — added required `X-BRING-API-KEY`, `X-BRING-CLIENT`, `X-BRING-APPLICATION` headers
- **GitHub private repos** — uses `GET /user/repos` (authenticated) instead of `GET /users/{user}/repos` (public only)
- **`/start` HTML error** — escaped `<term>` to `&lt;term&gt;` in help text
- **Send Reply operation** — explicitly set `operation: sendMessage`

## [1.0.0] - 2026-02-24

### Added
- **Initial release** — classify-and-route pipeline with GPT-4o-mini structured JSON classification
- **Telegram + Galaxy Watch** input paths
- **Multi-task detection** — splits voice notes with multiple topics into separate tasks
- **Invoke Command** — "invoke command" trigger phrase activates GPT-4o for content transformation with Pinecone style memory
- **Integrations**: Memos, Bring! Shopping, GitHub Issues
- **Daily digest** — scheduled workflow sends open tasks summary each morning
- **Pinecone data loader** — utility workflow for loading writing samples
