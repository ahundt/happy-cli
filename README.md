# Happy

Code on the go controlling claude code from your mobile device.

Free. Open source. Code anywhere.

## Documentation

Comprehensive technical documentation is available in the [`/docs`](./docs) directory:

- **[Architecture Overview](./docs/ARCHITECTURE.md)** - Complete system architecture covering:
  - Security & authentication (challenge-response, end-to-end encryption)
  - Session syncing & real-time communication (WebSocket, RPC)
  - Claude Code CLI integration (local & remote modes)

- **[Claude Code Integration](./docs/CLAUDE-CODE-INTEGRATION.md)** - Deep dive into Claude Code CLI integration:
  - How happy-cli wraps and controls Claude Code
  - Code injection mechanisms (launcher scripts & hooks)
  - All 7 interaction points between happy-cli and Claude Code
  - Message formats (JSONL & stream-json)

- **[Mode Switching](./docs/MODE-SWITCHING.md)** - Explains mode switching behavior:
  - How processes are killed and restarted during mode switches
  - Local (PTY) ↔ Remote (SDK) transitions
  - Session continuity via `--resume` flag
  - Performance implications

## Installation

```bash
npm install -g happy-coder
```

## Usage

```bash
happy
```

This will:
1. Start a Claude Code session
2. Display a QR code to connect from your mobile device
3. Allow real-time session sharing between Claude Code and your mobile app

## Commands

- `happy auth` – Manage authentication
- `happy codex` – Start Codex mode
- `happy connect` – Store AI vendor API keys in Happy cloud
- `happy notify` – Send a push notification to your devices
- `happy daemon` – Manage background service
- `happy doctor` – System diagnostics & troubleshooting

## Options

- `-h, --help` - Show help
- `-v, --version` - Show version
- `-m, --model <model>` - Claude model to use (default: sonnet)
- `-p, --permission-mode <mode>` - Permission mode: auto, default, or plan
- `--claude-env KEY=VALUE` - Set environment variable for Claude Code
- `--claude-arg ARG` - Pass additional argument to Claude CLI

## Environment Variables

- `HAPPY_SERVER_URL` - Custom server URL (default: https://api.cluster-fluster.com)
- `HAPPY_WEBAPP_URL` - Custom web app URL (default: https://app.happy.engineering)
- `HAPPY_HOME_DIR` - Custom home directory for Happy data (default: ~/.happy)
- `HAPPY_DISABLE_CAFFEINATE` - Disable macOS sleep prevention (set to `true`, `1`, or `yes`)
- `HAPPY_EXPERIMENTAL` - Enable experimental features (set to `true`, `1`, or `yes`)

## Requirements

- Node.js >= 20.0.0
  - Required by `eventsource-parser@3.0.5`, which is required by
  `@modelcontextprotocol/sdk`, which we used to implement permission forwarding
  to mobile app
- Claude CLI installed & logged in (`claude` command available in PATH)

## License

MIT
