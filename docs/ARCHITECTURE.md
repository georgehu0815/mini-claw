# Mini-Claw Architecture

## Overview

Mini-Claw is a lightweight Telegram bot that provides persistent AI conversations using the Pi coding agent as a backend. The architecture follows a clean layered design with clear separation of concerns.

## Architecture Diagram

```mermaid
graph TB
    %% User & External Services Layer
    subgraph external["🌐 External Services"]
        User([👤 User])
        TelegramAPI[Telegram Bot API]
        Claude[Claude AI<br/>Anthropic]
        ChatGPT[ChatGPT<br/>OpenAI]
    end

    %% Application Entry Layer
    subgraph entry["🚀 Entry Point Layer"]
        Main[index.ts<br/>① Load Config<br/>② Check Pi Auth<br/>③ Create Bot<br/>④ Start Server]
        Config[config.ts<br/>Environment<br/>Variables]
    end

    %% Bot Layer
    subgraph bot["🤖 Bot Layer - bot.ts"]
        Commands[Command Handler<br/>/start, /help, /pwd<br/>/cd, /shell, /new<br/>/session, /status]
        MessageHandler[Message Handler<br/>Text Processing]
        CallbackHandler[Callback Handler<br/>Inline Keyboards]
    end

    %% Core Services Layer
    subgraph services["⚙️ Core Services"]
        RateLimit[rate-limiter.ts<br/>Cooldown<br/>Enforcement]
        Workspace[workspace.ts<br/>Directory State<br/>Per Chat]
        Sessions[sessions.ts<br/>Archive, Switch<br/>Title Generation<br/>Cleanup]
    end

    %% Pi Agent Layer
    subgraph piLayer["🧠 Pi Agent Layer - pi-runner.ts"]
        Lock[AsyncLock<br/>Per Chat<br/>Concurrency Control]
        PiExec[Pi Process<br/>Spawn & Monitor]
        ActivityStream[Activity Streaming<br/>Reading, Writing<br/>Running, Thinking]
    end

    %% Processing Layer
    subgraph processing["📝 Processing Layer"]
        FileDetector[file-detector.ts<br/>Parse Output<br/>Detect New Files<br/>Workspace Snapshot]
        Markdown[markdown.ts<br/>Markdown → HTML<br/>Telegram Formatting]
    end

    %% Storage Layer
    subgraph storage["💾 Storage Layer"]
        SessionFiles[(~/.mini-claw/sessions/<br/>telegram-{chatId}.jsonl<br/>JSONL Format)]
        WorkspaceState[(~/.mini-claw/<br/>workspaces.json<br/>Chat → Directory)]
        UserWorkspace[(~/mini-claw-workspace/<br/>User Files<br/>Pi Working Dir)]
        PiAuth[(~/.pi/agent/<br/>auth.json<br/>OAuth Tokens)]
    end

    %% Data Flow - External to Entry
    User -->|Message| TelegramAPI
    TelegramAPI -->|Webhook/Poll| MessageHandler

    %% Entry Layer Flow
    Main --> Config
    Main --> Commands

    %% Bot Layer Flow
    MessageHandler -->|Check| RateLimit
    RateLimit -->|Allow| Workspace
    Commands --> Workspace
    Commands --> Sessions
    CallbackHandler --> Sessions

    %% Core to Pi Flow
    Workspace -->|Get CWD| MessageHandler
    MessageHandler -->|Acquire| Lock
    Lock -->|Execute| PiExec

    %% Pi to AI Flow
    PiExec -->|API Call| Claude
    PiExec -->|API Call| ChatGPT

    %% Activity Updates
    PiExec -->|Stream| ActivityStream
    ActivityStream -.->|Status Updates| TelegramAPI

    %% Response Flow
    PiExec -->|Output| FileDetector
    PiExec -->|Output| Markdown
    FileDetector -->|Detected Files| TelegramAPI
    Markdown -->|HTML Response| TelegramAPI
    TelegramAPI -->|Reply| User

    %% Storage Connections
    PiExec <-->|Read/Write| SessionFiles
    Workspace <-->|Persist| WorkspaceState
    PiExec -->|Execute In| UserWorkspace
    PiExec -->|Auth| PiAuth
    Sessions <-->|Manage| SessionFiles
    FileDetector -->|Scan| UserWorkspace

    %% Styling
    classDef external fill:#e7f5ff,stroke:#1971c2,stroke-width:2px
    classDef entry fill:#d3f9d8,stroke:#2f9e44,stroke-width:2px
    classDef bot fill:#ffe8cc,stroke:#d9480f,stroke-width:2px
    classDef services fill:#e5dbff,stroke:#5f3dc4,stroke-width:2px
    classDef pi fill:#ffe3e3,stroke:#c92a2a,stroke-width:2px
    classDef processing fill:#c5f6fa,stroke:#0c8599,stroke-width:2px
    classDef storage fill:#fff4e6,stroke:#e67700,stroke-width:2px

    class User,TelegramAPI,Claude,ChatGPT external
    class Main,Config entry
    class Commands,MessageHandler,CallbackHandler bot
    class RateLimit,Workspace,Sessions services
    class Lock,PiExec,ActivityStream pi
    class FileDetector,Markdown processing
    class SessionFiles,WorkspaceState,UserWorkspace,PiAuth storage
```

