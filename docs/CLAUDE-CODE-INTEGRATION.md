# How Happy-CLI Integrates with Claude Code CLI

**A Complete Technical Reference**

This document explains exactly how happy-cli wraps, monitors, and controls the Claude Code CLI, including all hooking mechanisms, code injection points, and communication channels.

---

## Executive Summary

**Quick Answers:**

| Question | Answer |
|----------|--------|
| **Does happy-cli wrap Claude Code?** | **YES** - Spawns Claude as a child process with custom launcher scripts |
| **Does it use hooks?** | **YES** - Intercepts `crypto.randomUUID()` and `global.fetch()` |
| **Is code injected into Claude Code?** | **YES** - Via launcher scripts that execute before Claude Code loads |
| **Is terminal output parsed?** | **PARTIAL** - Inherits stdout/stderr in local mode, parses in remote mode |
| **How many interaction points?** | **7 major channels** (detailed below) |

---

## Table of Contents

1. [Integration Architecture](#integration-architecture)
2. [Local Mode: Interactive Terminal](#local-mode-interactive-terminal)
3. [Remote Mode: SDK Control](#remote-mode-sdk-control)
4. [Code Injection Mechanisms](#code-injection-mechanisms)
5. [All Interaction Points](#all-interaction-points)
6. [Message Formats & Parsing](#message-formats--parsing)
7. [Complete Data Flow](#complete-data-flow)
8. [Actionable Implementation Guide](#actionable-implementation-guide)

---

## Integration Architecture

### High-Level Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    HAPPY-CLI WRAPPER                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Decision: Local Mode or Remote Mode?               │  │
│  └────────────┬─────────────────────────────────────────┘  │
│               │                                             │
│       ┌───────┴────────┐                                    │
│       ▼                ▼                                    │
│  ┌─────────┐      ┌──────────┐                            │
│  │  LOCAL  │      │  REMOTE  │                            │
│  │  MODE   │      │   MODE   │                            │
│  └────┬────┘      └────┬─────┘                            │
└───────┼──────────────────┼──────────────────────────────────┘
        │                  │
        │                  │
        ▼                  ▼
┌────────────────┐  ┌────────────────┐
│ Launcher       │  │ Launcher       │
│ (local)        │  │ (remote)       │
│ ├─ Hook UUID   │  │ ├─ Minimal     │
│ └─ Hook fetch  │  │ └─ hooks       │
└───────┬────────┘  └───────┬────────┘
        │                   │
        └─────────┬─────────┘
                  ▼
        ┌──────────────────┐
        │  Claude Code CLI │
        │  (@anthropic-ai) │
        └──────────────────┘
```

### Two Operating Modes

**Local Mode**: User at terminal, full PTY, interactive
- **Use Case**: Developer typing commands directly
- **Integration**: Process spawning + file descriptor hooks + JSONL file watching
- **Output**: Terminal passthrough (inherited stdio)

**Remote Mode**: Mobile app control, headless, automated
- **Use Case**: Mobile device sends commands remotely
- **Integration**: SDK-based spawning + stdin/stdout streaming
- **Output**: Structured JSON messages parsed from stdout

---

## Local Mode: Interactive Terminal

### Process Spawning

**File**: `src/claude/claudeLocal.ts:110-115`

```typescript
const child = spawn('node', [
    '/path/to/scripts/claude_local_launcher.cjs',  // Custom launcher
    '--resume', sessionId,
    '--append-system-prompt', 'You are...',
    '--mcp-config', JSON.stringify({...}),
    ...customArgs
], {
    stdio: ['inherit', 'inherit', 'inherit', 'pipe'],
    //     ┬         ┬         ┬          ┬
    //     │         │         │          └─ fd 3: Custom pipe (instrumentation)
    //     │         │         └─ fd 2: stderr → terminal
    //     │         └─ fd 1: stdout → terminal
    //     └─ fd 0: stdin ← terminal
    cwd: workingDirectory,
    signal: abortController.signal,
    env: {
        ...process.env,
        DISABLE_AUTOUPDATER: '1'
    }
});
```

**Key Characteristics**:
- **stdin/stdout/stderr**: INHERITED → User sees and interacts with Claude directly
- **fd 3**: PIPED → Happy-cli receives instrumentation data (invisible to user)
- **Not parsed**: Terminal output goes directly to user, not intercepted

### File Descriptor 3: The Instrumentation Channel

**Purpose**: Invisible side-channel for session tracking and thinking state detection

**What flows through fd 3**:

```json
{"type":"uuid","value":"aada10c6-9299-4c45-abc4-91db9c0f935d"}
{"type":"fetch-start","id":1,"hostname":"api.anthropic.com","path":"/v1/messages","method":"POST","timestamp":1731600000000}
{"type":"fetch-end","id":1,"timestamp":1731600100000}
```

**Listening Code** (`src/claude/claudeLocal.ts:118-182`):

```typescript
if (child.stdio[3]) {
    const rl = createInterface({
        input: child.stdio[3] as any,
        crlfDelay: Infinity
    });

    rl.on('line', (line) => {
        try {
            const message = JSON.parse(line);

            switch (message.type) {
                case 'uuid':
                    // Track potential session IDs
                    detectedIdsRandomUUID.add(message.value);

                    // Match with filesystem watcher
                    if (detectedIdsFileSystem.has(message.value)) {
                        resolvedSessionId = message.value;
                        opts.onSessionFound(message.value);
                    }
                    break;

                case 'fetch-start':
                    // API call started → thinking
                    activeFetches.set(message.id, {
                        hostname: message.hostname,
                        path: message.path,
                        startTime: message.timestamp
                    });
                    updateThinking(true);
                    break;

                case 'fetch-end':
                    // API call finished
                    activeFetches.delete(message.id);

                    // All fetches done? Stop thinking (with debounce)
                    if (activeFetches.size === 0) {
                        setTimeout(() => {
                            if (activeFetches.size === 0) {
                                updateThinking(false);
                            }
                        }, 500);
                    }
                    break;
            }
        } catch (e) {
            // Not JSON, ignore
        }
    });
}
```

### JSONL File Watching

**File**: `src/claude/utils/sessionScanner.ts`

**What's watched**: `~/.claude/projects/{project-hash}/{sessionId}.jsonl`

**Detection Flow**:

```
1. UUID hook fires → sends to fd 3
   detectedIdsRandomUUID.add('aada10c6-...')

2. Filesystem watcher sees new file
   detectedIdsFileSystem.add('aada10c6-...')

3. Match found!
   if (detectedIdsRandomUUID.has(id) && detectedIdsFileSystem.has(id)) {
       resolvedSessionId = id
   }

4. Start scanning JSONL file every 3 seconds
   readSessionLog(projectDir, sessionId)
   → parse each line as RawJSONLines
   → emit via onMessage callback
```

**Scanning Logic**:

```typescript
const sync = new InvalidateSync(async () => {
    const sessions = await getPendingSessions(projectDir);

    for (let sessionId of sessions) {
        const messages = await readSessionLog(projectDir, sessionId);

        for (let message of messages) {
            const key = messageKey(message);

            if (!processedMessageKeys.has(key)) {
                processedMessageKeys.add(key);
                opts.onMessage(message);  // Emit to handlers
            }
        }
    }
});

// Scan every 3 seconds
setInterval(() => sync.invalidate(), 3000);
```

**Message Deduplication**: Uses `messageKey()` to prevent re-emitting:
- User/Assistant/System: Uses `message.uuid`
- Summary: Uses `'summary:' + leafUuid + ':' + summary`

---

## Remote Mode: SDK Control

### Process Spawning

**File**: `src/claude/sdk/query.ts:330-337`

```typescript
const child = spawn('node', [
    '/path/to/scripts/claude_remote_launcher.cjs',  // Minimal launcher
    '--output-format', 'stream-json',               // CRITICAL FLAG
    '--verbose',
    '--input-format', 'stream-json',
    '--permission-prompt-tool', 'stdio',
    '--resume', sessionId,
    ...otherArgs
], {
    stdio: ['pipe', 'pipe', 'pipe'],
    //     ┬      ┬      ┬
    //     │      │      └─ fd 2: stderr → logged (DEBUG mode only)
    //     │      └─ fd 1: stdout → PARSED as stream-json
    //     └─ fd 0: stdin ← JSON messages written here
    cwd: workingDirectory,
    signal: abortController.signal,
    env: {
        ...process.env,
        CLAUDE_CODE_ENTRYPOINT: 'sdk-ts'
    }
});
```

**Key Characteristics**:
- **ALL stdio channels piped** → No terminal interaction
- **stdin**: Happy-cli writes JSON messages (user input, control responses)
- **stdout**: Happy-cli reads JSON messages (Claude responses, tool calls, etc.)
- **stderr**: Logged only in debug mode

### stdin: Writing Commands

**Format**: JSON lines, one message per line

```typescript
// Stream user messages to stdin
for await (const userMessage of messageQueue) {
    child.stdin.write(JSON.stringify({
        type: 'user',
        message: {
            role: 'user',
            content: userMessage.text
        }
    }) + '\n');
}

// Also send control responses (permissions)
child.stdin.write(JSON.stringify({
    type: 'control_response',
    response: {
        subtype: 'success',
        request_id: '12345',
        response: { behavior: 'allow' }
    }
}) + '\n');
```

### stdout: Reading Responses

**Format**: JSON lines with `--output-format stream-json`

**Parsing Code** (`src/claude/sdk/query.ts:85-123`):

```typescript
private async readMessages(): Promise<void> {
    const rl = createInterface({ input: this.childStdout });

    for await (const line of rl) {
        if (!line.trim()) continue;

        try {
            const message = JSON.parse(line) as SDKMessage;

            // Route based on message type
            if (message.type === 'control_response') {
                // Permission approval from Claude
                const handler = this.pendingControlResponses.get(message.response.request_id);
                handler?.(message.response);
            }
            else if (message.type === 'control_request') {
                // Claude asking for permission
                await this.handleControlRequest(message);
            }
            else {
                // Regular conversation message
                this.inputStream.enqueue(message);
            }
        } catch (e) {
            logger.debug('Failed to parse:', line);
        }
    }
}
```

### Permission Flow (Control Messages)

```
┌──────────────────┐         ┌──────────────────┐
│   Claude Code    │         │    Happy-CLI     │
└────────┬─────────┘         └────────┬─────────┘
         │                            │
         │ 1. Want to use tool        │
         │                            │
         │ stdout: control_request    │
         ├───────────────────────────>│
         │ {                          │
         │   type: 'control_request', │
         │   request_id: '123',       │
         │   request: {               │
         │     subtype: 'can_use_tool'│
         │     tool_name: 'bash',     │
         │     input: {...}           │
         │   }                        │
         │ }                          │
         │                            │
         │                            │ 2. Ask mobile app
         │                            │    (via WebSocket)
         │                            │
         │                            │ 3. User approves
         │                            │
         │ stdin: control_response    │
         │<───────────────────────────┤
         │ {                          │
         │   type: 'control_response',│
         │   response: {              │
         │     request_id: '123',     │
         │     response: {            │
         │       behavior: 'allow'    │
         │     }                      │
         │   }                        │
         │ }                          │
         │                            │
         │ 4. Execute tool            │
         │                            │
         │ stdout: assistant message  │
         ├───────────────────────────>│
         │ (tool use content block)   │
         │                            │
```

---

## Code Injection Mechanisms

### Launcher Script Architecture

Both local and remote modes use **launcher scripts** that execute **before** Claude Code loads, allowing happy-cli to hook into Node.js globals.

### Local Launcher: `scripts/claude_local_launcher.cjs`

**File Contents**:

```javascript
#!/usr/bin/env node
const crypto = require('crypto');
const fs = require('fs');

// Disable autoupdater
process.env.DISABLE_AUTOUPDATER = '1';

// ============================================
// Helper: Write to fd 3
// ============================================
function writeMessage(message) {
    try {
        fs.writeSync(3, JSON.stringify(message) + '\n');
    } catch (err) {
        // fd 3 not available, gracefully ignore
    }
}

// ============================================
// HOOK 1: crypto.randomUUID()
// ============================================
const originalRandomUUID = crypto.randomUUID;

// Hook global.crypto (used by most code)
Object.defineProperty(global, 'crypto', {
    configurable: true,
    enumerable: true,
    get() {
        return {
            randomUUID: () => {
                const uuid = originalRandomUUID();
                writeMessage({ type: 'uuid', value: uuid });
                return uuid;
            }
        };
    }
});

// Hook crypto module directly (used by require('crypto'))
Object.defineProperty(crypto, 'randomUUID', {
    configurable: true,
    enumerable: true,
    get() {
        return () => {
            const uuid = originalRandomUUID();
            writeMessage({ type: 'uuid', value: uuid });
            return uuid;
        };
    }
});

// ============================================
// HOOK 2: global.fetch()
// ============================================
const originalFetch = global.fetch;
let fetchCounter = 0;

global.fetch = function(...args) {
    const id = ++fetchCounter;
    const url = typeof args[0] === 'string' ? args[0] : args[0]?.url || '';
    const method = args[1]?.method || 'GET';

    // Parse URL (privacy-safe)
    let hostname = '';
    let path = '';
    try {
        const urlObj = new URL(url, 'http://localhost');
        hostname = urlObj.hostname;
        path = urlObj.pathname;
    } catch (e) {
        hostname = 'unknown';
        path = url;
    }

    // Send fetch-start event
    writeMessage({
        type: 'fetch-start',
        id,
        hostname,
        path,
        method,
        timestamp: Date.now()
    });

    // Call original fetch (non-blocking)
    const fetchPromise = originalFetch(...args);

    // Send fetch-end when done
    const sendEnd = () => {
        writeMessage({
            type: 'fetch-end',
            id,
            timestamp: Date.now()
        });
    };

    fetchPromise.then(sendEnd, sendEnd);  // Success or failure

    return fetchPromise;
};

// ============================================
// HOOK 3: Load Claude Code CLI
// ============================================
// All hooks now in place, import Claude Code
import('@anthropic-ai/claude-code/cli.js');
```

**How It Works**:

1. **Node's require cache**: When this script runs first, it establishes hooks
2. **Dynamic import**: `import('@anthropic-ai/claude-code/cli.js')` loads Claude Code
3. **Hook preservation**: All subsequent `require('crypto')` and `global.fetch` calls use hooked versions
4. **Transparent wrapping**: Hooks return original values, Claude Code unaware

### Remote Launcher: `scripts/claude_remote_launcher.cjs`

**File Contents**:

```javascript
#!/usr/bin/env node

// Minimal hook (currently unused, placeholder)
const originalSetTimeout = global.setTimeout;

global.setTimeout = function(callback, delay, ...args) {
    return originalSetTimeout(callback, delay, ...args);
};

// Preserve function properties
Object.defineProperty(global.setTimeout, 'name', { value: 'setTimeout' });
Object.defineProperty(global.setTimeout, 'length', { value: originalSetTimeout.length });

// Load Claude Code CLI
import('@anthropic-ai/claude-code/cli.js');
```

**Why Minimal?**
- SDK handles session ID via `system.init` message (no UUID hook needed)
- Thinking state inferred from message flow (no fetch hook needed)
- Placeholder for future instrumentation

---

## All Interaction Points

### Complete Reference Table

| # | Channel | Direction | Mode | Format | Purpose | File |
|---|---------|-----------|------|--------|---------|------|
| **1** | **stdin** | Happy → Claude | Local | Terminal input | User typing | `claudeLocal.ts` (inherited) |
| **2** | **stdout** | Claude → User | Local | Terminal output | Claude responses | `claudeLocal.ts` (inherited) |
| **3** | **stderr** | Claude → User | Local | Terminal output | Error messages | `claudeLocal.ts` (inherited) |
| **4** | **fd 3** | Claude → Happy | Local | JSON lines | Instrumentation | `claudeLocal.ts:118` |
| **5** | **JSONL files** | Claude → Happy | Local | JSONL | Session messages | `sessionScanner.ts` |
| **6** | **stdin** | Happy → Claude | Remote | JSON lines | User messages, control responses | `sdk/query.ts:340` |
| **7** | **stdout** | Claude → Happy | Remote | JSON lines (stream-json) | All Claude output | `sdk/query.ts:85` |

### Interaction Flow Diagram

```
LOCAL MODE:
┌─────────┐
│  User   │
└────┬────┘
     │ Types
     ▼
┌──────────────────┐
│ stdin (inherit)  │
└────┬─────────────┘
     │
     ▼
┌──────────────────────────────┐
│   Claude Code Process        │
│                              │
│  ┌────────────────────────┐ │
│  │ Hooked crypto.UUID     │ │ ──fd 3──> Happy-CLI (UUID tracking)
│  │ Hooked global.fetch    │ │ ──fd 3──> Happy-CLI (thinking state)
│  │                        │ │
│  │ Writes to:             │ │
│  │ ~/.claude/.../id.jsonl │ │ ──watched─> Happy-CLI (messages)
│  └────────────────────────┘ │
│                              │
│  stdout/stderr               │
└────┬─────────────────────────┘
     │
     ▼
┌──────────────────┐
│ Terminal Display │
└──────────────────┘

REMOTE MODE:
┌─────────────────┐
│  Mobile App     │
└────┬────────────┘
     │ WebSocket
     ▼
┌─────────────────┐
│  Happy-CLI      │
└────┬────────────┘
     │ JSON lines
     ▼
┌──────────────────────────────┐
│ stdin (pipe)                 │
└────┬─────────────────────────┘
     │
     ▼
┌──────────────────────────────┐
│   Claude Code Process        │
│   (--output-format           │
│    stream-json)              │
│                              │
│  ┌────────────────────────┐ │
│  │ SDK processes input    │ │
│  │ Generates responses    │ │
│  │ Asks permissions       │ │
│  └────────────────────────┘ │
└────┬─────────────────────────┘
     │ JSON lines
     ▼
┌──────────────────────────────┐
│ stdout (pipe)                │
└────┬─────────────────────────┘
     │
     ▼
┌─────────────────┐
│  Happy-CLI      │
│  (parses JSON)  │
└────┬────────────┘
     │
     ▼
┌─────────────────┐
│  Mobile App     │
└─────────────────┘
```

---

## Message Formats & Parsing

### Local Mode: JSONL Session Files

**Location**: `~/.claude/projects/{project-hash}/{sessionId}.jsonl`

**Schema**: `RawJSONLines` (defined in `src/claude/types.ts`)

#### User Message Format

```json
{
  "type": "user",
  "uuid": "523a67c0-a9bf-4cef-b886-d71f390b5a2f",
  "sessionId": "93a9705e-bc6a-406d-8dce-8acc014dedbd",
  "parentUuid": null,
  "isSidechain": false,
  "userType": "external",
  "cwd": "/home/user/project",
  "version": "1.0.56",
  "gitBranch": "main",
  "message": {
    "role": "user",
    "content": "say hello"
  },
  "timestamp": "2025-11-14T22:21:00.576Z"
}
```

#### Assistant Message Format

```json
{
  "type": "assistant",
  "uuid": "deae7c10-1e9a-466a-bbd0-1cd31b43d823",
  "sessionId": "93a9705e-bc6a-406d-8dce-8acc014dedbd",
  "parentUuid": "523a67c0-a9bf-4cef-b886-d71f390b5a2f",
  "message": {
    "role": "assistant",
    "content": [
      {
        "type": "text",
        "text": "Hello!"
      }
    ],
    "usage": {
      "input_tokens": 42,
      "output_tokens": 5,
      "cache_creation_input_tokens": 17755,
      "cache_read_input_tokens": 0
    }
  },
  "requestId": "req_011CRHA3FkiUoJRZqSjN67by",
  "timestamp": "2025-11-14T22:21:04.265Z"
}
```

#### Summary Message Format

```json
{
  "type": "summary",
  "summary": "Greeting exchange between user and assistant",
  "leafUuid": "deae7c10-1e9a-466a-bbd0-1cd31b43d823"
}
```

#### Parsing Code

```typescript
async function readSessionLog(projectDir: string, sessionId: string): Promise<RawJSONLines[]> {
    const filePath = join(projectDir, `${sessionId}.jsonl`);
    const file = await readFile(filePath, 'utf-8');
    const lines = file.split('\n');
    const messages: RawJSONLines[] = [];

    for (const line of lines) {
        if (!line.trim()) continue;

        try {
            const message = JSON.parse(line);
            const parsed = RawJSONLinesSchema.safeParse(message);

            if (parsed.success) {
                messages.push(parsed.data);
            } else {
                logger.debugLargeJson('[PARSE ERROR]', message);
            }
        } catch (e) {
            logger.debug('[JSON ERROR]', line);
        }
    }

    return messages;
}
```

### Remote Mode: Stream-JSON Messages

**Schema**: `SDKMessage` (defined in `src/claude/sdk/types.ts`)

#### System Init Message

```json
{
  "type": "system",
  "uuid": "...",
  "subtype": "init",
  "session_id": "93a9705e-bc6a-406d-8dce-8acc014dedbd",
  "model": "claude-opus-4-20250514",
  "cwd": "/home/user/project",
  "tools": ["bash", "edit", "glob", "grep"]
}
```

#### Assistant Tool Use

```json
{
  "type": "assistant",
  "uuid": "...",
  "message": {
    "role": "assistant",
    "content": [
      {
        "type": "tool_use",
        "id": "toolu_01ABC123",
        "name": "bash",
        "input": {
          "command": "ls -la"
        }
      }
    ]
  }
}
```

#### Control Request (Permission)

```json
{
  "type": "control_request",
  "request_id": "ctrl_12345",
  "request": {
    "subtype": "can_use_tool",
    "tool_name": "bash",
    "input": {
      "command": "rm -rf /"
    }
  }
}
```

#### Control Response (Approval)

```json
{
  "type": "control_response",
  "response": {
    "request_id": "ctrl_12345",
    "subtype": "success",
    "response": {
      "behavior": "allow",
      "updatedInput": {
        "command": "rm -rf /"
      }
    }
  }
}
```

---

## Complete Data Flow

### Local Mode End-to-End

```
1. USER TYPES IN TERMINAL
   └─> stdin (inherited) → Claude Code process

2. CLAUDE GENERATES SESSION ID
   └─> crypto.randomUUID() called
       └─> Hook intercepts
           └─> fd 3: { type: 'uuid', value: 'abc-123' }
               └─> Happy-CLI: detectedIdsRandomUUID.add('abc-123')

3. CLAUDE CREATES JSONL FILE
   └─> ~/.claude/projects/{hash}/abc-123.jsonl
       └─> Filesystem watcher detects
           └─> Happy-CLI: detectedIdsFileSystem.add('abc-123')
               └─> MATCH FOUND → Start scanning file

4. CLAUDE WRITES TO JSONL FILE
   └─> Every 3 seconds: readSessionLog()
       └─> Parse JSONL lines
           └─> Validate with RawJSONLinesSchema
               └─> Deduplicate by UUID
                   └─> onMessage(message)
                       └─> apiSession.sendClaudeSessionMessage()
                           └─> Encrypt → WebSocket → Server → Mobile

5. CLAUDE MAKES API CALL
   └─> global.fetch() called
       └─> Hook intercepts
           └─> fd 3: { type: 'fetch-start', id: 1, ... }
               └─> Happy-CLI: updateThinking(true)
                   └─> WebSocket keep-alive: { thinking: true }
                       └─> Mobile shows "thinking" indicator

6. API CALL COMPLETES
   └─> fetch promise resolves
       └─> Hook intercepts
           └─> fd 3: { type: 'fetch-end', id: 1 }
               └─> Happy-CLI: updateThinking(false)
                   └─> Mobile hides "thinking" indicator

7. CLAUDE OUTPUTS TO TERMINAL
   └─> stdout (inherited) → User's terminal
       └─> User sees response in real-time
```

### Remote Mode End-to-End

```
1. MOBILE APP SENDS MESSAGE
   └─> WebSocket → Happy-CLI server
       └─> apiSession receives encrypted message
           └─> Decrypt
               └─> messageQueue.enqueue()
                   └─> messageIterator.enqueue()

2. HAPPY-CLI WRITES TO CLAUDE STDIN
   └─> stdin.write(JSON.stringify({
         type: 'user',
         message: { role: 'user', content: 'hello' }
       }) + '\n')

3. CLAUDE PROCESSES MESSAGE
   └─> Reads from stdin (--input-format stream-json)
       └─> Generates response
           └─> Writes to stdout (--output-format stream-json)

4. CLAUDE WANTS TO USE TOOL
   └─> stdout: {
         type: 'control_request',
         request_id: '123',
         request: { subtype: 'can_use_tool', tool_name: 'bash', ... }
       }
       └─> Happy-CLI: handleControlRequest()
           └─> permissionHandler.ask()
               └─> apiSession.sendSessionEvent({ type: 'permission-request' })
                   └─> WebSocket → Mobile app
                       └─> User approves
                           └─> WebSocket → Happy-CLI
                               └─> permissionHandler.approve()
                                   └─> stdin.write({ type: 'control_response', ... })

5. CLAUDE EXECUTES TOOL
   └─> stdout: {
         type: 'assistant',
         message: {
           content: [{ type: 'tool_use', name: 'bash', input: {...} }]
         }
       }
       └─> Happy-CLI: readMessages()
           └─> Parse JSON
               └─> sdkToLogConverter.convert()
                   └─> apiSession.sendClaudeSessionMessage()
                       └─> Encrypt → WebSocket → Mobile

6. CONVERSATION ENDS
   └─> stdout: {
         type: 'result',
         duration_ms: 5432,
         usage: { input_tokens: 100, output_tokens: 50 }
       }
       └─> Happy-CLI: Close stdin, process exits
```

---

## Actionable Implementation Guide

### How to Add a New Hook

**Example**: Track when Claude Code writes files

**Step 1: Update Launcher Script** (`scripts/claude_local_launcher.cjs`)

```javascript
const originalWriteFile = fs.promises.writeFile;
fs.promises.writeFile = async function(path, data, options) {
    writeMessage({
        type: 'file-write',
        path: path.toString(),
        size: data.length,
        timestamp: Date.now()
    });

    return originalWriteFile(path, data, options);
};
```

**Step 2: Handle in Happy-CLI** (`src/claude/claudeLocal.ts`)

```typescript
case 'file-write':
    logger.debug('[FILE WRITE]', message.path, message.size);

    // Track file writes
    opts.onFileWrite?.({
        path: message.path,
        size: message.size,
        timestamp: message.timestamp
    });
    break;
```

**Step 3: Expose to API** (`src/claude/claudeLocal.ts` options)

```typescript
export interface ClaudeLocalOptions {
    // ... existing options
    onFileWrite?: (event: { path: string; size: number; timestamp: number }) => void;
}
```

### How to Parse New Message Types

**Example**: Handle new SDK message type `progress`

**Step 1: Add to Schema** (`src/claude/sdk/types.ts`)

```typescript
export interface SDKProgressMessage {
    type: 'progress';
    uuid: string;
    progress: {
        current: number;
        total: number;
        operation: string;
    };
}

export type SDKMessage =
    | SDKUserMessage
    | SDKAssistantMessage
    | SDKProgressMessage  // Add here
    | ...;
```

**Step 2: Handle in Parser** (`src/claude/sdk/query.ts`)

```typescript
for await (const line of rl) {
    const message = JSON.parse(line) as SDKMessage;

    if (message.type === 'progress') {
        // Custom handling
        logger.debug('[PROGRESS]', message.progress);
        this.onProgress?.(message.progress);
        continue;  // Don't enqueue
    }

    this.inputStream.enqueue(message);
}
```

### How to Intercept Different Stdio Streams

**Example**: Capture stderr in remote mode

**Current** (`src/claude/sdk/query.ts:349-353`):

```typescript
if (process.env.DEBUG) {
    child.stderr.on('data', (chunk) => {
        logger.debug('[CLAUDE STDERR]', chunk.toString());
    });
}
```

**Enhanced**:

```typescript
const stderrLines = createInterface({ input: child.stderr });

for await (const line of stderrLines) {
    // Parse error messages
    if (line.includes('ERROR')) {
        opts.onError?.(line);
    }

    // Log all stderr
    logger.debug('[STDERR]', line);
}
```

### How to Add New RPC Methods

**Example**: Add `git status` RPC method

**Step 1: Register Handler** (`src/modules/common/registerCommonHandlers.ts`)

```typescript
rpc.register('gitStatus', async (params: { path: string }) => {
    const result = await execAsync('git status --short', {
        cwd: params.path,
        timeout: 5000
    });

    return {
        stdout: result.stdout,
        files: parseGitStatus(result.stdout)
    };
});
```

**Step 2: Call from Mobile** (mobile app)

```typescript
const result = await sessionClient.callRpc('gitStatus', {
    path: '/home/user/project'
});

console.log('Git status:', result.files);
```

---

## Key Files Reference

| Component | File Path |
|-----------|-----------|
| **Local Mode Launcher** | `/scripts/claude_local_launcher.cjs` |
| **Remote Mode Launcher** | `/scripts/claude_remote_launcher.cjs` |
| **Local Mode Integration** | `/src/claude/claudeLocal.ts` |
| **Remote Mode Integration** | `/src/claude/claudeRemote.ts` |
| **SDK Query Implementation** | `/src/claude/sdk/query.ts` |
| **JSONL File Scanning** | `/src/claude/utils/sessionScanner.ts` |
| **Message Type Definitions** | `/src/claude/types.ts` (Local), `/src/claude/sdk/types.ts` (Remote) |
| **SDK to Log Converter** | `/src/claude/utils/sdkToLogConverter.ts` |
| **RPC Handler Manager** | `/src/api/rpc/RpcHandlerManager.ts` |
| **Common RPC Handlers** | `/src/modules/common/registerCommonHandlers.ts` |

---

## Summary

### What Happy-CLI Does

1. **Wraps**: Spawns Claude Code as child process with custom launcher scripts
2. **Hooks**: Intercepts `crypto.randomUUID()` and `global.fetch()` via launcher
3. **Injects**: Runs launcher scripts before Claude Code loads (via `import()`)
4. **Monitors**: Watches JSONL files (local) or parses stdout (remote)
5. **Communicates**: Uses fd 3 (local) or stdin/stdout pipes (remote)
6. **Syncs**: Encrypts and sends all messages to server via WebSocket
7. **Controls**: Handles permission requests from mobile app

### What Happy-CLI Does NOT Do

- ❌ Modify Claude Code's source code
- ❌ Parse terminal output in local mode (inherited directly)
- ❌ Block or prevent Claude Code from running
- ❌ Store unencrypted messages on server
- ❌ Require Claude Code API changes

### Integration Summary

| Aspect | Implementation |
|--------|----------------|
| **Code Injection** | Launcher scripts with `import()` |
| **Session Tracking** | UUID hook + filesystem watcher |
| **Thinking State** | Fetch hook + activity counter |
| **Message Capture** | JSONL file scanning (local) / stdout parsing (remote) |
| **Permission Control** | stdin/stdout control messages (remote only) |
| **Communication** | fd 3 (local) / stdin+stdout (remote) |

---

**Last Updated**: 2025-11-14
**Happy-CLI Version**: 0.11.2
**Claude Code Version**: Compatible with @anthropic-ai/claude-code 1.0.56+
