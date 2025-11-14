# Mode Switching in Happy-CLI

**Author**: Andrew Hundt <ATHundt@gmail.com>

This document explains exactly what happens when happy-cli switches between local (interactive terminal) and remote (mobile control) modes.

---

## Quick Answer

**YES** - Happy-CLI **kills and restarts** the Claude Code CLI process when switching modes.

- **Local → Remote**: Kills PTY process, spawns new SDK process with `--output-format stream-json` and `--input-format stream-json`
- **Remote → Local**: Kills SDK process, spawns new PTY process with inherited stdio

---

## Why Restart?

The two modes require **fundamentally different process configurations**:

| Aspect | Local Mode | Remote Mode |
|--------|-----------|-------------|
| **stdio** | `['inherit', 'inherit', 'inherit', 'pipe']` | `['pipe', 'pipe', 'pipe']` |
| **Launcher** | `claude_local_launcher.cjs` | `claude_remote_launcher.cjs` |
| **Flags** | Interactive (no `--output-format`) | `--output-format stream-json --input-format stream-json` |
| **stdin** | Terminal (user typing) | Piped JSON messages |
| **stdout** | Terminal display | Piped JSON messages (parsed) |
| **fd 3** | Custom instrumentation pipe | Not used |
| **User Interaction** | Direct (PTY) | Indirect (via mobile app) |

**You cannot change stdio configuration or command-line flags of a running process** - hence the restart.

---

## Mode Switching Flow

### Local → Remote Transition

```
┌──────────────────────────────────────────────────────────┐
│ USER IN TERMINAL (Local Mode)                           │
│                                                          │
│  - Claude running with inherited stdio                  │
│  - User sees output in terminal                         │
│  - JSONL files scanned every 3 seconds                  │
└──────────────────┬───────────────────────────────────────┘
                   │
                   │ TRIGGER: Mobile app sends message
                   │         OR RPC 'switch' called
                   │
                   ▼
┌──────────────────────────────────────────────────────────┐
│ SHUTDOWN SEQUENCE (claudeLocalLauncher.ts:38-50)        │
│                                                          │
│  1. doSwitch() called                                    │
│  2. exitReason = 'switch'                                │
│  3. session.queue.reset()                                │
│  4. processAbortController.abort()                       │
│     └─> Triggers abort signal on spawn()                │
│         └─> Claude process receives SIGTERM              │
│             └─> Process killed                           │
│                                                          │
│  5. await exutFuture.promise (wait for full exit)       │
│  6. scanner.cleanup() (stop file watching)              │
│  7. return 'switch'                                      │
└──────────────────┬───────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────┐
│ LOOP DETECTS 'switch' (loop.ts:66-76)                   │
│                                                          │
│  if (reason === 'exit') {                                │
│      return; // Normal exit                              │
│  }                                                       │
│                                                          │
│  // reason === 'switch'                                  │
│  mode = 'remote';                                        │
│  onModeChange(mode);                                     │
│  continue; // Loop again with remote mode                │
└──────────────────┬───────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────┐
│ START REMOTE MODE (claudeRemoteLauncher.ts:327)         │
│                                                          │
│  const remoteResult = await claudeRemote({              │
│      sessionId: session.sessionId,  // SAME SESSION     │
│      path: session.path,                                 │
│      allowedTools: [...],                                │
│      canCallTool: permissionHandler.handleToolCall,      │
│      nextMessage: async () => { ... },                   │
│      signal: abortController.signal,                     │
│      ...                                                 │
│  });                                                     │
└──────────────────┬───────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────┐
│ SPAWN NEW CLAUDE PROCESS (sdk/query.ts:330)             │
│                                                          │
│  spawn('node', [                                         │
│      'claude_remote_launcher.cjs',                       │
│      '--output-format', 'stream-json',  ← NEW FLAG      │
│      '--input-format', 'stream-json',   ← NEW FLAG      │
│      '--resume', sessionId,             ← SAME SESSION  │
│      ...args                                             │
│  ], {                                                    │
│      stdio: ['pipe', 'pipe', 'pipe'],   ← NEW CONFIG    │
│      signal: abortController.signal                      │
│  })                                                      │
└──────────────────┬───────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────┐
│ REMOTE MODE ACTIVE                                       │
│                                                          │
│  - Mobile app controls via stdin JSON messages           │
│  - Happy-CLI parses stdout JSON responses                │
│  - Session resumed from same sessionId                   │
│  - FULL CONTEXT PRESERVED (Claude's --resume)            │
└──────────────────────────────────────────────────────────┘
```

