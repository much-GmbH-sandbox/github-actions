# Claude Code Web Architecture

Verified empirically from inside a live Claude Code Web container (session `session_01Nsqr6XCs3pscnDp62qVQzH`), February 2026.

## Container Runtime Stack

```
┌──────────────────────────────────────────────────────────────────────┐
│  Browser (claude.ai)                                                  │
└──────────────┬────────────────────────────────────▲──────────────────┘
               │ HTTPS                              │ HTTPS
               ▼                                    │
┌──────────────────────────────────────────────────────────────────────┐
│  api.anthropic.com — Session Ingress Service                          │
│                                                                       │
│  Endpoints:                                                           │
│  ├─ /v1/session_ingress/ws/{session_id}       ← WebSocket relay       │
│  ├─ /v2/.../session/{session_id}/events       ← Event posting         │
│  ├─ /v1/.../session/{session_id}/git_proxy/   ← Git proxy             │
│  └─ /v1/.../session/{session_id}              ← Session persist/fetch │
│                                                                       │
│  Browser and CLI both connect here.                                   │
│  Anthropic relays messages between them.                              │
└──────────────┬────────────────────────────────────▲──────────────────┘
               │ WebSocket (wss://)                 │ WebSocket
               │ (CLI connects OUTBOUND)            │
               ▼                                    │
┌──────────────────────────────────────────────────────────────────────┐
│  Docker Container (gVisor runtime, ephemeral, per-session)            │
│  Ubuntu 24.04.3 LTS, Linux runsc 4.4.0 kernel                        │
│                                                                       │
│  PID 1: /process_api (Rust, stripped ELF binary)                      │
│  ├── Listens on 0.0.0.0:2024 (management WebSocket)                  │
│  ├── --memory-limit-bytes 17179869184 (16 GiB)                        │
│  ├── --oom-polling-period-ms 100                                      │
│  ├── --block-local-connections                                        │
│  ├── cgroups: pids, memory, job, devices, cpuset, cpuacct, cpu        │
│  └── Spawns environment-manager via shell (PID 23 → PID 25)          │
│                                                                       │
│  PID 25: /usr/local/bin/environment-manager (Go, with debug_info)     │
│  ├── Reads session JSON from stdin (5,450 bytes)                      │
│  ├── Parses V0 input format                                           │
│  ├── Initializes git proxy JWT authentication                         │
│  ├── Clones repo: git fetch --depth 50 (~883ms network-bound)         │
│  ├── Starts local git proxy on 127.0.0.1:{dynamic_port}              │
│  ├── Installs Python 3.11 (~116ms), Node 20 (~383ms)                 │
│  ├── Starts MCP codesign server on 127.0.0.1:{port} (~5ms)           │
│  ├── Reports events to api.anthropic.com/v2/session_ingress           │
│  └── Launches claude CLI with args from session config                │
│                                                                       │
│  PID ~157: claude (Node.js — @anthropic-ai/claude-code v2.1.19)       │
│  ├── Connects OUTBOUND to wss://api.anthropic.com/.../ws/{session}    │
│  ├── Auth via CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR=3            │
│  ├── Receives user messages via WebSocket relay                       │
│  ├── Sends conversation to Claude Messages API                        │
│  ├── Executes tools locally (Bash, Read/Write, etc.)                  │
│  ├── Streams responses back via WebSocket                             │
│  ├── Dual-writes transcript: local JSONL + cloud persistence          │
│  └── Handles autocompact/summarization internally                     │
│                                                                       │
│  Git Proxy: 127.0.0.1:{port} → api.anthropic.com git_proxy endpoint  │
│  └── Branch restrictions enforced at API layer (claude/* prefix)      │
│                                                                       │
│  Egress Proxy: JWT-authenticated HTTPS proxy                          │
│  └── Explicit domain allowlist (npm, pypi, GitHub, etc.)              │
│                                                                       │
│  MCP Codesign Server: 127.0.0.1:{port}                                │
│  └── Ed25519 signatures for git commits                               │
└──────────────────────────────────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────────────────┐
│  api.anthropic.com — Claude Messages API                              │
│  Claude Opus 4.5 processes and responds                               │
└──────────────────────────────────────────────────────────────────────┘
```

## Cold Start Timeline

Reconstructed from environment-manager logs and `/proc` start times:

| Offset    | Event                                | Duration |
|-----------|--------------------------------------|----------|
| T+0.000s  | Container created                    |          |
| T+0.413s  | PID 1 process_api starts             |          |
| T+1.272s  | PID 25 environment-manager starts    |          |
| T+1.283s  | Reads session JSON from stdin        | 11ms     |
| T+1.762s  | Setup phase complete (parse, auth)   | 479ms    |
|           | **— Parallel phase begins —**        |          |
| T+3.046s  | Environment init starts              |          |
| T+3.217s  | ├─ git init                          | 39ms     |
| T+3.255s  | ├─ git remote add                    | 22ms     |
| T+4.180s  | ├─ git fetch (depth 50)              | 883ms    |
| T+4.207s  | ├─ git checkout                      | 25ms     |
| T+4.897s  | ├─ Repo clone total                  | 1,682ms  |
| T+4.898s  | ├─ Git proxy started                 |          |
| T+5.052s  | ├─ Python 3.11 installed             | 116ms    |
| T+5.319s  | └─ Node 20 installed                 | 383ms    |
| T+5.320s  | Environment init done                | 2,272ms  |
| T+6.372s  | Plugin marketplace done              | 3,326ms  |
|           | **— Parallel phase ends —**          |          |
| T+6.373s  | Claude Code executor created         |          |
| T+6.382s  | `claude` CLI launched                |          |

