# Claude Code Web UI - Build Prompt

Build a self-hosted web UI for Claude Code by proxying ACP (Agent Client Protocol) messages between a browser and the `@zed-industries/claude-code-acp` npm package.

## Architecture

```
Browser ←—WebSocket—→ Node.js Server ←—stdio—→ claude-code-acp process
```

The server spawns `claude-code-acp` as a subprocess and pipes JSON-RPC messages between the browser (via WebSocket) and the agent (via stdin/stdout).

## Phase 1: Discovery (Do This First)

Before writing any UI code, discover the actual message format:

```bash
# Create a test script
cat > test-acp.js << 'EOF'
const { spawn } = require('child_process');
const readline = require('readline');

const agent = spawn('npx', ['@zed-industries/claude-code-acp'], {
  stdio: ['pipe', 'pipe', 'pipe'],
  env: { ...process.env }
});

const rl = readline.createInterface({ input: agent.stdout });

// Log all messages from agent
rl.on('line', (line) => {
  console.log('AGENT:', line);
  try {
    const msg = JSON.parse(line);
    console.log('PARSED:', JSON.stringify(msg, null, 2));
  } catch {}
});

agent.stderr.on('data', (d) => console.error('STDERR:', d.toString()));

// Send initialize
const init = {
  jsonrpc: '2.0',
  id: 1,
  method: 'initialize',
  params: {
    clientVersion: 'test/1.0',
    capabilities: {
      filesystem: { read: true, write: true, roots: [process.cwd()] },
      terminal: { run: true, background: true }
    }
  }
};

console.log('SENDING:', JSON.stringify(init));
agent.stdin.write(JSON.stringify(init) + '\n');

// Keep process alive
process.stdin.resume();

// Allow manual message input
const userRl = readline.createInterface({ input: process.stdin });
userRl.on('line', (line) => {
  console.log('SENDING:', line);
  agent.stdin.write(line + '\n');
});
EOF

# Run it
ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY node test-acp.js
```

Capture the output and document:
1. What methods are available (from initialize response)
2. What notification types exist
3. Exact field names and structures

## Phase 2: Server (~50 lines)

Create `server/index.ts`:

```typescript
import { spawn, ChildProcess } from "child_process";
import { createServer } from "http";
import { WebSocketServer, WebSocket } from "ws";
import express from "express";
import path from "path";
import readline from "readline";

const app = express();
const PORT = process.env.PORT || 3000;

app.use(express.static(path.join(__dirname, "../dist/client")));
app.get("*", (req, res) => res.sendFile(path.join(__dirname, "../dist/client/index.html")));

const server = createServer(app);
const wss = new WebSocketServer({ server });

wss.on("connection", (ws) => {
  console.log("[Server] Browser connected");

  const agent = spawn("npx", ["@zed-industries/claude-code-acp"], {
    stdio: ["pipe", "pipe", "pipe"],
    env: { ...process.env, ANTHROPIC_API_KEY: process.env.ANTHROPIC_API_KEY },
  });

  const rl = readline.createInterface({ input: agent.stdout! });

  // Agent → Browser
  rl.on("line", (line) => {
    if (ws.readyState === WebSocket.OPEN) ws.send(line);
  });

  agent.stderr?.on("data", (data) => console.error("[Agent]", data.toString()));

  // Browser → Agent
  ws.on("message", (data) => agent.stdin?.write(data.toString() + "\n"));

  // Cleanup
  ws.on("close", () => { agent.kill(); console.log("[Server] Disconnected"); });
  agent.on("close", (code) => { ws.close(); console.log("[Server] Agent exited:", code); });
});

server.listen(PORT, () => console.log(`Server: http://localhost:${PORT}`));
```

## Phase 3: Types

Create types based on what you discovered in Phase 1. Expected structure:

```typescript
// client/types/acp.ts

// JSON-RPC base
export interface JsonRpcMessage {
  jsonrpc: "2.0";
  id?: number | string;
  method?: string;
  params?: unknown;
  result?: unknown;
  error?: { code: number; message: string };
}

// TODO: Fill these in based on Phase 1 discovery
export interface AcpNotification {
  // Discovered notification types go here
}
```

## Phase 4: React Hook

```typescript
// client/hooks/useAcp.ts
import { useCallback, useRef, useState } from "react";