### Remote → Local Transition

```
┌──────────────────────────────────────────────────────────┐
│ MOBILE APP CONTROL (Remote Mode)                        │
│                                                          │
│  - Claude running with piped stdio                       │
│  - Messages via stdin/stdout JSON                        │
│  - Permission requests handled remotely                  │
└──────────────────┬───────────────────────────────────────┘
                   │
                   │ TRIGGER: User presses space-space
                   │         OR RPC 'switch' called
                   │
                   ▼
┌──────────────────────────────────────────────────────────┐
│ SHUTDOWN SEQUENCE (claudeRemoteLauncher.ts:86-92)       │
│                                                          │
│  1. doSwitch() called                                    │
│  2. exitReason = 'switch'                                │
│  3. abortController.abort()                              │
│     └─> Triggers abort signal on SDK spawn()            │
│         └─> Claude process receives SIGTERM              │
│             └─> Process killed                           │
│                                                          │
│  4. await abortFuture.promise (wait for exit)           │
│  5. messageQueue.flush() (send any pending)             │
│  6. messageQueue.destroy()                               │
│  7. permissionHandler.reset()                            │
│  8. inkInstance.unmount() (cleanup UI)                   │
│  9. return 'switch'                                      │
└──────────────────┬───────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────┐
│ LOOP DETECTS 'switch' (loop.ts:81-91)                   │
│                                                          │
│  if (reason === 'exit') {                                │
│      return; // Normal exit                              │
│  }                                                       │
│                                                          │
│  // reason === 'switch'                                  │
│  mode = 'local';                                         │
│  onModeChange(mode);                                     │
│  continue; // Loop again with local mode                 │
└──────────────────┬───────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────┐
│ START LOCAL MODE (claudeLocalLauncher.ts:94)            │
│                                                          │
│  await claudeLocal({                                     │
│      path: session.path,                                 │
│      sessionId: session.sessionId,  // SAME SESSION     │
│      onSessionFound: handleSessionStart,                 │
│      onThinkingChange: session.onThinkingChange,         │
│      abort: processAbortController.signal,               │
│      ...                                                 │
│  });                                                     │
└──────────────────┬───────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────┐
│ SPAWN NEW CLAUDE PROCESS (claudeLocal.ts:110)           │
│                                                          │
│  spawn('node', [                                         │
│      'claude_local_launcher.cjs',                        │
│      '--resume', sessionId,         ← SAME SESSION      │
│      '--append-system-prompt', '...',                    │
│      ...args                                             │
│  ], {                                                    │
│      stdio: ['inherit', 'inherit',  ← NEW CONFIG        │
│             'inherit', 'pipe'],                          │
│      signal: processAbortController.signal               │
│  })                                                      │
└──────────────────┬───────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────┐
│ LOCAL MODE ACTIVE                                        │
│                                                          │
│  - User types commands in terminal                       │
│  - Claude output displays directly                       │
│  - Session resumed from same sessionId                   │
│  - FULL CONTEXT PRESERVED (Claude's --resume)            │
└──────────────────────────────────────────────────────────┘
```

---

## Key Implementation Details

### Abort Signal Mechanism

**Local Mode** (`src/claude/claudeLocal.ts:110-115`):
```typescript
const child = spawn('node', [claudeCliPath, ...args], {
    stdio: ['inherit', 'inherit', 'inherit', 'pipe'],
    signal: opts.abort,  // ← AbortSignal passed here
    cwd: opts.path,
    env,
});
```