**Total cold start: ~6.4 seconds** from container creation to CLI launch.
Critical path: plugin marketplace installation (3.3s), not git clone (1.7s).

### Fast Resume (Same Container)

When a session is resumed on the same container:
- Skips git clone ("Fast resume: Using existing repository clones")
- Updates git remote URLs to new proxy port
- Skips language installation
- **Total resume time: ~1.5s**

This container has been through 5 lifecycle events: 1 `new` + 4 `resume`.

## Three-Token Authentication Model

No GitHub token ever reaches the container. Three distinct tokens manage different concerns:

### 1. Session Ingress JWT (`sk-ant-si-...`)

```
iss: session-ingress
aud: anthropic-api
session_id: session_01Nsqr6XCs3pscnDp62qVQzH
organization_uuid: eecf82a7-...
account_uuid: c8fd1ea8-...
application: ccr
exp: 4 hours after issuance (refreshable)
```

Used for: WebSocket relay authentication, git proxy requests, session event posting, cloud transcript persistence.

### 2. Anthropic OAuth Token (via file descriptor 4)

Used for: Claude Messages API calls from the CLI.

### 3. Egress Proxy JWT

```
iss: anthropic-egress-control
exp: 4 hours after issuance
allowed_hosts: [npm, pypi, github, ...]
```

Used for: All outbound HTTPS traffic. JWT-authenticated proxy with explicit domain allowlist.

## Session Persistence: Dual-Write Architecture

Every message is persisted to two locations simultaneously:

### Local JSONL File

```
~/.claude/projects/{sanitized-project-path}/{session-uuid}.jsonl
```

Written via synchronous `appendFileSync` after every event. Append-only log.

### Cloud Persistence

```
PUT https://api.anthropic.com/v1/session_ingress/session/{session_id}
Authorization: Bearer {session-ingress-jwt}
Content-Type: application/json
Last-Uuid: {previous-message-uuid}   ← optimistic concurrency
```

Retry logic: Up to 10 attempts with 500ms base exponential backoff, capped at 8 seconds.
Cloud persistence failure is **fatal** in remote mode — CLI exits with code 1.

### What Gets Persisted Remotely

The `Ru()` function filters: `user`, `assistant`, `attachment`, `system`, `progress` events.
Tool progress (bash streaming) is persisted but filtered out during resume loading.

### Resume Flow

1. Orchestrator creates new container with same `session_id`
2. CLI starts with `--resume=https://api.anthropic.com/v1/session_ingress/session/{session_id}`
3. CLI calls `FqK()` which downloads cloud transcript via GET
4. Writes to local JSONL (recreates local state)
5. Loads messages into memory via `Ja()` → `CBA()` (filters to user/assistant)
6. Autocompact fires at next API call if context exceeds threshold

## Git Proxy Architecture

### Initial Clone

```
https://{session-ingress-jwt}@api.anthropic.com/v1/session_ingress/session/{id}/git_proxy/{org}/{repo}.git
```

### After Clone (Local Proxy)

```
http://local_proxy@127.0.0.1:{dynamic_port}/git/{org}/{repo}
```

The local proxy (run by environment-manager) forwards to the API git proxy endpoint.
A secure random bearer token authenticates local proxy requests.

### Branch Restrictions

Enforced at the API proxy layer, not in the container:

```
$ git push origin test-non-claude-branch       → HTTP 403
$ git push origin claude/wrong-suffix-xyz      → HTTP 403
$ git push origin claude/correct-suffix-abcDE  → Success
```

Both the `claude/` prefix AND the session-specific suffix must match.

## Container Lifecycle

### Maximum Lifetime: 4 Hours

Both JWTs (session ingress and egress proxy) expire exactly 4 hours after container creation.
Tokens can be refreshed for active sessions (observed: token refreshed 11+ hours after creation).

### Termination Triggers

| Trigger          | Mechanism                                                     |
|------------------|---------------------------------------------------------------|
| Session ends     | User closes session, orchestrator kills container             |
| Token expiry     | 4-hour hard limit, tokens become invalid                      |
| Idle timeout     | Enforced externally by orchestrator (not visible from inside) |
| OOM              | process_api polls every 100ms, kills at 16 GiB               |

### Container Runtime

- **Docker** is the orchestration layer (`/.dockerenv` exists, zero-byte marker file)
- **gVisor (runsc)** is the OCI runtime (`Linux runsc 4.4.0` kernel)
- gVisor intercepts syscalls in userspace for stronger isolation
- Total memory: ~21 GiB available, 16 GiB enforcement ceiling

