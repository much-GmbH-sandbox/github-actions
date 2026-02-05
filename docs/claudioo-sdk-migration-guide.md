# Claudioo: Claude Agent SDK Migration Guide

Migration guide for replacing `spawn('claude', ...)` with the Agent SDK's `query()` function
to enable `AskUserQuestion` support via the `canUseTool` callback.

## Why Migrate

The `AskUserQuestion` tool requires the `control_request`/`control_response` protocol,
which is only activated when `--sdk-url` is provided. The raw CLI with `--permission-mode delegate`
does NOT enable this protocol — it only changes the `$_` permission function's behavior. The
Agent SDK handles the `--sdk-url` plumbing internally and exposes a `canUseTool` callback.

## Architecture: Before and After

| Aspect               | Current (CLI)                        | After (SDK)                              |
|----------------------|--------------------------------------|------------------------------------------|
| Process model        | `spawn('claude', ...)`               | `query({ prompt: generator })`           |
| Message input        | Write JSON to stdin pipe             | Yield into AsyncGenerator                |
| Message output       | Parse stdout JSONL                   | Iterate AsyncGenerator                   |
| AskUserQuestion      | Broken (permission_denials)          | Works via `canUseTool` callback          |
| Session resume       | `--resume transcript.jsonl`          | `resume: sessionId`                      |
| MCP servers          | `--mcp-server odoo:...`              | `mcpServers: { odoo: {...} }`            |
| Working directory    | `cwd: workspacePath`                 | `cwd: workspacePath`                     |
| Auth                 | `CLAUDE_CODE_OAUTH_TOKEN`            | `ANTHROPIC_API_KEY`                      |
| System prompt        | `--append-system-prompt`             | `systemPrompt: { preset, append }`       |
| Transcript storage   | `~/.claude/projects/`                | `~/.claude/projects/` (same)             |
| Process lifecycle    | Long-lived, manual kill              | Long-lived via streaming generator       |
| Latency (first msg)  | ~5s                                  | ~5s (same — spawns CLI underneath)       |
| Latency (subsequent) | ~100ms                               | ~100ms (same — process stays alive)      |

## Key Insight: Streaming Input Mode

Using streaming input mode (`AsyncGenerator` as `prompt`), the SDK keeps a single long-lived
process per session — identical to the current architecture. No per-message startup overhead.

```typescript
// Current: spawn + stdin/stdout
const proc = spawn('claude', args);
proc.stdin.write(message);
proc.stdout.on('data', parse);

// New: SDK query() with streaming input
for await (const msg of query({ prompt: messageGenerator(), options })) {
  emit('claudeEvent', msg);
}
```

## Files to Modify

| File                         | Change                                             |
|------------------------------|----------------------------------------------------|
| `package.json`               | Add `@anthropic-ai/claude-agent-sdk` dependency    |
| `src/lib/session-manager.ts` | Replace `spawn()` with `query()`, add `canUseTool` |
| `src/lib/types.ts`           | Add SDK-related types                              |

### Files NOT Needing Changes

- SSE routes (`/api/sessions/[id]/events`) — same EventEmitter pattern
- Message API (`/api/sessions/[id]/message`) — calls `sendMessage()`
- Control API (`/api/sessions/[id]/control`) — calls `submitAnswers()`
- React components — consume SSE events unchanged
- Database schema — session storage unchanged
- Transcript persistence — keep dual JSONL storage
- Workspace management — unchanged

## Implementation

### Step 1: Install SDK

```bash
npm install @anthropic-ai/claude-agent-sdk
```

### Step 2: Message Channel Utility

Must use a proper async queue to avoid dropped messages. A naive `resolve`-based approach
loses messages when `push()` is called before the generator reaches `await`:

```typescript
function createMessageChannel() {
  const queue: SDKUserMessage[] = [];
  let notify: (() => void) | null = null;
  let closed = false;

  async function* generator(): AsyncGenerator<SDKUserMessage> {
    while (!closed) {
      while (queue.length > 0) {
        yield queue.shift()!;
      }
      if (!closed) {
        await new Promise<void>((r) => { notify = r; });
        notify = null;
      }
    }
  }

  return {
    generator: generator(),
    push: (msg: SDKUserMessage) => {
      if (closed) return;
      queue.push(msg);
      notify?.();
    },
    close: () => {
      closed = true;
      notify?.();
    },
  };
}
```

### Step 3: Updated ActiveSession Type