## Layer Descriptions

### 🌐 External Services Layer
- **User**: End user interacting via Telegram
- **Telegram Bot API**: Message delivery platform
- **AI Providers**: Claude (Anthropic) or ChatGPT (OpenAI) via Pi agent

### 🚀 Entry Point Layer (`index.ts`, `config.ts`)
- **Main Entry**: Bootstraps the application
  - Loads environment configuration
  - Validates Pi authentication
  - Creates bot instance
  - Sets up graceful shutdown handlers
- **Config**: Parses environment variables
  - Telegram bot token
  - Workspace paths
  - Timeouts and rate limits
  - User access control

### 🤖 Bot Layer (`bot.ts`)
- **Command Handler**: Processes Telegram commands
  - `/start` - Welcome message
  - `/help` - Command reference
  - `/pwd`, `/cd`, `/home` - Directory navigation
  - `/shell <cmd>` - Direct shell execution
  - `/new` - Start fresh session
  - `/session` - List and switch sessions
  - `/status` - Bot status check
- **Message Handler**: Processes natural language messages
  - Rate limiting check
  - Workspace resolution
  - Pi execution coordination
  - Response delivery
- **Callback Handler**: Manages inline keyboard interactions
  - Session switching
  - Session cleanup

### ⚙️ Core Services Layer

#### Rate Limiter (`rate-limiter.ts`)
- Enforces cooldown period between messages (default 5s)
- Prevents spam and API abuse
- Per-chat tracking

#### Workspace Manager (`workspace.ts`)
- Maintains current working directory per chat
- Persists state to `~/.mini-claw/workspaces.json`
- Handles path resolution (relative, absolute, `~`)
- Validates directory existence

#### Session Manager (`sessions.ts`)
- **Session Files**: JSONL format conversation history
- **Archiving**: Timestamps old sessions when creating new ones
- **Switching**: Copy archived session to active session
- **Title Generation**: Uses Pi to generate short titles from first message
- **Cleanup**: Deletes old sessions (keeps latest 5 per chat)

### 🧠 Pi Agent Layer (`pi-runner.ts`)

#### AsyncLock
- One execution per chat at a time
- Prevents concurrent Pi processes on same session
- Queues rapid-fire messages

#### Pi Process Manager
- Spawns `pi` CLI process
- Passes session file and workspace
- Monitors stdout/stderr
- Handles timeouts (default 5min)

#### Activity Streaming
- Parses Pi output in real-time
- Detects activities:
  - 🧠 Thinking
  - 📖 Reading files
  - ✍️ Writing files
  - ⚡ Running commands
  - 🔍 Searching codebase
- Updates Telegram status message (throttled to 2s)

### 📝 Processing Layer

#### File Detector (`file-detector.ts`)
- **Output Parsing**: Looks for "Created: /path/file.txt" patterns
- **Workspace Diffing**: Compares before/after snapshots
- **Auto-Send**: Sends detected images/documents to Telegram
- Supported types: PNG, JPG, PDF, TXT, MD, JSON, etc.

#### Markdown Renderer (`markdown.ts`)
- Converts Pi's Markdown output to HTML
- Telegram-compatible formatting
- Fallback to plain text if HTML fails
- Handles code blocks, links, emphasis

### 💾 Storage Layer

#### Session Files (`~/.mini-claw/sessions/`)
- Format: `telegram-{chatId}.jsonl` (active session)
- Format: `telegram-{chatId}-{timestamp}.jsonl` (archived)
- JSONL: One JSON message per line
- Pi handles automatic context compaction

#### Workspace State (`~/.mini-claw/workspaces.json`)
- Maps chat ID to current working directory
- Persists across bot restarts

#### User Workspace (`~/mini-claw-workspace/`)
- Default working directory for Pi execution
- User can change per chat via `/cd`

#### Pi Auth (`~/.pi/agent/auth.json`)
- OAuth tokens for Claude/ChatGPT
- Managed by `pi /login` command
- Shared with Mini-Claw automatically