**Remote Mode** (`src/claude/sdk/query.ts:330-337`):
```typescript
const child = spawn('node', [...executableArgs, pathToClaudeCodeExecutable, ...args], {
    cwd,
    stdio: ['pipe', 'pipe', 'pipe'],
    signal: config.options?.abort,  // ← AbortSignal passed here
    env: { ...process.env }
}) as ChildProcessWithoutNullStreams
```

### Abort Trigger Locations

**Local Launcher** (`src/claude/claudeLocalLauncher.ts:53-62`):
```typescript
async function doSwitch() {
    logger.debug('[local]: doSwitch');

    // Switching to remote mode
    if (!exitReason) {
        exitReason = 'switch';
    }

    // Abort the process
    await abort();  // ← Triggers processAbortController.abort()
}
```

**Remote Launcher** (`src/claude/claudeRemoteLauncher.ts:86-92`):
```typescript
async function doSwitch() {
    logger.debug('[remote]: doSwitch');
    if (!exitReason) {
        exitReason = 'switch';
    }
    await abort();  // ← Triggers abortController.abort()
}
```

### Process Cleanup

**SDK Cleanup** (`src/claude/sdk/query.ts:356-363`):
```typescript
const cleanup = () => {
    if (!child.killed) {
        child.kill('SIGTERM');  // ← Explicit SIGTERM if not already dead
    }
}

config.options?.abort?.addEventListener('abort', cleanup)
process.on('exit', cleanup)
```

---

## Session Continuity

### How Context is Preserved

Even though the **process is killed and restarted**, the **conversation context is preserved** via Claude Code's `--resume` flag:

```typescript
// Both modes pass the same sessionId
const args = ['--resume', session.sessionId, ...]
```

**What happens with `--resume`**:

1. Claude Code creates a **new session file** with a **new session ID**
2. The new file contains the **complete history** from the original session
3. All message `sessionId` fields are **rewritten** to the new ID
4. A summary line is prepended to the file
5. Claude maintains **full conversation context**

**Example**:

```
Original session: aada10c6-9299-4c45-abc4-91db9c0f935d.jsonl
                 ↓ --resume aada10c6-...
New session:     1433467f-ff14-4292-b5b2-2aac77a808f0.jsonl
                 (contains full history with updated sessionIds)
```

**Files** (`src/claude/session.ts:consumeOneTimeFlags()`):

After the first spawn, the `--resume` flag is **removed** from `claudeArgs` to prevent re-resuming in subsequent spawns within the same mode.

---

## What is NOT Preserved Across Switches

While session context is preserved, these are **reset**:

| Item | What Happens | Why |
|------|--------------|-----|
| **Process PID** | New PID assigned | New process spawned |
| **stdio file descriptors** | New FDs created | New stdio configuration |
| **Permission handler state** | Reset | `permissionHandler.reset()` called |
| **Message queue** | Flushed and destroyed | `messageQueue.flush()` + `destroy()` |
| **Thinking state** | Reset to false | New process hasn't started thinking yet |
| **Active tool calls** | Terminated | `generateInterruptedToolResult()` sent |
| **File descriptor 3 hooks** | Re-established | New launcher script runs |

---

## Switch Triggers

### Local → Remote Triggers

1. **Message from mobile app** (`claudeLocalLauncher.ts:68-71`):
   ```typescript
   session.queue.setOnMessage((message: string, mode) => {
       doSwitch(); // Switch when any message received
   });
   ```

2. **RPC 'switch' call** (`claudeLocalLauncher.ts:67`):
   ```typescript
   session.client.rpcHandlerManager.registerHandler('switch', doSwitch);
   ```

3. **RPC 'abort' call** (`claudeLocalLauncher.ts:66`):
   ```typescript
   session.client.rpcHandlerManager.registerHandler('abort', doAbort);
   ```

4. **Existing queue messages** (`claudeLocalLauncher.ts:74-76`):
   ```typescript
   if (session.queue.size() > 0) {
       return 'switch'; // Immediately switch if messages waiting
   }
   ```