```typescript
export interface ActiveSession extends Session {
  abortController?: AbortController;
  messageChannel?: {
    push: (msg: SDKUserMessage) => void;
    close: () => void;
  };
  sdkSessionId?: string;  // SDK's internal UUID, NOT Claudioo's session ID
  pendingQuestionResolve?: (answers: Record<string, string>) => void;
  messages: StreamMessage[];
  pendingQuestion?: PendingQuestionState;
}
```

### Step 4: Query Loop

```typescript
private async runQueryLoop(session: ActiveSession, config: Partial<CreateSessionConfig>): Promise<void> {
  const { id, workspacePath } = session;

  const composer = new PromptComposer(path.join(process.cwd(), 'claude', 'prompts'));
  const appendPrompt = composer.compose({ /* context */ });

  try {
    for await (const msg of query({
      prompt: session.messageChannel!.generator,
      options: {
        cwd: workspacePath,
        resume: session.sdkSessionId,  // SDK's UUID, not Claudioo's
        env: { ...process.env, ANTHROPIC_API_KEY: config.apiKey },

        // ALWAYS use preset — without it Claude loses tool instructions
        systemPrompt: {
          type: 'preset',
          preset: 'claude_code',
          ...(appendPrompt ? { append: appendPrompt } : {}),
        },

        // Required to load CLAUDE.md from cwd
        settingSources: ['project'],

        mcpServers: config.odooConfig ? {
          odoo: {
            command: 'python',
            args: ['-m', 'odoo_mcp', '--url', config.odooConfig.url, /* ... */],
          },
        } : undefined,

        // CRITICAL: Use 'default', NOT 'bypassPermissions'
        // bypassPermissions auto-allows AskUserQuestion before canUseTool fires
        permissionMode: 'default',

        canUseTool: (toolName, input, { signal }) =>
          this.handleCanUseTool(session, toolName, input, signal),

        // Pass the AbortController, not the signal
        abortController: session.abortController,

        includePartialMessages: true,
      },
    })) {
      // Capture SDK session ID on init
      if (msg.type === 'system' && msg.subtype === 'init') {
        session.sdkSessionId = msg.session_id;
        await db.sessions.update(id, { sdkSessionId: msg.session_id });
      }

      // Persist and emit events
      if (msg.type === 'assistant' || msg.type === 'user' || msg.type === 'result') {
        const streamMessage = { ...msg, timestamp: new Date().toISOString() };
        session.messages.push(streamMessage);
        this.persistMessage(session, streamMessage);
      }

      this.emit('claudeEvent', id, msg);
    }

  } catch (err) {
    if (!session.abortController?.signal.aborted) {
      logger.error('SDK query error', { sessionId: id, error: String(err) });
      session.status = 'error';
      updateSessionStatus(id, 'error');
      this.emit('sessionError', id, err as Error);
    }
  } finally {
    // Auto-resume on unexpected exit
    if (session.status === 'active' && !session.abortController?.signal.aborted) {
      logger.warn('SDK query ended unexpectedly, auto-resuming', { sessionId: id });
      setTimeout(() => this.runQueryLoop(session, config), 2000);
    }
  }
}
```

### Step 5: canUseTool Handler

```typescript
private async handleCanUseTool(
  session: ActiveSession,
  toolName: string,
  input: Record<string, unknown>,
  signal: AbortSignal
): Promise<{ behavior: 'allow' | 'deny'; updatedInput?: unknown; message?: string }> {

  if (toolName === 'AskUserQuestion') {
    const questions = input.questions as AskUserQuestionInput['questions'];

    session.pendingQuestion = {
      requestId: crypto.randomUUID(),
      toolInput: { questions },
      receivedAt: new Date(),
    };

    // Emit to frontend via SSE
    this.emit('controlRequest', session.id, {
      type: 'control_request',
      request_id: session.pendingQuestion.requestId,
      request: {
        subtype: 'can_use_tool',
        tool_name: 'AskUserQuestion',
        input,
      },
    });

    // Wait for user to answer (can wait indefinitely — no SDK timeout)
    const answers = await new Promise<Record<string, string>>((resolve, reject) => {
      session.pendingQuestionResolve = resolve;
      signal.addEventListener('abort', () => reject(new Error('Aborted')));
    });

    session.pendingQuestion = undefined;
    session.pendingQuestionResolve = undefined;

    return {
      behavior: 'allow',
      updatedInput: { questions, answers },
    };
  }

  // Auto-allow all other tools
  return { behavior: 'allow', updatedInput: input };
}
```

### Step 6: sendMessage / submitAnswers