## Cost Tracking

Cost is calculated **client-side** in the CLI using API response usage data and a local pricing table.

### Pricing Table (per million tokens)

| Tier                    | Input  | Output  | Cache Write | Cache Read | Web Search |
|-------------------------|--------|---------|-------------|------------|------------|
| Haiku                   | $0.80  | $4.00   | $1.00       | $0.08      | $0.01      |
| Sonnet (≤200k ctx)      | $3.00  | $15.00  | $3.75       | $0.30      | $0.01      |
| Sonnet (>200k ctx)      | $6.00  | $22.50  | $7.50       | $0.60      | $0.01      |
| Opus                    | $15.00 | $75.00  | $18.75      | $1.50      | $0.01      |

### Budget Enforcement

```javascript
if (wG() >= maxBudgetUsd) {
    yield { type: "result", subtype: "error_max_budget_usd", is_error: false };
    return;  // Hard stop between turns
}
```

Check happens between turns. Current turn completes, but no new turn begins.

## System Prompt Assembly

The CLI assembles the system prompt via `zH1()`:

```
overrideSystemPrompt     → replaces everything (internal only)
   ↓ (if null)
Agent system prompt      → replaces custom + default
   ↓ (if null)
customSystemPrompt       → from --system-prompt, REPLACES default entirely
   ↓ (if null)
defaultSystemPrompt      → built-in + CLAUDE.md + memory files
   ↓ (always appended)
appendSystemPrompt       → from --append-system-prompt, ADDED to the end
```

Claude Code Web uses `--append-system-prompt` to inject task-specific context
while preserving the full default system prompt.

## Autocompact / Context Management

Autocompact triggers when context usage exceeds a threshold before an API call:

1. **Microcompact**: Fast, small reduction
2. **Autocompact**: Full summarization
3. **Server-side compaction**: Uses `context_management` beta API parameter (`context-management-2025-06-27`)

Post-compaction, a note is injected referencing the full transcript file path.
`PreCompact` hooks allow custom instructions before compaction.

## Stream-JSON Output Event Types

Events emitted on stdout in `--output-format stream-json --verbose` mode:

| Type              | Purpose                                        | Persist? |
|-------------------|------------------------------------------------|----------|
| `assistant`       | Complete assistant message (text or tool_use)  | Yes      |
| `user`            | User message or tool_result                    | Yes      |
| `result`          | Turn finished — final event per request cycle  | Yes      |
| `system`          | Status updates, compaction notices             | No       |
| `tool_progress`   | Bash command running, elapsed time             | No       |
| `control_request` | Permission prompt for SDK consumers            | No       |
| `stream_event`    | Raw API stream events (partial messages)       | No       |

### Result Subtypes

| Subtype                    | Meaning                       |
|----------------------------|-------------------------------|
| `success`                  | Normal completion             |
| `error_max_turns`          | Hit turn limit                |
| `error_max_budget_usd`     | Budget exceeded               |
| `error_during_execution`   | Tool or API error mid-stream  |

## control_request Protocol

The `control_request`/`control_response` protocol enables interactive tools like `AskUserQuestion`.

**Activation**: Only when `--sdk-url` is provided. The `mn2()` function returns:
- `K.createCanUseTool(Y)` when `sdkUrl` is set (emits `control_request`)
- `$_` (basic permission check) otherwise — **no `control_request` emitted**

`--permission-mode delegate` does NOT activate this protocol. It only affects the `$_` permission function's behavior.

### Flow

```
1. Claude generates tool_use for AskUserQuestion
2. CLI emits control_request on stdout/WebSocket:
   { type: "control_request", request_id: "req_xyz", request: {
       subtype: "can_use_tool", tool_name: "AskUserQuestion", input: {...}
   }}
3. Consumer sends control_response on stdin/WebSocket:
   { type: "control_response", response: {
       subtype: "success", request_id: "req_xyz",
       response: { behavior: "allow", updatedInput: { questions: [...], answers: {...} } }
   }}
4. CLI calls AskUserQuestion.call() with updatedInput
5. mapToolResultToToolResultBlockParam formats:
   "User has answered your questions: "Q1"="A1", "Q2"="A2". You can now continue..."
```

### MultiSelect Answers

Comma-separated string: `"Dark mode, Notifications, Analytics"`

## AskUserQuestion Tool Schema

### Input (from Claude)

```json
{
  "questions": [{
    "question": "Which database?",
    "header": "Database Selection",
    "options": [
      { "label": "PostgreSQL", "description": "Relational" },
      { "label": "MongoDB", "description": "Document-oriented" }
    ],
    "multiSelect": false
  }]
}
```

### Answers (from user, via updatedInput)

```json
{
  "questions": [/* same as above */],
  "answers": {
    "Which database?": "PostgreSQL"
  }
}
```

Constraints: 1-4 questions, unique question texts, unique option labels per question.
An "Other" option is always appended by the CLI's UI for freeform input.