## Message Flow (Typical User Interaction)

```
┌─────────────────────────────────────────────────────────────┐
│ 1. User sends: "Create a Python script to sort a CSV file" │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Telegram API delivers to MessageHandler                  │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Rate Limiter checks cooldown (5s default)                │
│    ✅ Allowed → Continue                                     │
│    ❌ Denied → Reply "Please wait Xs"                        │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. Workspace Manager resolves CWD for this chat             │
│    Returns: ~/mini-claw-workspace (or custom path)          │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. File Detector snapshots workspace (before execution)     │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. Send status message: "🔄 Working..."                      │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 7. AsyncLock acquires lock for this chat                    │
│    (Blocks if Pi already running for this chat)             │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 8. Pi Process spawns with:                                  │
│    - Session: ~/.mini-claw/sessions/telegram-123.jsonl      │
│    - CWD: ~/mini-claw-workspace                             │
│    - Prompt: "Create a Python script to sort a CSV file"    │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 9. Pi executes (internally):                                │
│    a. Reads session history                                 │
│    b. Calls Claude/ChatGPT API                              │
│    c. AI generates code + plan                              │
│    d. Pi writes sort_csv.py                                 │
│    e. Pi runs chmod +x sort_csv.py                          │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 10. Activity Streaming parses Pi output:                    │
│     - "Reading: sort_csv.py" → 📖 Reading...                │
│     - "Writing: sort_csv.py" → ✍️ Writing...                │
│     - "Running: chmod +x" → ⚡ Running... (2s)               │
│     Updates status message every 2s                          │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 11. Pi completes, returns output (Markdown)                 │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 12. Delete status message                                   │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 13. Markdown Renderer converts to HTML                      │
│     Send formatted response to Telegram                      │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 14. File Detector compares workspace snapshots              │
│     Detected: sort_csv.py (new file)                        │
│     Send as Telegram document attachment                     │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 15. AsyncLock released                                      │
│     Bot ready for next message                               │
└─────────────────────────────────────────────────────────────┘
```

## Concurrency & Performance

### Per-Chat Locking
- **Problem**: Multiple messages from same user could corrupt session file
- **Solution**: `AsyncLock` ensures sequential Pi execution per chat
- **Impact**: Different chats can run Pi in parallel

### Rate Limiting
- **Problem**: User sends rapid-fire messages
- **Solution**: Enforce cooldown (default 5s) between messages
- **Impact**: Prevents API abuse, gives Pi time to complete

### Typing Indicators
- **Implementation**: Send `typing` action every 4s during Pi execution
- **Purpose**: Shows user the bot is actively working

### Status Updates (Streaming)
- **Throttling**: Max 1 status update per 2 seconds
- **Prevents**: Telegram API rate limits
- **UX**: User sees real-time activity (reading, writing, running)

### Message Splitting
- **Telegram Limit**: 4096 characters per message
- **Solution**: Split long responses at newlines or spaces
- **Preserves**: Markdown formatting integrity

## Session Persistence

### Session File Format (JSONL)
```jsonl
{"role":"user","content":"Hello"}
{"role":"assistant","content":"Hi! How can I help?"}
{"role":"user","content":"Create a Python script"}
{"role":"assistant","content":"Here's a Python script..."}
```

### Auto-Compaction
- Pi automatically compacts when context window fills
- Keeps recent messages, summarizes older ones
- Full history preserved in JSONL file

### Session Operations

#### Archive Session (`/new`)
1. Acquire lock (wait for Pi to finish)
2. Rename `telegram-{chatId}.jsonl` → `telegram-{chatId}-{timestamp}.jsonl`
3. Clear active session tracking
4. Next message creates fresh session

#### Switch Session (`/session`)
1. List all sessions for this chat
2. Generate titles using Pi (parallel, max 10 sessions)
3. Show inline keyboard
4. On selection:
   - Archive current session (if default)
   - Copy selected session to default path
   - Update active session tracking

#### Cleanup (`/session` → Clean Up)
- Keep latest 5 sessions per chat
- Delete older sessions
- Prevents disk bloat

## Configuration

### Environment Variables

```bash
# Required
TELEGRAM_BOT_TOKEN=your_bot_token_here

# Optional - Paths
MINI_CLAW_WORKSPACE=~/mini-claw-workspace    # Default Pi working directory
MINI_CLAW_SESSION_DIR=~/.mini-claw/sessions  # Session storage

# Optional - Behavior
PI_THINKING_LEVEL=low|medium|high             # Pi verbosity (default: low)
ALLOWED_USERS=123,456,789                     # Comma-separated Telegram user IDs

# Optional - Timeouts (milliseconds)
RATE_LIMIT_COOLDOWN_MS=5000                   # Message cooldown (default: 5s)
PI_TIMEOUT_MS=300000                          # Pi execution timeout (default: 5min)
SHELL_TIMEOUT_MS=60000                        # /shell command timeout (default: 60s)
SESSION_TITLE_TIMEOUT_MS=10000                # Title generation timeout (default: 10s)
```