```typescript
sendMessage(sessionId: string, content: string): void {
  const session = this.sessions.get(sessionId);
  if (!session?.messageChannel) {
    throw new Error(`Session ${sessionId} not found or not active`);
  }

  session.messageChannel.push({
    type: 'user',
    message: { role: 'user', content },
  });

  session.lastActivityAt = new Date();
  updateSessionActivity(sessionId);
  incrementSessionMessageCount(sessionId);
  this.scheduleCleanup(sessionId);
}

submitAnswers(sessionId: string, answers: Record<string, string>): void {
  const session = this.sessions.get(sessionId);
  if (!session?.pendingQuestionResolve) {
    throw new Error(`No pending question for session ${sessionId}`);
  }

  session.pendingQuestionResolve(answers);
  session.lastActivityAt = new Date();
  updateSessionActivity(sessionId);
  this.scheduleCleanup(sessionId);
}
```

### Step 7: Session Close and Resume

```typescript
async close(sessionId: string): Promise<void> {
  const session = this.sessions.get(sessionId);
  if (!session) return;

  session.abortController?.abort();
  session.messageChannel?.close();

  // Clear timers, update DB (same as before)
}

async resume(sessionId: string): Promise<ActiveSession> {
  const dbSession = getSessionById(sessionId);

  const channel = createMessageChannel();
  const abortController = new AbortController();

  const session: ActiveSession = {
    ...dbSession,
    abortController,
    messageChannel: channel,
    sdkSessionId: dbSession.sdkSessionId,  // SDK's UUID from database
    messages: loadedMessages,
  };

  this.sessions.set(sessionId, session);
  this.runQueryLoop(session, { odooConfig: dbSession.odooConfig });

  return session;
}
```

## Critical Pitfalls

### 1. bypassPermissions Prevents canUseTool from Firing

With `bypassPermissions`, the CLI's internal permission check returns `{behavior: "allow"}`
for every tool, including `AskUserQuestion`. The `createCanUseTool` wrapper sees "allow" and
returns immediately — the control_request is never sent, so your `canUseTool` callback never fires.

```javascript
// From CLI internals (cli.js):
createCanUseTool(A) {
    return async(K,q,Y,z,w) => {
        let H = await $_(K,q,Y,z,w);
        if (H.behavior === "allow" || H.behavior === "deny") return H; // returns here
        // control_request is NEVER sent
        let J = await this.sendRequest({subtype: "can_use_tool"...});
    };
}
```

**Fix**: Use `permissionMode: 'default'` and auto-allow tools in your `canUseTool` callback.
The overhead per tool (~2-5ms local WebSocket round-trip) is negligible.

### 2. Message Channel Must Use Proper Queue

A naive `resolve`-based approach silently drops messages if `push()` is called:
- Before the generator reaches `await`
- Multiple times between `yield`s

Use the queue-based implementation above.

### 3. SDK Uses API Keys, Not OAuth Tokens

The SDK documentation states third-party developers cannot use `claude.ai` OAuth login.
Use `ANTHROPIC_API_KEY` from the Anthropic Console, not `CLAUDE_CODE_OAUTH_TOKEN`.

### 4. SDK Session ID ≠ Claudioo Session ID

The SDK generates its own UUID. Store it from the `system/init` message:

```typescript
if (msg.type === 'system' && msg.subtype === 'init') {
  session.sdkSessionId = msg.session_id;
  await db.sessions.update(id, { sdkSessionId: msg.session_id });
}
```

Pass it back for resume: `resume: session.sdkSessionId`

### 5. Always Use System Prompt Preset

Without `systemPrompt: { type: 'preset', preset: 'claude_code' }`, the SDK uses
a minimal system prompt and Claude loses all tool-use instructions, file operation
conventions, and security rules.

### 6. settingSources Required for CLAUDE.md

Without `settingSources: ['project']`, the SDK loads no filesystem settings.
CLAUDE.md files in the workspace are ignored.

### 7. abortController, Not abortSignal

Pass the full `AbortController`, not `controller.signal`. The SDK Options type
expects the controller.

## Verification Checklist

1. `npm install` — SDK installs without errors
2. `npm run build` — TypeScript compiles
3. Create new session — query loop starts, SDK spawns CLI
4. Send message — message yields into generator, response streams back
5. AskUserQuestion — `canUseTool` fires, `controlRequest` emits, frontend shows wizard
6. Submit answers — promise resolves, Claude continues with answers
7. Resume session — `resume: sdkSessionId` loads history correctly
8. Close session — abort controller stops query loop cleanly
9. Server restart — sessions resume from saved `sdkSessionId`
10. Multiple concurrent sessions — no cross-session leakage