export function useAcp(onMessage: (msg: any) => void) {
  const wsRef = useRef<WebSocket | null>(null);
  const idRef = useRef(0);
  const [connected, setConnected] = useState(false);

  const connect = useCallback(() => {
    const ws = new WebSocket(`ws://${location.host}`);
    wsRef.current = ws;
    ws.onopen = () => setConnected(true);
    ws.onclose = () => setConnected(false);
    ws.onmessage = (e) => {
      try { onMessage(JSON.parse(e.data)); } catch {}
    };
  }, [onMessage]);

  const send = useCallback((method: string, params?: unknown) => {
    wsRef.current?.send(JSON.stringify({
      jsonrpc: "2.0",
      id: ++idRef.current,
      method,
      params,
    }));
  }, []);

  return { connected, connect, send };
}
```

## Phase 5: Minimal UI

Start with just chat, add features incrementally:

```typescript
// client/App.tsx
import { useState, useCallback } from "react";
import { useAcp } from "./hooks/useAcp";

export function App() {
  const [messages, setMessages] = useState<string[]>([]);
  const [input, setInput] = useState("");
  const [sessionId, setSessionId] = useState<string | null>(null);

  const handleMessage = useCallback((msg: any) => {
    // Log everything during development
    console.log("ACP:", msg);

    // Handle based on what you discovered in Phase 1
    if (msg.result?.sessionId) {
      setSessionId(msg.result.sessionId);
    }

    // TODO: Handle notifications based on actual format
    if (msg.params?.text) {
      setMessages(prev => [...prev, msg.params.text]);
    }
  }, []);

  const { connected, connect, send } = useAcp(handleMessage);

  const initialize = () => {
    send("initialize", {
      clientVersion: "web/1.0",
      capabilities: {
        filesystem: { read: true, write: true, roots: ["/"] },
        terminal: { run: true, background: true },
      },
    });
  };

  const startSession = () => {
    send("newSession", { mcpServers: [] });
  };

  const sendPrompt = () => {
    if (!sessionId || !input.trim()) return;
    send("prompt", {
      sessionId,
      messages: [{ role: "human", content: [{ type: "text", text: input }] }],
    });
    setInput("");
  };

  return (
    <div style={{ padding: 20, fontFamily: "monospace" }}>
      <h1>Claude Code Web</h1>

      {!connected ? (
        <button onClick={connect}>Connect</button>
      ) : !sessionId ? (
        <div>
          <button onClick={initialize}>Initialize</button>
          <button onClick={startSession}>Start Session</button>
        </div>
      ) : (
        <div>
          <div style={{ border: "1px solid #ccc", padding: 10, minHeight: 200, marginBottom: 10 }}>
            {messages.map((m, i) => <div key={i}>{m}</div>)}
          </div>
          <input
            value={input}
            onChange={(e) => setInput(e.target.value)}
            onKeyDown={(e) => e.key === "Enter" && sendPrompt()}
            style={{ width: 300 }}
          />
          <button onClick={sendPrompt}>Send</button>
        </div>
      )}

      <pre style={{ fontSize: 10, marginTop: 20 }}>
        Connected: {String(connected)} | Session: {sessionId || "none"}
      </pre>
    </div>
  );
}
```

## Phase 6: Iterate Based on Reality

Once basic chat works:

1. **Log all messages** - See exactly what the agent sends
2. **Add notification handlers** - One at a time based on actual types
3. **Add UI components** - Permission modal, diff view, tool status
4. **Style it** - Tailwind, proper layout

## Files to Create

```
project/
├── server/
│   └── index.ts          # WebSocket proxy
├── client/
│   ├── index.html
│   ├── main.tsx
│   ├── App.tsx           # Start minimal, grow
│   ├── hooks/
│   │   └── useAcp.ts
│   └── types/
│       └── acp.ts        # Fill from Phase 1
├── package.json
├── tsconfig.json
└── vite.config.ts
```

## package.json

```json
{
  "name": "claude-code-web",
  "type": "module",
  "scripts": {
    "dev": "concurrently \"tsx watch server/index.ts\" \"vite\"",
    "build": "vite build && tsc -p tsconfig.server.json"
  },
  "dependencies": {
    "express": "^4.18.2",
    "ws": "^8.14.2"
  },
  "devDependencies": {
    "@types/express": "^4.17.21",
    "@types/ws": "^8.5.10",
    "@vitejs/plugin-react": "^4.2.0",
    "concurrently": "^8.2.2",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "tsx": "^4.6.2",
    "typescript": "^5.3.2",
    "vite": "^5.0.0"
  }
}
```

## Key Principle

**Don't assume message formats. Discover them.**

1. Run the agent
2. Send messages
3. Log what comes back
4. Build UI for what actually exists

The architecture (WebSocket ↔ stdio proxy) is solid. The message types need to be discovered by actually running the agent.