### Access Control
- If `ALLOWED_USERS` is empty → Anyone can use bot
- If set → Only listed user IDs can interact
- All others receive "Not authorized" message

## Error Handling

### Pi Not Authenticated
```bash
# Check
make status

# Fix
make login  # or: pi /login
```

### Session File Locked
- Another Pi process running on same session
- Lock ensures safety, waits for completion
- Check: `ps aux | grep pi`

### Timeout
- Default: 5 minutes for Pi execution
- Configurable via `PI_TIMEOUT_MS`
- Pi process killed with SIGTERM
- User receives timeout error

### File Detection Failures
- File might be deleted between detection and sending
- Gracefully handles with "(Could not send file: ...)"

## Security Considerations

### User Access Control
- Telegram user ID whitelist (`ALLOWED_USERS`)
- Prevents unauthorized bot usage
- Important: Tokens in `.env` should never be committed

### Shell Command Execution
- `/shell` runs bash commands directly
- **Risk**: No sandboxing
- **Mitigation**: User access control + trust model
- **Note**: Pi also executes commands, same risk level

### Session Isolation
- Each chat has separate session file
- No cross-chat data leakage
- Workspace can be shared (use `/cd` to isolate)

### API Token Storage
- Pi stores OAuth tokens in `~/.pi/agent/auth.json`
- Not exposed to bot code
- Managed by Pi CLI

## Deployment Options

### Option 1: systemd (Linux)
```bash
make install-service
systemctl --user start mini-claw
systemctl --user enable mini-claw  # Auto-start on boot
```

### Option 2: pm2 (Cross-platform)
```bash
pnpm build
pm2 start dist/index.js --name mini-claw
pm2 save
pm2 startup  # Auto-start on boot
```

### Option 3: tmux (Manual)
```bash
tmux new -s mini-claw
make start
# Ctrl+B, D to detach
# tmux attach -t mini-claw to reattach
```

### Option 4: Docker (Planned)
```dockerfile
# Future: Dockerfile for containerized deployment
```

## Testing

### Unit Tests
```bash
pnpm test
```

Test files:
- `bot.test.ts` - Bot command handling
- `config.test.ts` - Configuration loading
- `file-detector.test.ts` - File detection logic
- `pi-runner.test.ts` - Pi process management
- `rate-limiter.test.ts` - Rate limiting
- `sessions.test.ts` - Session management
- `workspace.test.ts` - Workspace handling

### Integration Testing
- Requires actual Telegram bot token
- Requires Pi authentication
- Manual testing via Telegram app

## Performance Characteristics

### Latency
- **Message → First Response**: 2-30 seconds (depends on Pi/AI)
- **Status Updates**: Every 2 seconds during execution
- **Typing Indicator**: Every 4 seconds

### Throughput
- **Per Chat**: Sequential (1 message at a time via lock)
- **Multi-Chat**: Parallel (no global lock)
- **Rate Limit**: 1 message per 5 seconds per chat

### Resource Usage
- **Memory**: ~50MB base + Pi process overhead
- **Disk**: Session files grow with conversation length
  - Mitigated by Pi auto-compaction
  - Mitigated by `/new` and cleanup
- **CPU**: Minimal (Pi does the heavy lifting)

## Future Enhancements

### Planned Features
- [ ] Voice message transcription (Whisper API)
- [ ] Image analysis (Vision models via Pi)
- [ ] Multiple workspace support (project switching)
- [ ] Inline keyboard for model switching
- [ ] Session backup/restore to cloud
- [ ] `/compact` command for manual compaction
- [ ] Docker deployment option
- [ ] Webhook mode (instead of polling)

### Architecture Improvements
- [ ] Separate process for Pi execution (resilience)
- [ ] Message queue (Redis) for better scalability
- [ ] Metrics and monitoring (Prometheus)
- [ ] Structured logging (Pino, Winston)
- [ ] Healthcheck endpoints

## References

- **Pi Coding Agent**: https://github.com/badlogic/pi-mono
- **grammY Framework**: https://grammy.dev/
- **Telegram Bot API**: https://core.telegram.org/bots/api
- **Node.js**: https://nodejs.org/
- **TypeScript**: https://www.typescriptlang.org/

## License

MIT License - See LICENSE file for details
