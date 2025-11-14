# Happy CLI Architecture: Security, Syncing, and Claude Code Integration

This document provides a comprehensive overview of how Happy CLI implements security, synchronizes sessions, and integrates with the Claude Code CLI.

## Table of Contents

1. [System Overview](#system-overview)
2. [Security Architecture](#security-architecture)
3. [Session Syncing & Real-time Communication](#session-syncing--real-time-communication)
4. [Claude Code CLI Integration](#claude-code-cli-integration)
5. [Data Flow Diagrams](#data-flow-diagrams)

---

## System Overview

Happy CLI is a command-line wrapper for Claude Code that enables:
- **Remote control** of Claude sessions from mobile devices
- **Session sharing** across multiple devices
- **End-to-end encryption** for all communications
- **Dual-mode operation**: interactive terminal and remote mobile control

### Component Architecture

```
┌─────────────────┐
│  Mobile App     │ (React Native)
│  (handy)        │
└────────┬────────┘
         │
         │ WebSocket + Encryption
         │
┌────────▼────────┐
│  Server         │ (Node.js + Prisma)
│  (handy-server) │ https://api.happy-servers.com/
└────────┬────────┘
         │
         │ WebSocket + Encryption
         │
┌────────▼────────┐
│  CLI            │ (TypeScript)
│  (happy-cli)    │
└────────┬────────┘
         │
         │ SDK/PTY
         │
┌────────▼────────┐
│  Claude Code    │ (@anthropic-ai/claude-code)
└─────────────────┘
```

---

## Security Architecture

Happy CLI implements **defense-in-depth** with multiple encryption layers, challenge-response authentication, and secure key management.

### 1. Authentication Flow

**Method**: Challenge-Response with Ed25519 Digital Signatures

**Process** (`src/api/auth.ts` + `src/ui/auth.ts`):

```
┌──────────┐                           ┌──────────┐
│  Client  │                           │  Server  │
└─────┬────┘                           └─────┬────┘
      │                                      │
      │ 1. Generate ephemeral keypair        │
      │    (32-byte random secret)           │
      │                                      │
      │ 2. POST /v1/auth/request             │
      │    { publicKey, method }             │
      ├─────────────────────────────────────>│
      │                                      │
      │ 3. Server encrypts response          │
      │<─────────────────────────────────────┤
      │    ephemeralPubKey + nonce + encrypted│
      │                                      │
      │ 4. Decrypt with Box encryption       │
      │    Extract: secret OR (pubKey + key) │
      │                                      │
      │ 5. Sign random challenge             │
      │    signature = sign(challenge, secret)│
      │                                      │
      │ 6. POST /v1/auth/token               │
      │    { challenge, publicKey, signature }│
      ├─────────────────────────────────────>│
      │                                      │
      │ 7. Verify signature                  │
      │<─────────────────────────────────────┤
      │    Returns: { token }                │
      │                                      │
```

**Key Components**:

1. **Ephemeral Key Generation** (`src/ui/auth.ts:189-191`):
   ```typescript
   const secret = new Uint8Array(randomBytes(32));
   const keypair = tweetnacl.box.keyPair.fromSecretKey(secret);
   ```

2. **QR Code Authentication**:
   - Mobile scans QR code containing ephemeral public key
   - Server encrypts credentials for this specific key
   - Provides forward secrecy (keys used once)

3. **Challenge-Response** (`src/api/encryption.ts:128-138`):
   ```typescript
   const keypair = tweetnacl.sign.keyPair.fromSeed(secret);
   const signature = tweetnacl.sign.detached(challenge, keypair.secretKey);
   return { challenge, publicKey: keypair.publicKey, signature };
   ```

**Security Properties**:
- ✅ Prevents replay attacks (random challenge)
- ✅ Mutual authentication (signature verification)
- ✅ Forward secrecy (ephemeral keys)
- ✅ No password storage

### 2. Encryption Mechanisms

Happy CLI uses **three encryption systems** depending on context:

#### A. Legacy Encryption (TweetNaCl SecretBox)

**Location**: `src/api/encryption.ts:45-81`

```typescript
// XSalsa20-Poly1305 authenticated encryption
const nonce = randomBytes(24);
const encrypted = tweetnacl.secretbox(data, nonce, secret);
// Format: nonce (24 bytes) + encrypted_data
```

**Properties**:
- Cipher: XSalsa20 (stream cipher)
- MAC: Poly1305 (authentication)
- Key: 32-byte symmetric secret
- Nonce: 24 random bytes

#### B. Data Key Encryption (AES-256-GCM)

**Location**: `src/api/encryption.ts:83-126`

```typescript
// AES-256-GCM with random nonce
const cipher = crypto.createCipheriv('aes-256-gcm', key, nonce);
const encrypted = Buffer.concat([cipher.update(data), cipher.final()]);
const authTag = cipher.getAuthTag();
// Format: version(1) + nonce(12) + encrypted + authTag(16)
```

**Properties**:
- Cipher: AES-256 (block cipher)
- Mode: GCM (Galois/Counter Mode)
- Authentication: 16-byte auth tag
- Nonce: 12 random bytes
- Better performance than TweetNaCl

#### C. Public Key Encryption (Box - Curve25519)

**Location**: `src/api/encryption.ts:140-155`

```typescript
// Ephemeral keypair for each encryption
const ephemeralKeypair = tweetnacl.box.keyPair();
const nonce = randomBytes(24);
const encrypted = tweetnacl.box(data, nonce, recipientPubKey, ephemeralSecret);
// Format: ephemeralPubKey(32) + nonce(24) + encrypted_data
```

**Properties**:
- ECDH: Curve25519 (key exchange)
- Encryption: XSalsa20-Poly1305
- Forward secrecy: new ephemeral key per message
- Asymmetric: encrypt for recipient's public key

#### Encryption Decision Tree

```
┌─────────────────────────────────┐
│  What needs encryption?         │
└────────┬────────────────────────┘
         │
    ┌────▼─────────────────────────┐
    │ Legacy credentials?          │
    │ (old installations)          │
    └─Yes─┬───────────────No───────┘
          │                    │
          ▼                    ▼
    ┌──────────────┐    ┌──────────────────┐
    │  SecretBox   │    │ Have public key? │
    │ (symmetric)  │    └──Yes─┬────No─────┘
    └──────────────┘           │     │
                               ▼     ▼
                         ┌─────────┐ ┌──────────┐
                         │   Box   │ │ AES-GCM  │
                         │ (async) │ │  (sym)   │
                         └─────────┘ └──────────┘
```

### 3. Key Management

#### Key Derivation Tree

**Location**: `src/utils/deriveKey.ts`

```typescript
// HMAC-SHA512 hierarchical derivation
I = HMAC-SHA512(usage_string + " Master Seed", seed)
masterKey = I[0:32]
chainCode = I[32:64]

// Child keys
I = HMAC-SHA512(chainCode, 0x00 || index_string)
childKey = I[0:32]
childChainCode = I[32:64]
```

**Usage Example**:
```typescript
const contentKey = await deriveKey(masterSecret, 'Happy EnCoder', ['content']);
```

**Benefits**:
- Single master key → multiple derived keys
- Deterministic (same input → same output)
- One-way (cannot derive master from child)
- Supports hierarchical paths

#### Key Storage

**Location**: `~/.happy/access.key` (or `$HAPPY_HOME_DIR/access.key`)

**Format** (`src/persistence.ts`):

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "encryption": {
    "publicKey": "base64_encoded_32_bytes",
    "machineKey": "base64_encoded_32_bytes"
  }
}
```

**OR** (legacy):

```json
{
  "token": "auth_token...",
  "secret": "base64_encoded_32_bytes"
}
```

**Security Measures**:
- File stored in user home directory (not in version control)
- Atomic writes via `writeFileAtomic()` (`src/utils/writeFileAtomic.ts`)
- Validated with Zod schemas on read
- Cleared on logout

#### Key Backup Format

**Location**: `src/utils/backupKey.ts`

```typescript
// Converts 32-byte secret to Base32 (RFC 4648)
// Format: XXXXX-XXXXX-XXXXX-XXXXX-XXXXX-XXXXX-XXXXX
// Example: AB3D4-5GH7J-9KL2M-4NP6Q-8RST9-VWX2Y-Z4567
```

**Properties**:
- Human-readable (no ambiguous characters)
- Compatible with mobile app import
- 52 characters grouped in 11 blocks
- Checksum validation

### 4. Security Measures

#### Session-Level Encryption

**Location**: `src/api/apiSession.ts`

```typescript
// 1. Generate session-specific key
const sessionKey = randomBytes(32);

// 2. Encrypt all session data
const encryptedMetadata = encrypt(sessionKey, variant, metadata);
const encryptedAgentState = encrypt(sessionKey, variant, agentState);

// 3. Encrypt session key with public key (for sharing)
const encryptedSessionKey = encryptForPublicKey(sessionKey, recipientPubKey);

// 4. Send to server
server.createSession({
  dataEncryptionKey: encryptedSessionKey,
  metadata: encryptedMetadata,
  agentState: encryptedAgentState
});
```

**Message Flow**:
```
User Input (plaintext)
    ↓
[Encrypt with session key]
    ↓
Base64 encode
    ↓
Send to server (encrypted)
    ↓
Server stores (cannot decrypt)
    ↓
Broadcast to devices
    ↓
[Decrypt with session key]
    ↓
Display to user
```

#### RPC Security

**Location**: `src/api/rpc/RpcHandlerManager.ts`

**Method Scoping**:
```typescript
// RPC methods prefixed with session ID
const methodName = `${sessionId}:bash`;

// Server can only call methods for this session
// Prevents cross-session RPC attacks
```

**Parameter Encryption**:
```typescript
// Incoming RPC request
const encryptedParams = request.params; // base64
const params = decrypt(sessionKey, variant, decodeBase64(encryptedParams));

// Execute handler
const result = await handler(params);

// Encrypt response
const encryptedResult = encodeBase64(encrypt(sessionKey, variant, result));
return { result: encryptedResult };
```

#### File System Security

**Lock File Management** (`src/daemon/lock.ts`):
```typescript
// Exclusive daemon instance
const fd = fs.openSync(lockFile, 'wx'); // Fails if exists

// PID verification
const runningPid = fs.readFileSync(lockFile, 'utf-8');
if (!isProcessRunning(runningPid)) {
  fs.unlinkSync(lockFile); // Clean up stale lock
}
```

**Atomic File Operations** (`src/utils/writeFileAtomic.ts`):
```typescript
// Write to temp file + rename (atomic on POSIX)
fs.writeFileSync(tempFile, data);
fs.renameSync(tempFile, targetFile);
```

### 5. Threat Model

| Threat | Protection | Location |
|--------|-----------|----------|
| **Replay attacks** | Random challenge + signature verification | `src/api/auth.ts` |
| **Man-in-the-middle** | End-to-end encryption, ephemeral keys | `src/api/encryption.ts` |
| **Session hijacking** | Session-scoped RPC with ID prefixing | `src/api/rpc/RpcHandlerManager.ts` |
| **Key compromise** | Multiple encryption layers, key rotation | `src/api/encryption.ts` |
| **Offline attacks** | Keys stored in `~/.happy`, not in repo | `src/persistence.ts` |
| **Concurrent updates** | Optimistic concurrency with versioning | `src/api/apiSession.ts` |
| **Daemon impersonation** | Exclusive lock with PID verification | `src/daemon/lock.ts` |
| **File corruption** | Atomic writes via temp file + rename | `src/utils/writeFileAtomic.ts` |

### 6. Cryptographic Libraries

**TweetNaCl.js** (libsodium-like):
- `tweetnacl.sign`: Ed25519 signatures (authentication)
- `tweetnacl.box`: Curve25519 ECDH + XSalsa20-Poly1305 (async encryption)
- `tweetnacl.secretbox`: XSalsa20-Poly1305 (symmetric encryption)

**Node.js Crypto Module**:
- `createCipheriv('aes-256-gcm')`: AES-256-GCM encryption
- `crypto.createHmac('sha512')`: HMAC-SHA512 for key derivation
- `randomBytes()`: Secure random number generation

**Security Audit**:
- ✅ All encryption uses authenticated modes (AEAD)
- ✅ Random nonces (never reused)
- ✅ No hardcoded secrets
- ✅ Constant-time comparisons for signatures
- ✅ Forward secrecy for initial handshake

---

## Session Syncing & Real-time Communication

Happy CLI maintains real-time synchronization between CLI, server, and mobile app using WebSocket connections and encrypted message passing.

### 1. Session Management

#### Session Creation Flow

**Location**: `src/api/api.ts:getOrCreateSession()`

```typescript
// 1. Generate session encryption key
const sessionKey = randomBytes(32);

// 2. Encrypt metadata and agent state
const encryptedMetadata = encrypt(sessionKey, variant, {
  name: 'My Session',
  lastActive: Date.now(),
  workingDirectory: process.cwd()
});

// 3. Encrypt session key for recipient
const dataEncryptionKey = variant === 'dataKey'
  ? encryptForPublicKey(sessionKey, recipientPublicKey)
  : null;

// 4. Create session on server
const session = await api.post('/v1/sessions', {
  id: uuidv4(),
  dataEncryptionKey,
  metadata: base64(encryptedMetadata),
  agentState: base64(encryptedAgentState),
  version: 0
});
```

**Session Properties**:
- Unique ID (UUID v4)
- Versioned state (optimistic concurrency)
- Encrypted metadata (name, last active, etc.)
- Encrypted agent state (Claude session info)
- Session-specific encryption key

#### Session Lifecycle

```
┌──────────┐
│  CREATE  │  Generate ID, encrypt metadata/state
└─────┬────┘
      │
      ▼
┌──────────┐
│  CONNECT │  Establish WebSocket to /v1/updates
└─────┬────┘
      │
      ▼
┌──────────┐
│  SYNC    │  Send/receive encrypted messages
└─────┬────┘
      │
      ├─ Keep-alive heartbeat (every 2s)
      │
      ├─ Handle mode switches (local ↔ remote)
      │
      ▼
┌──────────┐
│  UPDATE  │  Modify metadata/state with versioning
└─────┬────┘
      │
      ▼
┌──────────┐
│  CLOSE   │  Disconnect WebSocket
└──────────┘
```

### 2. WebSocket Session Client

**Location**: `src/api/apiSession.ts:ApiSessionClient`

#### Connection Setup

```typescript
const socket = io(serverUrl, {
  path: '/v1/updates',
  auth: { token: authToken },
  query: { sessionId },
  transports: ['websocket'],
  reconnection: true,
  reconnectionAttempts: Infinity,
  reconnectionDelay: 1000
});
```

#### Event Handlers

| Event | Purpose | Handler |
|-------|---------|---------|
| `connect` | WebSocket connected | Log connection |
| `disconnect` | WebSocket disconnected | Log disconnection |
| `update` | Server broadcast | Decrypt and route message |
| `rpc` | Remote procedure call | Decrypt, execute, respond |

#### Update Message Types

**Location**: `src/api/types.ts:ServerMessage`

1. **new-message**: Claude session message from another device
   ```typescript
   {
     type: 'new-message',
     payload: {
       encryptedMessage: string, // base64(encrypted(ClaudeMessage))
       messageType: 'claude-session' | 'codex' | 'event'
     }
   }
   ```

2. **update-session**: Session metadata changed
   ```typescript
   {
     type: 'update-session',
     payload: {
       encryptedMetadata: string,
       encryptedAgentState: string,
       version: number
     }
   }
   ```

3. **update-machine**: Machine-specific update
   ```typescript
   {
     type: 'update-machine',
     payload: {
       encryptedMetadata: string,
       version: number
     }
   }
   ```

### 3. Message Synchronization

#### Outgoing Flow (CLI → Server)

**Location**: `src/api/apiSession.ts`

```typescript
// Send Claude SDK message
async sendClaudeSessionMessage(message: SDKMessage): Promise<void> {
  const encrypted = encodeBase64(
    encrypt(this.sessionKey, this.variant, message)
  );

  socket.emit('message', {
    sessionId: this.sessionId,
    messageType: 'claude-session',
    encryptedMessage: encrypted
  });
}

// Send session event (mode switch, ready signal, etc.)
async sendSessionEvent(event: SessionEvent): Promise<void> {
  const encrypted = encodeBase64(
    encrypt(this.sessionKey, this.variant, event)
  );

  socket.emit('message', {
    sessionId: this.sessionId,
    messageType: 'event',
    encryptedMessage: encrypted
  });
}
```

#### Incoming Flow (Server → CLI)

```typescript
socket.on('update', async (data: ServerMessage) => {
  switch (data.type) {
    case 'new-message':
      const message = decrypt(
        this.sessionKey,
        this.variant,
        decodeBase64(data.payload.encryptedMessage)
      );

      // Route to appropriate handler
      if (data.payload.messageType === 'claude-session') {
        this.onClaudeMessage?.(message);
      } else if (data.payload.messageType === 'event') {
        this.onSessionEvent?.(message);
      }
      break;

    case 'update-session':
      const metadata = decrypt(...data.payload.encryptedMetadata);
      const agentState = decrypt(...data.payload.encryptedAgentState);
      this.localMetadata = metadata;
      this.localAgentState = agentState;
      this.localVersion = data.payload.version;
      break;
  }
});
```

#### Message Queue (Batching & Ordering)

**Location**: `src/utils/MessageQueue2.ts`

```typescript
class MessageQueue2 {
  private queue: QueuedMessage[] = [];
  private processing = false;
  private currentModeHash: string | null = null;

  async enqueue(message: Message, mode: Mode): Promise<void> {
    const modeHash = hash(mode);

    // Mode changed? Isolate previous batch
    if (this.currentModeHash !== null && this.currentModeHash !== modeHash) {
      this.isolateCurrentBatch();
    }

    this.currentModeHash = modeHash;
    this.queue.push({ message, mode, isolated: false });

    if (!this.processing) {
      this.processBatch();
    }
  }

  private async processBatch(): Promise<void> {
    this.processing = true;

    while (this.queue.length > 0) {
      // Find isolated message or batch of same mode
      const batch = this.extractNextBatch();

      // Send batch to API
      await sendBatch(batch);
    }

    this.processing = false;
  }
}
```

**Properties**:
- Maintains strict ordering
- Groups messages by mode (prevents context mixing)
- Isolates mode changes
- Handles `/clear` and `/compact` commands specially

### 4. RPC System (Remote Procedure Calls)

**Location**: `src/api/rpc/RpcHandlerManager.ts`

#### Handler Registration

```typescript
class RpcHandlerManager {
  private handlers = new Map<string, RpcHandler>();

  register(method: string, handler: RpcHandler): void {
    const scopedMethod = `${this.sessionId}:${method}`;
    this.handlers.set(scopedMethod, handler);

    // Tell server about this handler
    this.socket.emit('rpc:register', { method: scopedMethod });
  }
}
```

#### RPC Request Flow

```
Mobile App                 Server                   CLI
    │                         │                      │
    │ 1. Call RPC method      │                      │
    ├────────────────────────>│                      │
    │   { method, params }    │                      │
    │                         │                      │
    │                         │ 2. Forward to CLI    │
    │                         ├─────────────────────>│
    │                         │   'rpc' event        │
    │                         │                      │
    │                         │ 3. Decrypt params    │
    │                         │      Execute handler │
    │                         │      Encrypt result  │
    │                         │<─────────────────────┤
    │                         │   { result }         │
    │                         │                      │
    │ 4. Return result        │                      │
    │<────────────────────────┤                      │
    │                         │                      │
```

#### Registered RPC Methods

**Location**: `src/modules/common/registerCommonHandlers.ts`

| Method | Purpose | Parameters | Returns |
|--------|---------|------------|---------|
| `bash` | Execute shell command | `{ command, timeout? }` | `{ stdout, stderr, exitCode }` |
| `readFile` | Read file contents | `{ path }` | `{ content: base64 }` |
| `writeFile` | Write file (hash-verified) | `{ path, content, expectedHash? }` | `{ hash }` |
| `listDirectory` | List directory | `{ path, sortBy? }` | `{ entries: FileEntry[] }` |
| `getDirectoryTree` | Recursive tree | `{ path, maxDepth? }` | `{ tree: TreeNode }` |
| `ripgrep` | Search code | `{ pattern, path?, flags? }` | `{ matches: Match[] }` |
| `difftastic` | Show diff | `{ file1, file2 }` | `{ diff: string }` |
| `abort` | Stop current task | `{}` | `{ success: true }` |
| `switch` | Switch to local mode | `{}` | `{ success: true }` |

#### RPC Encryption

```typescript
// Incoming request
socket.on('rpc', async (request: RpcRequest) => {
  const { id, method, params: encryptedParams } = request;

  // Decrypt parameters
  const params = decrypt(
    sessionKey,
    variant,
    decodeBase64(encryptedParams)
  );

  // Execute handler
  const result = await handlers.get(method)?.(params);

  // Encrypt response
  const encryptedResult = encodeBase64(
    encrypt(sessionKey, variant, result)
  );

  // Send back to server
  socket.emit('rpc:response', {
    id,
    result: encryptedResult
  });
});
```

### 5. Optimistic Concurrency Control

**Location**: `src/api/apiSession.ts:updateMetadata()`

#### Version-Based Updates

```typescript
async updateMetadata(updates: Partial<Metadata>): Promise<void> {
  // Merge with local state
  const newMetadata = { ...this.localMetadata, ...updates };

  // Encrypt
  const encrypted = encodeBase64(
    encrypt(this.sessionKey, this.variant, newMetadata)
  );

  // Send with expected version
  const response = await this.api.patch('/v1/sessions/:id', {
    encryptedMetadata: encrypted,
    expectedVersion: this.localVersion
  });

  if (response.status === 409) {
    // Version conflict - retry with exponential backoff
    await sleep(Math.pow(2, retryCount) * 100);
    return this.updateMetadata(updates);
  }

  // Success - update local version
  this.localVersion = response.data.version;
  this.localMetadata = newMetadata;
}
```

#### Conflict Resolution

```
Local State (v3)         Server State (v4)
      │                         │
      │ 1. Update metadata      │
      ├────────────────────────>│
      │    expectedVersion: 3   │
      │                         │
      │ 2. Version mismatch!    │
      │<────────────────────────┤
      │    409 Conflict         │
      │                         │
      │ 3. Fetch latest         │
      ├────────────────────────>│
      │<────────────────────────┤
      │    v4 metadata          │
      │                         │
      │ 4. Retry update         │
      ├────────────────────────>│
      │    expectedVersion: 4   │
      │                         │
      │ 5. Success              │
      │<────────────────────────┤
      │    200 OK, v5           │
      │                         │
```

**Properties**:
- Prevents lost updates
- Automatic retry with backoff
- No manual conflict resolution needed
- Works across multiple devices

### 6. Keep-Alive Mechanism

**Location**: `src/claude/session.ts:keepAlive()`

```typescript
setInterval(() => {
  socket.volatile.emit('keep-alive', {
    sessionId: this.sessionId,
    mode: this.mode, // 'local' or 'remote'
    thinking: this.thinking, // Claude is processing
    claudeSessionId: this.claudeSessionId
  });
}, 2000); // Every 2 seconds
```

**Volatile Emit**:
- Message not queued if disconnected
- Reduces backlog on reconnection
- Server knows client is alive

**Thinking State**:
- Tracked via `fetch-start`/`fetch-end` events (local mode)
- Inferred from message types (remote mode)
- Displayed in mobile UI

---

## Claude Code CLI Integration

Happy CLI integrates with Claude Code using two approaches: **PTY-based** (interactive) and **SDK-based** (remote).

### 1. Integration Modes

#### Mode Comparison

| Aspect | Local (PTY) | Remote (SDK) |
|--------|-------------|--------------|
| **Process Type** | Full terminal with stdin/stdout | Headless with pipes |
| **Invocation** | `spawn()` with inherited stdio | `spawn()` via SDK |
| **Input Method** | User types in terminal | AsyncIterable pushed to stdin |
| **Output Method** | Terminal display | JSON lines on stdout |
| **Session Detection** | File watcher + UUID interception | System 'init' message |
| **Message Parsing** | Read `.jsonl` files | Parse stdout directly |
| **Tool Permissions** | Claude UI handles | Intercepted via control messages |
| **Thinking State** | fd 3 lifecycle events | Inferred from message types |
| **Use Case** | Developer at terminal | Mobile app control |

#### Mode Decision Tree

```
User Request
    │
    ├─ User at terminal?
    │  └─ YES → Local Mode (PTY)
    │
    └─ Remote control request?
       └─ YES → Remote Mode (SDK)
```

### 2. Local Mode (Interactive PTY)

**Location**: `src/claude/claudeLocal.ts`

#### Process Spawning

```typescript
const claudeProcess = spawn('node', [
  '/path/to/scripts/claude_local_launcher.cjs',
  '--verbose',
  ...(sessionId ? ['--resume', sessionId] : []),
  ...(appendSystemPrompt ? ['--append-system-prompt', appendSystemPrompt] : []),
  ...(mcpConfig ? ['--mcp-config', JSON.stringify(mcpConfig)] : []),
  ...customArgs
], {
  stdio: ['inherit', 'inherit', 'inherit', 'pipe'], // fd 3 = custom pipe
  cwd: workingDirectory,
  env: {
    ...process.env,
    NODE_OPTIONS: '--no-warnings',
    ANTHROPIC_API_KEY: apiKey
  }
});
```

**Key Features**:
- `stdio[0-2]` inherited → user can interact directly
- `stdio[3]` = pipe → receives lifecycle events
- No message parsing needed (terminal shows output)

#### Session Detection

**Problem**: How to get the session ID when Claude spawns?

**Solution**: Intercept UUID generation on file descriptor 3

**Launcher Hook** (`scripts/claude_local_launcher.cjs`):

```javascript
// Hook into crypto.randomUUID()
const originalRandomUUID = crypto.randomUUID;
crypto.randomUUID = function() {
  const uuid = originalRandomUUID();

  // Send to parent process on fd 3
  if (process.stdout.fd !== undefined && fd3Stream) {
    fd3Stream.write(JSON.stringify({
      type: 'uuid-generated',
      uuid
    }) + '\n');
  }

  return uuid;
};
```

**Detection Flow**:

```
1. Claude CLI starts
    ↓
2. Generates session ID via randomUUID()
    ↓
3. Hook intercepts and sends UUID on fd 3
    ↓
4. Parent watches project directory for .jsonl file
    ↓
5. Matches UUID against filesystem
    ↓
6. Resolves actual session ID
    ↓
7. Returns to caller
```

**Code** (`src/claude/claudeLocal.ts:54-82`):

```typescript
// Listen to fd 3 for UUIDs
claudeProcess.stdio[3].on('data', (chunk) => {
  const line = chunk.toString();
  const event = JSON.parse(line);

  if (event.type === 'uuid-generated') {
    possibleUuids.push(event.uuid);
  }
});

// Watch project directory
const watcher = fs.watch(projectDir, (eventType, filename) => {
  if (filename.endsWith('.jsonl')) {
    const sessionId = filename.replace('.jsonl', '');

    // Check if this UUID was generated by our process
    if (possibleUuids.includes(sessionId)) {
      detectedSessionId = sessionId;
      watcher.close();
    }
  }
});
```

#### Thinking State Detection

**Problem**: How to know when Claude is thinking (fetching from API)?

**Solution**: Hook `fetch()` and emit lifecycle events

**Launcher Hook** (`scripts/claude_local_launcher.cjs`):

```javascript
const originalFetch = global.fetch;
global.fetch = async function(...args) {
  // Notify fetch started
  fd3Stream?.write(JSON.stringify({
    type: 'fetch-start',
    url: args[0]
  }) + '\n');

  try {
    const response = await originalFetch(...args);

    // Notify fetch ended
    fd3Stream?.write(JSON.stringify({
      type: 'fetch-end',
      status: response.status
    }) + '\n');

    return response;
  } catch (err) {
    fd3Stream?.write(JSON.stringify({
      type: 'fetch-error',
      error: err.message
    }) + '\n');
    throw err;
  }
};
```

**Usage** (`src/claude/session.ts`):

```typescript
claudeProcess.stdio[3].on('data', (chunk) => {
  const event = JSON.parse(chunk.toString());

  if (event.type === 'fetch-start') {
    session.thinking = true;
    socket.volatile.emit('keep-alive', { thinking: true });
  } else if (event.type === 'fetch-end' || event.type === 'fetch-error') {
    session.thinking = false;
    socket.volatile.emit('keep-alive', { thinking: false });
  }
});
```

### 3. Remote Mode (SDK-based)

**Location**: `src/claude/claudeRemote.ts`

#### SDK Query Invocation

```typescript
import { query } from './sdk/query';

const response = query({
  prompt: messageIterator, // AsyncIterable<SDKUserMessage>
  options: {
    cwd: workingDirectory,
    resume: sessionId,
    model: 'claude-3-5-sonnet-20241022',
    fallbackModel: 'claude-3-5-haiku-20250110',
    appendSystemPrompt: 'You are a helpful assistant...',
    mcpServers: {
      'happy': {
        type: 'http',
        url: 'http://localhost:3001'
      }
    },
    permissionMode: 'default',
    allowedTools: [...],
    canCallTool: async (toolName, input, { signal }) => {
      // Permission handler
      return { behavior: 'allow', updatedInput: input };
    },
    executable: 'node',
    pathToClaudeCodeExecutable: '/path/to/scripts/claude_remote_launcher.cjs',
    abort: abortController.signal
  }
});
```

#### Message Iteration

```typescript
for await (const message of response) {
  switch (message.type) {
    case 'system':
      // Session initialized
      sessionId = message.session_id;
      model = message.model;
      tools = message.tools;
      break;

    case 'assistant':
      // Claude response (thinking, tool use, text)
      for (const block of message.message.content) {
        if (block.type === 'tool_use') {
          // Tool call detected
        } else if (block.type === 'thinking') {
          // Thinking block
        } else if (block.type === 'text') {
          // Text response
        }
      }
      break;

    case 'user':
      // Tool results
      break;

    case 'result':
      // Conversation ended
      break;
  }
}
```

#### Permission Handling

**Flow**:

```
Claude wants to use tool
    ↓
SDK emits control_request
    ↓
canCallTool callback invoked
    ↓
Ask mobile app for approval
    ↓
Return { behavior: 'allow' } or { behavior: 'deny' }
    ↓
SDK responds with control_response
    ↓
Claude executes tool or shows denial
```

**Implementation** (`src/claude/utils/permissionHandler.ts`):

```typescript
class PermissionHandler {
  async askPermission(
    toolName: string,
    input: unknown
  ): Promise<PermissionResult> {
    const toolCallId = generateId();

    // Send permission request to mobile app
    await sessionClient.sendSessionEvent({
      type: 'permission-request',
      toolCallId,
      toolName,
      input
    });

    // Wait for mobile app response
    const approval = await this.waitForResponse(toolCallId);

    return approval
      ? { behavior: 'allow', updatedInput: input }
      : { behavior: 'deny', message: 'User denied permission' };
  }
}
```

#### Outgoing Message Queue

**Location**: `src/claude/utils/OutgoingMessageQueue.ts`

**Problem**: Messages must arrive in order, but some are delayed (permissions)

**Solution**: Incremental ID tracking with early release

```typescript
class OutgoingMessageQueue {
  private queue: QueuedMessage[] = [];
  private nextExpectedId = 0;

  async enqueue(message: Message, id: number, delay?: boolean): Promise<void> {
    this.queue.push({ message, id, delay });
    this.queue.sort((a, b) => a.id - b.id);

    if (!delay) {
      this.releaseReady();
    }
  }

  private releaseReady(): void {
    while (this.queue.length > 0 && this.queue[0].id === this.nextExpectedId && !this.queue[0].delay) {
      const { message } = this.queue.shift()!;
      this.send(message);
      this.nextExpectedId++;
    }
  }

  earlyRelease(toolCallId: string): void {
    // Find delayed message and release it
    const index = this.queue.findIndex(m => m.toolCallId === toolCallId);
    if (index !== -1) {
      this.queue[index].delay = false;
      this.releaseReady();
    }
  }
}
```

**Example**:

```
Messages generated:
  0: User message "help me"
  1: System message (init)
  2: Assistant message (tool call - DELAYED)
  3: User message (tool result)

Queue state:
  [0, 1, 2*, 3]  (* = delayed)

Release 0, 1
  nextExpected = 2, but 2 is delayed

Permission approved
  earlyRelease(toolCallId)

Release 2, 3
  Done
```

### 4. Claude CLI Flags & Parameters

#### Common Flags

| Flag | Purpose | Value | When Used |
|------|---------|-------|-----------|
| `--output-format` | Output format | `stream-json` | Always (remote) |
| `--verbose` | Debug logging | - | Always |
| `--resume` | Resume session | `<sessionId>` | If resuming |
| `--append-system-prompt` | Add to prompt | `<text>` | Always (Happy prompt) |
| `--mcp-config` | MCP servers | `<json>` | If MCP enabled |
| `--allowedTools` | Tool whitelist | `comma,separated` | If specified |
| `--permission-mode` | Permission level | `default\|plan` | Always |
| `--model` | Override model | `<model-id>` | If specified |
| `--fallback-model` | Fallback | `<model-id>` | If specified |
| `--permission-prompt-tool` | Permission channel | `stdio` | Remote mode |
| `--input-format` | Input format | `stream-json` | Remote mode |
| `--max-turns` | Turn limit | `<number>` | If specified |
| `--system-prompt` | Replace prompt | `<text>` | If specified |

#### Flag Construction

**Location**: `src/claude/sdk/query.ts:constructArgs()`

```typescript
const args = [
  '--output-format', 'stream-json',
  '--verbose'
];

if (options.resume) {
  args.push('--resume', options.resume);
}

if (options.appendSystemPrompt) {
  args.push('--append-system-prompt', options.appendSystemPrompt);
}

if (options.mcpServers) {
  args.push('--mcp-config', JSON.stringify(options.mcpServers));
}

// ... etc
```

### 5. Session Resumption

#### Resumption Flow

```
┌─────────────────────────────────────┐
│ Happy CLI receives resume request   │
│ sessionId = 'aada10c6-...'          │
└──────────┬──────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│ Check if session file exists        │
│ claudeCheckSession(sessionId, path) │
└──────────┬──────────────────────────┘
           │
      ┌────┴────┐
      │ Exists? │
      └────┬────┘
           │
    ┌──────┴──────┐
    │             │
   YES           NO
    │             │
    ▼             ▼
┌────────┐   ┌────────┐
│ Resume │   │ Start  │
│  with  │   │  new   │
│  flag  │   │session │
└───┬────┘   └────────┘
    │
    ▼
┌──────────────────────────────────────┐
│ Claude creates NEW session file      │
│ New ID: '1433467f-...'               │
│ Contains FULL HISTORY from original  │
│ All sessionIds REWRITTEN to new ID   │
└──────────────────────────────────────┘
```

#### Session File Behavior

**Original Session** (`aada10c6-9299-4c45-abc4-91db9c0f935d.jsonl`):
```jsonl
{"parentUuid":null,"sessionId":"aada10c6-...","message":{"role":"user","content":"list files"},...}
{"parentUuid":"...","sessionId":"aada10c6-...","message":{"role":"assistant","content":[...]},...}
```

**Resumed Session** (`1433467f-ff14-4292-b5b2-2aac77a808f0.jsonl`):
```jsonl
{"type":"summary","summary":"Listing directory files","leafUuid":"..."}
{"parentUuid":null,"sessionId":"1433467f-...","message":{"role":"user","content":"list files"},...}
{"parentUuid":"...","sessionId":"1433467f-...","message":{"role":"assistant","content":[...]},...}
{"parentUuid":"...","sessionId":"1433467f-...","message":{"role":"user","content":"what did we see?"},...}
```

**Key Observations**:
- ✅ Original file untouched
- ✅ New file created with new ID
- ✅ Complete history copied
- ✅ **All sessionIds rewritten** to new ID
- ✅ Summary line prepended
- ✅ Claude maintains full context

#### One-Time Flag Cleanup

**Location**: `src/claude/session.ts:consumeOneTimeFlags()`

```typescript
consumeOneTimeFlags(): void {
  // Remove --resume flag after first use
  this.claudeArgs = this.claudeArgs.filter((arg, i) =>
    !(arg === '--resume' && this.claudeArgs[i-1] === '--resume')
  );
}
```

**Why?** Prevents re-resuming in subsequent spawns (e.g., mode switches)

### 6. Mode Switching

**Location**: `src/claude/loop.ts`

#### Main Control Loop

```typescript
async function loop(session: Session): Promise<void> {
  while (true) {
    const reason = session.mode === 'local'
      ? await claudeLocalLauncher(session)
      : await claudeRemoteLauncher(session);

    if (reason === 'exit') {
      break; // User requested exit
    } else if (reason === 'switch') {
      // Switch mode and continue
      session.mode = session.mode === 'local' ? 'remote' : 'local';
      continue;
    }
  }
}
```

#### Switch Triggers

**Local → Remote**:
- Message received from mobile app
- RPC call indicates remote control requested
- Returns `'switch'` from `claudeLocalLauncher`

**Remote → Local**:
- User presses space-space (double space) in mobile app
- RPC `switch` method called
- Returns `'switch'` from `claudeRemoteLauncher`

#### Switch Flow

```
┌──────────┐
│  LOCAL   │  User at terminal
└─────┬────┘
      │
      │ [Mobile app sends message]
      │
      ▼
┌──────────┐
│ CLEANUP  │  Kill Claude process
└─────┬────┘  Consume one-time flags
      │
      ▼
┌──────────┐
│  REMOTE  │  Mobile app control
└─────┬────┘
      │
      │ [User presses space-space]
      │
      ▼
┌──────────┐
│ CLEANUP  │  Abort SDK query
└─────┬────┘  Consume one-time flags
      │
      ▼
┌──────────┐
│  LOCAL   │  Back to terminal
└──────────┘
```

#### Message Queue Isolation

**Location**: `src/utils/MessageQueue2.ts`

```typescript
// When mode changes, isolate previous batch
if (this.currentModeHash !== modeHash) {
  this.isolateCurrentBatch(); // Prevents mixing contexts
}
```

**Why?** Ensures messages from one mode don't leak into another

---

## Data Flow Diagrams

### Complete System Flow

```
┌─────────────┐
│ Mobile App  │
└──────┬──────┘
       │
       │ 1. User types message
       │
       ▼
┌──────────────────────┐
│ Encrypt with session │
│ key (AES-256-GCM)    │
└──────┬───────────────┘
       │
       │ 2. Send via WebSocket
       │
       ▼
┌──────────────────────┐
│ Server broadcasts    │
│ (cannot decrypt)     │
└──────┬───────────────┘
       │
       │ 3. Receive on CLI
       │
       ▼
┌──────────────────────┐
│ Decrypt with session │
│ key                  │
└──────┬───────────────┘
       │
       │ 4. Add to MessageQueue
       │
       ▼
┌──────────────────────┐
│ Push to SDK input    │
│ (AsyncIterable)      │
└──────┬───────────────┘
       │
       │ 5. Claude processes
       │
       ▼
┌──────────────────────┐
│ Claude generates     │
│ response/tool calls  │
└──────┬───────────────┘
       │
       │ 6. Parse JSON output
       │
       ▼
┌──────────────────────┐
│ Permission check?    │
└──────┬───────────────┘
       │
   ┌───┴───┐
   │ Tool? │
   └───┬───┘
       │
    ┌──┴──┐
   YES   NO
    │     │
    ▼     ▼
┌────────┐ ┌─────────┐
│  Ask   │ │  Send   │
│ mobile │ │ direct  │
└───┬────┘ └────┬────┘
    │           │
    │ Wait      │
    │           │
    ▼           │
┌────────┐      │
│Approve?│      │
└───┬────┘      │
    │           │
    ▼           │
┌────────┐      │
│Execute │      │
└───┬────┘      │
    │           │
    └─────┬─────┘
          │
          │ 7. Encrypt result
          │
          ▼
    ┌──────────┐
    │  Server  │
    └─────┬────┘
          │
          │ 8. Broadcast
          │
          ▼
    ┌──────────┐
    │ Mobile   │
    │ displays │
    └──────────┘
```

### Authentication Flow

```
┌──────┐                               ┌──────┐
│ CLI  │                               │Server│
└───┬──┘                               └───┬──┘
    │                                      │
    │ 1. Generate ephemeral keypair        │
    │    secret = randomBytes(32)          │
    │    keypair = box.keyPair(secret)     │
    │                                      │
    │ 2. POST /v1/auth/request             │
    │    { publicKey, method: 'mobile' }   │
    ├─────────────────────────────────────>│
    │                                      │
    │                                      │ 3. Server encrypts credentials
    │                                      │    box(credentials, ephemeralPubKey)
    │                                      │
    │ 4. Encrypted response                │
    │<─────────────────────────────────────┤
    │    ephemPubKey + nonce + encrypted   │
    │                                      │
    │ 5. Decrypt with box.open()           │
    │    Extract: publicKey + machineKey   │
    │                                      │
    │ 6. Generate challenge                │
    │    challenge = randomBytes(32)       │
    │    signature = sign(challenge, key)  │
    │                                      │
    │ 7. POST /v1/auth/token               │
    │    { challenge, publicKey, signature }│
    ├─────────────────────────────────────>│
    │                                      │
    │                                      │ 8. Verify signature
    │                                      │    verify(signature, challenge, pubKey)
    │                                      │
    │ 9. Auth token                        │
    │<─────────────────────────────────────┤
    │    { token: "..." }                  │
    │                                      │
    │ 10. Save to ~/.happy/access.key      │
    │                                      │
```

### Permission Request Flow

```
┌────────┐           ┌─────┐          ┌────────┐           ┌────────┐
│ Claude │           │ CLI │          │ Server │           │ Mobile │
└───┬────┘           └──┬──┘          └───┬────┘           └───┬────┘
    │                   │                 │                    │
    │ 1. Tool use       │                 │                    │
    ├──────────────────>│                 │                    │
    │ control_request   │                 │                    │
    │                   │                 │                    │
    │                   │ 2. canCallTool  │                    │
    │                   │    callback     │                    │
    │                   │                 │                    │
    │                   │ 3. Permission   │                    │
    │                   │    event        │                    │
    │                   ├────────────────>│                    │
    │                   │ (encrypted)     │                    │
    │                   │                 │                    │
    │                   │                 │ 4. Broadcast       │
    │                   │                 ├───────────────────>│
    │                   │                 │                    │
    │                   │                 │                    │ 5. User
    │                   │                 │                    │    approves
    │                   │                 │                    │
    │                   │                 │ 6. Approval        │
    │                   │                 │<───────────────────┤
    │                   │                 │ (encrypted)        │
    │                   │                 │                    │
    │                   │ 7. Approval     │                    │
    │                   │<────────────────┤                    │
    │                   │                 │                    │
    │                   │ 8. Return       │                    │
    │                   │    { allow }    │                    │
    │                   │                 │                    │
    │ 9. control_resp   │                 │                    │
    │<──────────────────┤                 │                    │
    │ (allowed)         │                 │                    │
    │                   │                 │                    │
    │ 10. Execute tool  │                 │                    │
    │                   │                 │                    │
```

---

## Key Takeaways

### Security
- **End-to-end encryption**: Server never sees plaintext
- **Challenge-response**: Prevents replay attacks
- **Multiple encryption layers**: Defense in depth
- **Ephemeral keys**: Forward secrecy for auth
- **Session-scoped RPC**: Prevents cross-session attacks

### Syncing
- **Real-time WebSocket**: Bidirectional communication
- **Optimistic concurrency**: Automatic conflict resolution
- **Message queuing**: Maintains strict ordering
- **Keep-alive heartbeat**: Connection health monitoring

### Claude Integration
- **Dual-mode**: Interactive (PTY) and remote (SDK)
- **Session resumption**: Full context preservation
- **Permission system**: Mobile app approval for tools
- **Mode switching**: Seamless local ↔ remote transitions
- **Message ordering**: Guaranteed delivery order

### Architecture Strengths
- ✅ **Zero-trust encryption**: Everything encrypted before transmission
- ✅ **Stateless server**: Server is just a relay (cannot decrypt)
- ✅ **Robust concurrency**: Version-based updates prevent conflicts
- ✅ **Clean separation**: Local/remote modes isolated
- ✅ **Extensible RPC**: Easy to add new remote capabilities

---

## File Reference

| Component | Primary Files |
|-----------|--------------|
| **Authentication** | `src/api/auth.ts`, `src/ui/auth.ts` |
| **Encryption** | `src/api/encryption.ts`, `src/utils/deriveKey.ts` |
| **Session Management** | `src/api/apiSession.ts`, `src/api/api.ts` |
| **RPC System** | `src/api/rpc/RpcHandlerManager.ts`, `src/modules/common/registerCommonHandlers.ts` |
| **Message Queue** | `src/utils/MessageQueue2.ts`, `src/claude/utils/OutgoingMessageQueue.ts` |
| **Claude Local** | `src/claude/claudeLocal.ts`, `scripts/claude_local_launcher.cjs` |
| **Claude Remote** | `src/claude/claudeRemote.ts`, `src/claude/sdk/query.ts` |
| **Control Loop** | `src/claude/loop.ts`, `src/claude/session.ts` |
| **Permissions** | `src/claude/utils/permissionHandler.ts` |
| **Persistence** | `src/persistence.ts`, `src/utils/writeFileAtomic.ts` |

---

**Last Updated**: 2025-11-14
**Architecture Version**: 0.11.2