### Remote → Local Triggers

1. **Double space press** (`claudeRemoteLauncher.ts:50-54`):
   ```typescript
   onSwitchToLocal: () => {
       logger.debug('[remote]: Switching to local mode via double space');
       doSwitch();
   }
   ```

2. **RPC 'switch' call** (`claudeRemoteLauncher.ts:96`):
   ```typescript
   session.client.rpcHandlerManager.registerHandler('switch', doSwitch);
   ```

3. **RPC 'abort' call** (`claudeRemoteLauncher.ts:95`):
   ```typescript
   session.client.rpcHandlerManager.registerHandler('abort', doAbort);
   ```

---

## Performance Implications

### Process Restart Overhead

**What happens during restart**:

1. **Kill old process**: ~10-50ms (SIGTERM handling)
2. **Spawn new process**: ~100-200ms (Node.js startup)
3. **Load Claude Code**: ~200-500ms (module loading)
4. **Resume session**: ~500-1000ms (read JSONL, recreate context)

**Total**: ~1-2 seconds typical

### Optimization: One-Time Flags

To avoid re-resuming the same session multiple times within a mode:

```typescript
// src/claude/session.ts:consumeOneTimeFlags()
consumeOneTimeFlags(): void {
    this.claudeArgs = this.claudeArgs.filter((arg, i) =>
        !(arg === '--resume' && this.claudeArgs[i-1] === '--resume')
    );
}
```

Called after first spawn in both launchers:
- `claudeLocalLauncher.ts:108`
- `claudeRemoteLauncher.ts:396`

---

## Alternative Approaches (Not Implemented)

### Why Not Keep Process Running?

**Option**: Use a single long-running Claude process and switch input/output handling

**Problems**:
1. **Cannot change stdio**: Once stdio is set at spawn, it's immutable
2. **Cannot change CLI flags**: `--output-format stream-json` is startup-only
3. **Process state confusion**: Local vs remote have different expectations
4. **Tool permission handling**: Different mechanisms (UI vs RPC)
5. **Complexity**: Managing dual personalities in one process is error-prone

**Conclusion**: Clean restart is simpler, more reliable, and only adds ~1-2s overhead

### Why Not Use Single Mode?

**Option**: Always use remote mode (SDK), disable local mode

**Problems**:
1. **User experience**: Developers want direct terminal interaction
2. **Debugging**: Harder to see Claude's output directly
3. **Latency**: Extra hop through happy-cli for local use
4. **Dependency**: Requires happy-cli infrastructure even for local work

**Conclusion**: Supporting both modes provides best experience for different use cases

---

## Code References

| Component | File | Lines |
|-----------|------|-------|
| **Main Loop** | `src/claude/loop.ts` | 37-94 |
| **Local Launcher** | `src/claude/claudeLocalLauncher.ts` | 7-142 |
| **Remote Launcher** | `src/claude/claudeRemoteLauncher.ts` | 26-459 |
| **Local Spawn** | `src/claude/claudeLocal.ts` | 110-115 |
| **Remote Spawn** | `src/claude/sdk/query.ts` | 330-337 |
| **Abort Handling** | `src/claude/sdk/query.ts` | 356-363 |
| **Session Class** | `src/claude/session.ts` | - |

---

## Summary

**Mode Switching Behavior**:

✅ **KILLS** the Claude Code process
✅ **SPAWNS** new process with different configuration
✅ **PRESERVES** conversation context via `--resume`
✅ **RESETS** process state, permissions, and queues
✅ **TAKES** ~1-2 seconds typical

**Why Necessary**:
- Different stdio configurations (inherited vs piped)
- Different CLI flags (`--output-format stream-json`, etc.)
- Different launchers (hooks differ)
- Different input/output handling (terminal vs JSON)

**Context Preservation**:
- Claude's `--resume` flag reads previous session JSONL
- Creates new session file with full history
- Conversation continues seamlessly from user's perspective

---

**Last Updated**: 2025-11-14
**Author**: Andrew Hundt <ATHundt@gmail.com>
