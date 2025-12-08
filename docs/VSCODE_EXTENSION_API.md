# Claude Code VS Code Extension API Documentation

This document describes the transport mechanisms and APIs used by the Claude Code VS Code extension to communicate with the Claude Code CLI. This information is useful for implementing custom IDE integrations.

## Table of Contents

1. [Overview](#overview)
2. [Architecture Options](#architecture-options)
3. [IDE Integration Protocol (WebSocket/MCP)](#ide-integration-protocol-websocketmcp)
4. [Agent Client Protocol (ACP)](#agent-client-protocol-acp)
5. [SDK Headless Mode](#sdk-headless-mode)
6. [Message Type Reference](#message-type-reference)
7. [Security Considerations](#security-considerations)
8. [Implementation Examples](#implementation-examples)

---

## Overview

Claude Code provides three main ways to interface with it programmatically:

| Method | Transport | Use Case |
|--------|-----------|----------|
| **IDE Integration Protocol** | WebSocket + MCP | Native IDE extensions (VS Code, Neovim, etc.) |
| **Agent Client Protocol (ACP)** | JSON-RPC over stdio | Cross-editor agent standardization (Zed, etc.) |
| **SDK Headless Mode** | stdin/stdout + JSONL | Programmatic automation, CI/CD, custom apps |

---

## Architecture Options

### Option 1: IDE Integration Protocol (What VS Code Uses)

The VS Code extension implements a **WebSocket server** running on localhost that exposes **MCP (Model Context Protocol) tools**. The Claude Code CLI discovers and connects to this server.

```
┌─────────────────┐     WebSocket     ┌─────────────────┐
│  VS Code        │◄─────────────────►│  Claude Code    │
│  Extension      │   JSON-RPC 2.0    │  CLI            │
│  (MCP Server)   │                   │  (MCP Client)   │
└─────────────────┘                   └─────────────────┘
        │
        ▼
  Lock File Discovery
  (~/.claude/ide/[port].lock)
```

### Option 2: Agent Client Protocol (ACP)

ACP is an open standard for connecting AI agents to code editors, similar to LSP for language servers.

```
┌─────────────────┐    stdin/stdout    ┌─────────────────┐
│  Code Editor    │◄──────────────────►│  Claude Code    │
│  (ACP Client)   │   JSON-RPC 2.0     │  ACP Adapter    │
└─────────────────┘                    └─────────────────┘
```

### Option 3: SDK Headless Mode

Direct programmatic access via the Claude Code CLI or SDK libraries.

```
┌─────────────────┐    stdin/stdout    ┌─────────────────┐
│  Your App       │◄──────────────────►│  Claude Code    │
│  (SDK Client)   │      JSONL         │  CLI (-p mode)  │
└─────────────────┘                    └─────────────────┘
```

---

## IDE Integration Protocol (WebSocket/MCP)

This is the native protocol used by the VS Code extension.

### Protocol Stack

- **Transport**: WebSocket (RFC 6455) over TCP on localhost
- **Message Format**: JSON-RPC 2.0
- **Application Protocol**: MCP (Model Context Protocol) Specification 2025-03-26

### Discovery Mechanism

The IDE extension creates a lock file that the CLI uses for discovery:

**Lock File Location**: `~/.claude/ide/[port].lock`
(Or `$CLAUDE_CONFIG_DIR/ide/[port].lock` if `CLAUDE_CONFIG_DIR` is set)

**Lock File Structure**:
```json
{
  "pid": 12345,
  "workspaceFolders": ["/path/to/project"],
  "ideName": "VS Code",
  "transport": "ws",
  "authToken": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Environment Variables** (set by the IDE):
- `CLAUDE_CODE_SSE_PORT`: WebSocket server port (10000-65535 range)
- `ENABLE_IDE_INTEGRATION`: Set to `"true"`

### Authentication

Authentication uses a custom WebSocket header during the handshake:

```
x-claude-code-ide-authorization: [authToken from lock file]
```

The IDE validates this header against the stored token; mismatches result in connection rejection.

### Message Format

All messages use JSON-RPC 2.0:

```json
{
  "jsonrpc": "2.0",
  "method": "method_name",
  "params": {},
  "id": "unique-id"
}
```

### IDE → Claude Notifications

#### `selection_changed`
Notifies Claude when the user's text selection changes.

```json
{
  "method": "selection_changed",
  "params": {
    "text": "selected content",
    "filePath": "/absolute/path/to/file.js",
    "fileUrl": "file:///absolute/path/to/file.js",
    "selection": {
      "start": {"line": 10, "character": 5},
      "end": {"line": 15, "character": 20},
      "isEmpty": false
    }
  }
}
```

#### `at_mentioned`
Notifies Claude when the user @-mentions a file.

```json
{
  "method": "at_mentioned",
  "params": {
    "filePath": "/path/to/file",
    "lineStart": 10,
    "lineEnd": 20
  }
}
```

### MCP Tools (Claude → IDE Requests)

All tool responses follow the MCP content format:

```json
{
  "content": [
    {"type": "text", "text": "..."},
    {"type": "image", "data": "base64...", "mimeType": "image/png"}
  ]
}
```

#### `openFile`
Opens a file in the editor with optional text selection.

**Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `filePath` | string | Yes | Absolute path to the file |
| `preview` | boolean | No | Open in preview mode |
| `startText` | string | No | Text pattern for selection start |
| `endText` | string | No | Text pattern for selection end |
| `selectToEndOfLine` | boolean | No | Extend selection to line end |
| `makeFrontmost` | boolean | No | Bring editor to focus |

**Response** (with `makeFrontmost=true`):
```json
{"content": [{"type": "text", "text": "Opened file: /path/to/file.js"}]}
```

#### `openDiff`
Opens a git diff view (blocking operation awaiting user action).

**Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `old_file_path` | string | Yes | Path to original file |
| `new_file_path` | string | Yes | Path to modified file |
| `new_file_contents` | string | Yes | Content of modified file |
| `tab_name` | string | No | Name for the diff tab |

**Response**: `"FILE_SAVED"` or `"DIFF_REJECTED"`

#### `getCurrentSelection`
Retrieves the current editor selection.

**Parameters**: None

**Response**:
```json
{
  "content": [{
    "type": "text",
    "text": "{\"success\": true, \"text\": \"selected text\", \"filePath\": \"/path/to/file\", \"selection\": {...}}"
  }]
}
```

#### `getLatestSelection`
Retrieves the most recent selection across all editors.

**Parameters**: None

#### `getOpenEditors`
Lists currently open editor tabs.

**Parameters**: None

**Response**:
```json
{
  "content": [{
    "type": "text",
    "text": "[{\"uri\": \"file:///path\", \"isActive\": true, \"label\": \"file.js\", \"languageId\": \"javascript\", \"isDirty\": false}]"
  }]
}
```

#### `getWorkspaceFolders`
Returns all open workspace folders.

**Parameters**: None

**Response**:
```json
{
  "content": [{
    "type": "text",
    "text": "{\"folders\": [\"/path/to/project\"], \"rootPath\": \"/path/to/project\"}"
  }]
}
```

#### `getDiagnostics`
Retrieves language diagnostics (errors, warnings) from the editor.

**Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `uri` | string | No | File URI (omit for all files) |

**Response**:
```json
{
  "content": [{
    "type": "text",
    "text": "[{\"message\": \"Error message\", \"severity\": 1, \"range\": {...}, \"source\": \"typescript\"}]"
  }]
}
```

#### `checkDocumentDirty`
Checks if a file has unsaved changes.

**Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `filePath` | string | Yes | Path to the file |

**Response**:
```json
{
  "content": [{
    "type": "text",
    "text": "{\"isDirty\": false, \"isUntitled\": false}"
  }]
}
```

#### `saveDocument`
Saves a specified file.

**Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `filePath` | string | Yes | Path to the file |

**Response**:
```json
{
  "content": [{
    "type": "text",
    "text": "{\"success\": true, \"message\": \"Document saved\"}"
  }]
}
```

#### `close_tab`
Closes a named tab.

**Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tab_name` | string | Yes | Name of the tab to close |

**Response**: `"TAB_CLOSED"`

#### `closeAllDiffTabs`
Closes all diff view tabs.

**Parameters**: None

**Response**: `"CLOSED_${count}_DIFF_TABS"`

#### `executeCode`
Executes Python code in a Jupyter kernel (if available).

**Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `code` | string | Yes | Python code to execute |

**Response** (mixed content):
```json
{
  "content": [
    {"type": "text", "text": "output text"},
    {"type": "image", "data": "base64...", "mimeType": "image/png"}
  ]
}
```

---

## Agent Client Protocol (ACP)

ACP is an open standard for connecting any AI agent with any compatible editor.

### Protocol Stack

- **Transport**: Newline-delimited JSON over stdio (stdin/stdout)
- **Message Format**: JSON-RPC 2.0
- **Specification**: [agentclientprotocol.com](https://agentclientprotocol.com)

### Architecture Layers

1. **Application Layer**: Custom logic for specific agent/editor behaviors
2. **Session Layer**: Stateful conversations with history preservation
3. **Connection Layer**: Initialization and optional authentication
4. **Protocol Layer**: JSON-RPC 2.0 message handling
5. **Transport Layer**: Newline-delimited JSON over stdio

### Message Format

Same JSON-RPC 2.0 format as the IDE protocol:

```json
{
  "jsonrpc": "2.0",
  "method": "method_name",
  "params": {},
  "id": "unique-id"
}
```

### Using the ACP Adapter

Install and run the official adapter:

```bash
npm install @zed-industries/claude-code-acp
ANTHROPIC_API_KEY=sk-... claude-code-acp
```

### Key Features

- **Streaming Responses**: Real-time feedback during code generation
- **Session Management**: Conversation history across interactions
- **Tool Calls & Permissions**: Structured tool usage and permission workflows
- **MCP Integration**: Passes MCP server endpoints to the agent

### ACP-Compatible Editors

- **Zed**: Native support
- **Emacs**: via agent-shell.el
- **Neovim**: CodeCompanion.nvim or avante.nvim
- **marimo**: Python notebook integration

---

## SDK Headless Mode

For direct programmatic access without an IDE.

### Basic Usage

```bash
# Simple query
claude -p "Your prompt here"

# JSON output
claude -p "Your prompt" --output-format json

# Streaming JSON
claude -p "Your prompt" --output-format stream-json
```

### Output Formats

| Format | Description |
|--------|-------------|
| `text` | Human-readable text (default) |
| `json` | Final result as JSON |
| `stream-json` | Streaming JSONL with all messages |

### Message Types (JSONL Format)

When using `--output-format stream-json`, each line is a JSON object representing a message.

#### SystemMessage
System initialization and control messages.

```json
{
  "type": "system",
  "subtype": "init",
  "data": {
    "session_id": "...",
    "version": "..."
  }
}
```

#### UserMessage
Echoed user input.

```json
{
  "type": "user",
  "content": [
    {"type": "text", "text": "User prompt"}
  ]
}
```

#### AssistantMessage
Claude's responses and tool invocations.

```json
{
  "type": "assistant",
  "content": [
    {"type": "text", "text": "Response text"},
    {
      "type": "tool_use",
      "id": "tool_use_id",
      "name": "tool_name",
      "input": {}
    }
  ],
  "model": "claude-sonnet-4-20250514"
}
```

#### ResultMessage
Final result with cost and usage information.

```json
{
  "type": "result",
  "subtype": "success",
  "total_cost_usd": 0.0042,
  "is_error": false,
  "duration_ms": 5234,
  "duration_api_ms": 4821,
  "num_turns": 3,
  "session_id": "...",
  "result": "Final output text"
}
```

### Content Block Types

#### TextBlock
```json
{"type": "text", "text": "Content"}
```

#### ThinkingBlock
```json
{"type": "thinking", "thinking": "Claude's reasoning..."}
```

#### ToolUseBlock
```json
{
  "type": "tool_use",
  "id": "unique-id",
  "name": "tool_name",
  "input": {"param": "value"}
}
```

#### ToolResultBlock
```json
{
  "type": "tool_result",
  "tool_use_id": "unique-id",
  "content": "Result content"
}
```

### SDK Libraries

| Language | Package |
|----------|---------|
| TypeScript | `@anthropic-ai/claude-agent-sdk` |
| Python | `claude-agent-sdk` |
| Rust | `claude-code-sdk` (crates.io) |
| Go | `github.com/f-pisani/claude-code-sdk-go` |

### Python SDK Example

```python
from claude_agent_sdk import query, ClaudeSDKClient, ClaudeAgentOptions

# Simple query
async for message in query(prompt="Your request"):
    if hasattr(message, 'content'):
        for block in message.content:
            if hasattr(block, 'text'):
                print(block.text)

# Interactive client with custom tools
options = ClaudeAgentOptions(
    system_prompt="You are a helpful assistant",
    allowed_tools=["Read", "Write", "Bash"],
    permission_mode="acceptEdits",
    cwd="/path/to/project"
)

async with ClaudeSDKClient(options=options) as client:
    await client.query("Your prompt")
    async for msg in client.receive_response():
        print(msg)
```

### TypeScript SDK Example

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

for await (const message of query({ prompt: 'Your request' })) {
  if (message.type === 'assistant') {
    for (const block of message.content) {
      if (block.type === 'text') {
        console.log(block.text);
      }
    }
  }
}
```

---

## Message Type Reference

### Complete Union Types

```typescript
// TypeScript type definitions
type Message = UserMessage | AssistantMessage | SystemMessage | ResultMessage;

type ContentBlock = TextBlock | ThinkingBlock | ToolUseBlock | ToolResultBlock;

interface TextBlock {
  type: "text";
  text: string;
}

interface ThinkingBlock {
  type: "thinking";
  thinking: string;
}

interface ToolUseBlock {
  type: "tool_use";
  id: string;
  name: string;
  input: Record<string, unknown>;
}

interface ToolResultBlock {
  type: "tool_result";
  tool_use_id: string;
  content: string | ContentBlock[];
}

interface AssistantMessage {
  type: "assistant";
  content: ContentBlock[];
  model: string;
}

interface UserMessage {
  type: "user";
  content: ContentBlock[];
}

interface SystemMessage {
  type: "system";
  subtype: string;
  data: Record<string, unknown>;
}

interface ResultMessage {
  type: "result";
  subtype: "success" | "error";
  total_cost_usd?: number;
  is_error: boolean;
  duration_ms: number;
  duration_api_ms: number;
  num_turns: number;
  session_id: string;
  result?: string;
  usage?: Record<string, unknown>;
}
```

---

## Security Considerations

### IDE Integration Protocol

1. **Localhost Only**: Always bind WebSocket server to `127.0.0.1` only
2. **Authentication**: Use UUID v4 tokens in lock files
3. **Token Validation**: Validate `x-claude-code-ide-authorization` header
4. **Lock File Security**: Set appropriate file permissions

### SDK/ACP

1. **API Key Protection**: Never expose `ANTHROPIC_API_KEY` in logs or errors
2. **Input Validation**: Validate all inputs before passing to Claude
3. **Permission Controls**: Use `allowed_tools` to restrict tool access
4. **Environment Isolation**: Use `cwd` to limit file system access

---

## Implementation Examples

### Minimal WebSocket MCP Server (Node.js)

```javascript
const WebSocket = require('ws');
const fs = require('fs');
const path = require('path');
const { v4: uuidv4 } = require('uuid');

const port = Math.floor(Math.random() * 55535) + 10000;
const authToken = uuidv4();

// Create WebSocket server
const wss = new WebSocket.Server({ host: '127.0.0.1', port });

// Write lock file
const lockDir = path.join(process.env.HOME, '.claude', 'ide');
fs.mkdirSync(lockDir, { recursive: true });
const lockFile = path.join(lockDir, `${port}.lock`);
fs.writeFileSync(lockFile, JSON.stringify({
  pid: process.pid,
  workspaceFolders: [process.cwd()],
  ideName: 'Custom IDE',
  transport: 'ws',
  authToken
}));

wss.on('connection', (ws, req) => {
  // Validate auth token
  const token = req.headers['x-claude-code-ide-authorization'];
  if (token !== authToken) {
    ws.close(1008, 'Invalid auth token');
    return;
  }

  ws.on('message', (data) => {
    const message = JSON.parse(data);

    // Handle JSON-RPC requests
    if (message.method) {
      handleRequest(ws, message);
    }
  });
});

function handleRequest(ws, request) {
  const { id, method, params } = request;

  switch (method) {
    case 'tools/call':
      handleToolCall(ws, id, params);
      break;
    default:
      ws.send(JSON.stringify({
        jsonrpc: '2.0',
        id,
        error: { code: -32601, message: 'Method not found' }
      }));
  }
}

function handleToolCall(ws, id, params) {
  const { name, arguments: args } = params;

  let result;
  switch (name) {
    case 'getWorkspaceFolders':
      result = {
        content: [{
          type: 'text',
          text: JSON.stringify({ folders: [process.cwd()], rootPath: process.cwd() })
        }]
      };
      break;
    // Add more tool handlers...
    default:
      result = { content: [{ type: 'text', text: `Unknown tool: ${name}` }] };
  }

  ws.send(JSON.stringify({ jsonrpc: '2.0', id, result }));
}

// Cleanup on exit
process.on('exit', () => fs.unlinkSync(lockFile));
process.on('SIGINT', () => process.exit());
```

### ACP Adapter Usage

```bash
# Install the adapter
npm install -g @zed-industries/claude-code-acp

# Run with API key
ANTHROPIC_API_KEY=your-key claude-code-acp
```

---

## References

- [Claude Code Documentation](https://docs.anthropic.com/en/docs/claude-code)
- [Agent Client Protocol](https://agentclientprotocol.com)
- [MCP Specification](https://modelcontextprotocol.io)
- [Claude Agent SDK (Python)](https://github.com/anthropics/claude-agent-sdk-python)
- [Claude Code ACP Adapter](https://github.com/zed-industries/claude-code-acp)
- [Neovim Plugin (reverse-engineered)](https://github.com/coder/claudecode.nvim)
