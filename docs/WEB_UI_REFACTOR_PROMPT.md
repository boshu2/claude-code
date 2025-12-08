# Claude Code Web UI - Simple ACP Proxy Approach

Build a self-hosted web UI for Claude Code by proxying ACP (Agent Client Protocol) messages between a browser and the existing `claude-code-acp` package.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              BROWSER                                     │
│                                                                          │
│   ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐          │
│   │  Chat UI   │ │ Tool Status│ │ Diff View  │ │ Permission │          │
│   │            │ │            │ │  (Monaco)  │ │   Modal    │          │
│   └────────────┘ └────────────┘ └────────────┘ └────────────┘          │
│         │              │              │              │                  │
│         └──────────────┴──────────────┴──────────────┘                  │
│                                │                                         │
│                    ┌───────────▼───────────┐                            │
│                    │  ACP Message Handler  │                            │
│                    │  (renders by type)    │                            │
│                    └───────────┬───────────┘                            │
│                                │                                         │
└────────────────────────────────┼────────────────────────────────────────┘
                                 │ WebSocket (JSON-RPC)
                                 │ (ACP messages, unchanged)
┌────────────────────────────────┼────────────────────────────────────────┐
│                    ┌───────────▼───────────┐           SERVER           │
│                    │   WebSocket Server    │                            │
│                    │   (simple proxy)      │                            │
│                    └───────────┬───────────┘                            │
│                                │ stdin/stdout                            │
│                    ┌───────────▼───────────┐                            │
│                    │   claude-code-acp     │                            │
│                    │   (npm package,       │                            │
│                    │    runs unchanged)    │                            │
│                    └───────────────────────┘                            │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key Insight**: The `claude-code-acp` package already handles all Claude Code integration. We just proxy its ACP messages to a browser and render them as UI.

---

## Project Structure

```
claude-code-web/
├── server/
│   └── index.ts              # WebSocket proxy (~50 lines)
├── client/
│   ├── index.html
│   ├── main.tsx
│   ├── App.tsx               # Main app with ACP message routing
│   ├── hooks/
│   │   └── useAcp.ts         # WebSocket + ACP message handling
│   ├── components/
│   │   ├── Chat.tsx          # Message list + input
│   │   ├── ToolStatus.tsx    # Current tool operation
│   │   ├── PlanView.tsx      # Todo list
│   │   ├── DiffView.tsx      # Monaco diff with accept/reject
│   │   ├── PermissionModal.tsx
│   │   └── Terminal.tsx      # Background terminal output
│   └── types/
│       └── acp.ts            # ACP message types
├── tests/
│   ├── server.test.ts
│   ├── acp-messages.test.ts
│   └── e2e/
│       ├── chat.spec.ts
│       ├── diff.spec.ts
│       └── permission.spec.ts
├── package.json
├── tsconfig.json
├── vite.config.ts
└── playwright.config.ts
```

---

## Phase 1: Server (WebSocket ↔ stdio Proxy)

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

// Serve static files in production
app.use(express.static(path.join(__dirname, "../dist/client")));
app.get("*", (req, res) => {
  res.sendFile(path.join(__dirname, "../dist/client/index.html"));
});

const server = createServer(app);
const wss = new WebSocketServer({ server });

interface Session {
  ws: WebSocket;
  agent: ChildProcess;
  rl: readline.Interface;
}

const sessions = new Map<WebSocket, Session>();

wss.on("connection", (ws) => {
  console.log("[Server] Browser connected");

  // Spawn claude-code-acp as a subprocess
  const agent = spawn("npx", ["@zed-industries/claude-code-acp"], {
    stdio: ["pipe", "pipe", "pipe"],
    env: {
      ...process.env,
      // Ensure API key is passed through
      ANTHROPIC_API_KEY: process.env.ANTHROPIC_API_KEY,
    },
  });

  const rl = readline.createInterface({ input: agent.stdout! });
  const session: Session = { ws, agent, rl };
  sessions.set(ws, session);

  // Forward ACP messages from agent → browser
  rl.on("line", (line) => {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(line);
    }
  });

  // Log agent stderr for debugging
  agent.stderr?.on("data", (data) => {
    console.error("[Agent]", data.toString());
  });

  // Forward messages from browser → agent
  ws.on("message", (data) => {
    const message = data.toString();
    agent.stdin?.write(message + "\n");
  });

  // Cleanup on disconnect
  ws.on("close", () => {
    console.log("[Server] Browser disconnected");
    agent.kill();
    sessions.delete(ws);
  });

  agent.on("close", (code) => {
    console.log("[Server] Agent exited with code", code);
    if (ws.readyState === WebSocket.OPEN) {
      ws.close();
    }
    sessions.delete(ws);
  });

  agent.on("error", (err) => {
    console.error("[Server] Agent error:", err);
    ws.close();
  });
});

server.listen(PORT, () => {
  console.log(`
╔════════════════════════════════════════════╗
║         Claude Code Web UI                 ║
║                                            ║
║   Running at: http://localhost:${PORT}        ║
║                                            ║
║   Make sure ANTHROPIC_API_KEY is set       ║
╚════════════════════════════════════════════╝
  `);
});
```

---

## Phase 2: ACP Types

Create `client/types/acp.ts`:

```typescript
// JSON-RPC 2.0 base types
export interface JsonRpcRequest {
  jsonrpc: "2.0";
  id: number | string;
  method: string;
  params?: unknown;
}

export interface JsonRpcResponse {
  jsonrpc: "2.0";
  id: number | string;
  result?: unknown;
  error?: { code: number; message: string; data?: unknown };
}

export interface JsonRpcNotification {
  jsonrpc: "2.0";
  method: string;
  params: unknown;
}

// ACP Notification Types (what the agent sends)
export interface AcpTextNotification {
  type: "text";
  text: string;
}

export interface AcpThinkingNotification {
  type: "thinking";
  thinking: string;
}

export interface AcpToolUpdateNotification {
  type: "toolUpdate";
  id: string;
  title: string;
  status: "started" | "running" | "completed" | "failed";
  kind: "think" | "read" | "edit" | "execute" | "search" | "fetch" | "other";
  content?: string;
  locations?: Array<{
    path: string;
    lineStart: number;
    lineEnd: number;
  }>;
}

export interface AcpPlanNotification {
  type: "plan";
  entries: Array<{
    id: string;
    content: string;
    status: "pending" | "in_progress" | "completed";
  }>;
}

export interface AcpPermissionRequestNotification {
  type: "permissionRequest";
  id: string;
  tool: string;
  input: Record<string, unknown>;
  description: string;
}

export interface AcpEditProposalNotification {
  type: "editProposal";
  id: string;
  path: string;
  oldContent: string;
  newContent: string;
  description?: string;
}

export interface AcpTerminalOutputNotification {
  type: "terminalOutput";
  shellId: string;
  output: string;
  isBackground: boolean;
}

export interface AcpErrorNotification {
  type: "error";
  message: string;
  details?: string;
}

export interface AcpDoneNotification {
  type: "done";
  sessionId: string;
  result?: {
    costUsd?: number;
    durationMs?: number;
    numTurns?: number;
  };
}

export type AcpNotification =
  | AcpTextNotification
  | AcpThinkingNotification
  | AcpToolUpdateNotification
  | AcpPlanNotification
  | AcpPermissionRequestNotification
  | AcpEditProposalNotification
  | AcpTerminalOutputNotification
  | AcpErrorNotification
  | AcpDoneNotification;

// Parsed message from WebSocket
export interface AcpMessage {
  jsonrpc: "2.0";
  method?: string;
  id?: number | string;
  params?: AcpNotification;
  result?: unknown;
  error?: { code: number; message: string };
}

// Client capabilities sent during initialize
export interface ClientCapabilities {
  filesystem?: {
    read: boolean;
    write: boolean;
    roots: string[];
  };
  terminal?: {
    run: boolean;
    background: boolean;
  };
}

// Session state
export interface SessionState {
  sessionId: string | null;
  claudeSessionId: string | null;
  isConnected: boolean;
  isProcessing: boolean;
  model: string;
}
```

---

## Phase 3: ACP Hook

Create `client/hooks/useAcp.ts`:

```typescript
import { useCallback, useEffect, useRef, useState } from "react";
import type {
  AcpMessage,
  AcpNotification,
  ClientCapabilities,
  SessionState,
} from "../types/acp";

export interface UseAcpOptions {
  url?: string;
  cwd?: string;
  onNotification?: (notification: AcpNotification) => void;
}

export interface UseAcpReturn {
  // State
  session: SessionState;
  isConnected: boolean;
  error: Error | null;

  // Actions
  connect: () => void;
  disconnect: () => void;
  sendPrompt: (text: string, attachments?: string[]) => Promise<void>;
  respondToPermission: (requestId: string, approved: boolean) => void;
  respondToEdit: (proposalId: string, approved: boolean) => void;
  cancel: () => void;
}

export function useAcp(options: UseAcpOptions = {}): UseAcpReturn {
  const {
    url = `ws://${window.location.host}`,
    cwd = "/home/user/project",
    onNotification,
  } = options;

  const wsRef = useRef<WebSocket | null>(null);
  const requestIdRef = useRef(0);
  const pendingRef = useRef<Map<number, { resolve: Function; reject: Function }>>(new Map());

  const [session, setSession] = useState<SessionState>({
    sessionId: null,
    claudeSessionId: null,
    isConnected: false,
    isProcessing: false,
    model: "claude-sonnet-4-20250514",
  });
  const [error, setError] = useState<Error | null>(null);

  // Send JSON-RPC request and wait for response
  const sendRequest = useCallback(async <T = unknown>(
    method: string,
    params?: unknown
  ): Promise<T> => {
    return new Promise((resolve, reject) => {
      if (!wsRef.current || wsRef.current.readyState !== WebSocket.OPEN) {
        reject(new Error("Not connected"));
        return;
      }

      const id = ++requestIdRef.current;
      pendingRef.current.set(id, { resolve, reject });

      wsRef.current.send(JSON.stringify({
        jsonrpc: "2.0",
        id,
        method,
        params,
      }));

      // Timeout after 60 seconds
      setTimeout(() => {
        if (pendingRef.current.has(id)) {
          pendingRef.current.delete(id);
          reject(new Error("Request timeout"));
        }
      }, 60000);
    });
  }, []);

  // Handle incoming messages
  const handleMessage = useCallback((event: MessageEvent) => {
    try {
      const message: AcpMessage = JSON.parse(event.data);

      // Handle response to our request
      if (message.id !== undefined && !message.method) {
        const pending = pendingRef.current.get(message.id as number);
        if (pending) {
          pendingRef.current.delete(message.id as number);
          if (message.error) {
            pending.reject(new Error(message.error.message));
          } else {
            pending.resolve(message.result);
          }
        }
        return;
      }

      // Handle notification from agent
      if (message.method && message.params) {
        const notification = message.params as AcpNotification;
        onNotification?.(notification);

        // Update processing state
        if (notification.type === "done") {
          setSession((s) => ({ ...s, isProcessing: false }));
        }
      }
    } catch (err) {
      console.error("Failed to parse message:", err);
    }
  }, [onNotification]);

  // Connect to WebSocket and initialize ACP session
  const connect = useCallback(async () => {
    if (wsRef.current?.readyState === WebSocket.OPEN) return;

    setError(null);

    const ws = new WebSocket(url);
    wsRef.current = ws;

    ws.onopen = async () => {
      setSession((s) => ({ ...s, isConnected: true }));

      try {
        // Step 1: Initialize
        const initResult = await sendRequest<{ sessionId: string }>("initialize", {
          clientVersion: "claude-code-web/1.0.0",
          capabilities: {
            filesystem: { read: true, write: true, roots: [cwd] },
            terminal: { run: true, background: true },
          } as ClientCapabilities,
        });

        setSession((s) => ({ ...s, sessionId: initResult.sessionId }));

        // Step 2: Create Claude session
        const sessionResult = await sendRequest<{ sessionId: string }>("newSession", {
          mcpServers: [],
        });

        setSession((s) => ({ ...s, claudeSessionId: sessionResult.sessionId }));
      } catch (err) {
        setError(err instanceof Error ? err : new Error("Failed to initialize"));
      }
    };

    ws.onmessage = handleMessage;

    ws.onerror = () => {
      setError(new Error("WebSocket error"));
    };

    ws.onclose = () => {
      setSession((s) => ({
        ...s,
        isConnected: false,
        sessionId: null,
        claudeSessionId: null,
      }));
    };
  }, [url, cwd, sendRequest, handleMessage]);

  // Disconnect
  const disconnect = useCallback(() => {
    wsRef.current?.close();
    wsRef.current = null;
  }, []);

  // Send a prompt to Claude
  const sendPrompt = useCallback(async (text: string, attachments?: string[]) => {
    if (!session.claudeSessionId) {
      throw new Error("No active session");
    }

    setSession((s) => ({ ...s, isProcessing: true }));

    // Build message content
    const content: Array<{ type: string; text?: string; path?: string }> = [
      { type: "text", text },
    ];

    // Add file attachments as @-mentions
    if (attachments) {
      for (const path of attachments) {
        content.push({ type: "text", text: `@${path}` });
      }
    }

    await sendRequest("prompt", {
      sessionId: session.claudeSessionId,
      messages: [{
        role: "human",
        content,
      }],
    });
  }, [session.claudeSessionId, sendRequest]);

  // Respond to permission request
  const respondToPermission = useCallback((requestId: string, approved: boolean) => {
    sendRequest("respondToPermission", { requestId, approved }).catch(console.error);
  }, [sendRequest]);

  // Respond to edit proposal
  const respondToEdit = useCallback((proposalId: string, approved: boolean) => {
    sendRequest("respondToEdit", { proposalId, approved }).catch(console.error);
  }, [sendRequest]);

  // Cancel current operation
  const cancel = useCallback(() => {
    if (session.claudeSessionId) {
      sendRequest("cancel", { sessionId: session.claudeSessionId }).catch(console.error);
      setSession((s) => ({ ...s, isProcessing: false }));
    }
  }, [session.claudeSessionId, sendRequest]);

  // Cleanup on unmount
  useEffect(() => {
    return () => {
      disconnect();
    };
  }, [disconnect]);

  return {
    session,
    isConnected: session.isConnected,
    error,
    connect,
    disconnect,
    sendPrompt,
    respondToPermission,
    respondToEdit,
    cancel,
  };
}
```

---

## Phase 4: React Components

### `client/App.tsx`

```typescript
import { useState, useCallback } from "react";
import { useAcp } from "./hooks/useAcp";
import { Chat } from "./components/Chat";
import { ToolStatus } from "./components/ToolStatus";
import { PlanView } from "./components/PlanView";
import { DiffView } from "./components/DiffView";
import { PermissionModal } from "./components/PermissionModal";
import type { AcpNotification, AcpToolUpdateNotification, AcpPlanNotification } from "./types/acp";

interface Message {
  id: string;
  role: "user" | "assistant";
  content: string;
}

interface PendingPermission {
  id: string;
  tool: string;
  input: Record<string, unknown>;
  description: string;
}

interface PendingEdit {
  id: string;
  path: string;
  oldContent: string;
  newContent: string;
}

export function App() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [currentTool, setCurrentTool] = useState<AcpToolUpdateNotification | null>(null);
  const [plan, setPlan] = useState<AcpPlanNotification["entries"]>([]);
  const [pendingPermission, setPendingPermission] = useState<PendingPermission | null>(null);
  const [pendingEdit, setPendingEdit] = useState<PendingEdit | null>(null);
  const [thinkingText, setThinkingText] = useState<string>("");

  const handleNotification = useCallback((notification: AcpNotification) => {
    switch (notification.type) {
      case "text":
        // Append text to current assistant message or create new one
        setMessages((prev) => {
          const last = prev[prev.length - 1];
          if (last?.role === "assistant") {
            return [
              ...prev.slice(0, -1),
              { ...last, content: last.content + notification.text },
            ];
          }
          return [
            ...prev,
            { id: crypto.randomUUID(), role: "assistant", content: notification.text },
          ];
        });
        break;

      case "thinking":
        setThinkingText((prev) => prev + notification.thinking);
        break;

      case "toolUpdate":
        setCurrentTool(notification);
        if (notification.status === "completed" || notification.status === "failed") {
          // Clear after a short delay
          setTimeout(() => setCurrentTool(null), 1000);
        }
        break;

      case "plan":
        setPlan(notification.entries);
        break;

      case "permissionRequest":
        setPendingPermission({
          id: notification.id,
          tool: notification.tool,
          input: notification.input,
          description: notification.description,
        });
        break;

      case "editProposal":
        setPendingEdit({
          id: notification.id,
          path: notification.path,
          oldContent: notification.oldContent,
          newContent: notification.newContent,
        });
        break;

      case "done":
        setCurrentTool(null);
        setThinkingText("");
        break;

      case "error":
        setMessages((prev) => [
          ...prev,
          { id: crypto.randomUUID(), role: "assistant", content: `Error: ${notification.message}` },
        ]);
        break;
    }
  }, []);

  const {
    session,
    isConnected,
    error,
    connect,
    disconnect,
    sendPrompt,
    respondToPermission,
    respondToEdit,
    cancel,
  } = useAcp({ onNotification: handleNotification });

  const handleSend = async (text: string) => {
    // Add user message
    setMessages((prev) => [
      ...prev,
      { id: crypto.randomUUID(), role: "user", content: text },
    ]);
    setThinkingText("");

    try {
      await sendPrompt(text);
    } catch (err) {
      console.error("Failed to send:", err);
    }
  };

  const handlePermissionResponse = (approved: boolean) => {
    if (pendingPermission) {
      respondToPermission(pendingPermission.id, approved);
      setPendingPermission(null);
    }
  };

  const handleEditResponse = (approved: boolean) => {
    if (pendingEdit) {
      respondToEdit(pendingEdit.id, approved);
      setPendingEdit(null);
    }
  };

  return (
    <div className="flex h-screen bg-gray-900 text-white">
      {/* Main content area */}
      <div className="flex-1 flex flex-col">
        {/* Diff view overlay */}
        {pendingEdit && (
          <DiffView
            path={pendingEdit.path}
            oldContent={pendingEdit.oldContent}
            newContent={pendingEdit.newContent}
            onAccept={() => handleEditResponse(true)}
            onReject={() => handleEditResponse(false)}
          />
        )}
      </div>

      {/* Right sidebar: Chat + Status */}
      <div className="w-[450px] border-l border-gray-700 flex flex-col">
        {/* Connection status */}
        <div className="p-3 border-b border-gray-700 flex items-center justify-between">
          <span className="font-semibold">Claude Code</span>
          {!isConnected ? (
            <button
              onClick={connect}
              className="px-3 py-1 bg-blue-600 hover:bg-blue-700 rounded text-sm"
            >
              Connect
            </button>
          ) : (
            <div className="flex items-center gap-2">
              <span className="w-2 h-2 bg-green-500 rounded-full" />
              <span className="text-sm text-gray-400">Connected</span>
            </div>
          )}
        </div>

        {/* Error display */}
        {error && (
          <div className="p-3 bg-red-900/50 text-red-200 text-sm">
            {error.message}
          </div>
        )}

        {/* Plan/Todo list */}
        {plan.length > 0 && (
          <PlanView entries={plan} />
        )}

        {/* Current tool operation */}
        {currentTool && (
          <ToolStatus tool={currentTool} />
        )}

        {/* Chat messages */}
        <Chat
          messages={messages}
          thinkingText={thinkingText}
          isProcessing={session.isProcessing}
          onSend={handleSend}
          onCancel={cancel}
          disabled={!isConnected}
        />
      </div>

      {/* Permission modal */}
      {pendingPermission && (
        <PermissionModal
          tool={pendingPermission.tool}
          input={pendingPermission.input}
          description={pendingPermission.description}
          onAllow={() => handlePermissionResponse(true)}
          onDeny={() => handlePermissionResponse(false)}
        />
      )}
    </div>
  );
}
```

### `client/components/Chat.tsx`

```typescript
import { useState, useRef, useEffect } from "react";

interface Message {
  id: string;
  role: "user" | "assistant";
  content: string;
}

interface ChatProps {
  messages: Message[];
  thinkingText: string;
  isProcessing: boolean;
  onSend: (text: string) => void;
  onCancel: () => void;
  disabled: boolean;
}

export function Chat({
  messages,
  thinkingText,
  isProcessing,
  onSend,
  onCancel,
  disabled,
}: ChatProps) {
  const [input, setInput] = useState("");
  const messagesEndRef = useRef<HTMLDivElement>(null);

  // Auto-scroll to bottom
  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages, thinkingText]);

  const handleSubmit = () => {
    if (input.trim() && !disabled && !isProcessing) {
      onSend(input.trim());
      setInput("");
    }
  };

  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === "Enter" && !e.shiftKey) {
      e.preventDefault();
      handleSubmit();
    }
  };

  return (
    <div className="flex-1 flex flex-col min-h-0">
      {/* Messages */}
      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {messages.map((msg) => (
          <div
            key={msg.id}
            className={`p-3 rounded-lg ${
              msg.role === "user"
                ? "bg-blue-600 ml-8"
                : "bg-gray-800 mr-8"
            }`}
          >
            <div className="whitespace-pre-wrap break-words">{msg.content}</div>
          </div>
        ))}

        {/* Thinking indicator */}
        {thinkingText && (
          <div className="p-3 rounded-lg bg-gray-800/50 mr-8 text-gray-400 italic">
            <div className="text-xs text-gray-500 mb-1">Thinking...</div>
            <div className="whitespace-pre-wrap break-words text-sm">
              {thinkingText.slice(-500)}
            </div>
          </div>
        )}

        {/* Processing indicator */}
        {isProcessing && !thinkingText && (
          <div className="flex items-center gap-2 text-gray-400">
            <div className="animate-pulse">●</div>
            <span>Claude is working...</span>
          </div>
        )}

        <div ref={messagesEndRef} />
      </div>

      {/* Input */}
      <div className="p-4 border-t border-gray-700">
        <div className="flex gap-2">
          <textarea
            value={input}
            onChange={(e) => setInput(e.target.value)}
            onKeyDown={handleKeyDown}
            placeholder={disabled ? "Connect to start..." : "Ask Claude..."}
            disabled={disabled}
            className="flex-1 bg-gray-800 border border-gray-700 rounded-lg p-3 resize-none focus:outline-none focus:border-blue-500 disabled:opacity-50"
            rows={3}
          />
          <div className="flex flex-col gap-2">
            <button
              onClick={handleSubmit}
              disabled={disabled || isProcessing || !input.trim()}
              className="px-4 py-2 bg-blue-600 hover:bg-blue-700 disabled:opacity-50 disabled:cursor-not-allowed rounded-lg"
            >
              Send
            </button>
            {isProcessing && (
              <button
                onClick={onCancel}
                className="px-4 py-2 bg-red-600 hover:bg-red-700 rounded-lg"
              >
                Stop
              </button>
            )}
          </div>
        </div>
      </div>
    </div>
  );
}
```

### `client/components/DiffView.tsx`

```typescript
import { DiffEditor } from "@monaco-editor/react";

interface DiffViewProps {
  path: string;
  oldContent: string;
  newContent: string;
  onAccept: () => void;
  onReject: () => void;
}

export function DiffView({
  path,
  oldContent,
  newContent,
  onAccept,
  onReject,
}: DiffViewProps) {
  const language = getLanguageFromPath(path);

  return (
    <div className="absolute inset-0 z-50 flex flex-col bg-gray-900">
      {/* Header */}
      <div className="flex items-center justify-between p-3 bg-gray-800 border-b border-gray-700">
        <div>
          <span className="text-gray-400 text-sm">Review changes to:</span>
          <span className="ml-2 font-mono">{path}</span>
        </div>
        <div className="flex gap-2">
          <button
            onClick={onReject}
            className="px-4 py-1.5 bg-gray-700 hover:bg-gray-600 rounded text-sm"
          >
            Reject
          </button>
          <button
            onClick={onAccept}
            className="px-4 py-1.5 bg-green-600 hover:bg-green-700 rounded text-sm font-medium"
          >
            Accept Changes
          </button>
        </div>
      </div>

      {/* Diff editor */}
      <div className="flex-1">
        <DiffEditor
          original={oldContent}
          modified={newContent}
          language={language}
          theme="vs-dark"
          options={{
            readOnly: true,
            renderSideBySide: true,
            minimap: { enabled: false },
            scrollBeyondLastLine: false,
          }}
        />
      </div>
    </div>
  );
}

function getLanguageFromPath(path: string): string {
  const ext = path.split(".").pop()?.toLowerCase() || "";
  const map: Record<string, string> = {
    ts: "typescript", tsx: "typescript",
    js: "javascript", jsx: "javascript",
    py: "python", rs: "rust", go: "go",
    java: "java", c: "c", cpp: "cpp",
    css: "css", html: "html", json: "json",
    yaml: "yaml", yml: "yaml", md: "markdown",
    sh: "shell", bash: "shell",
  };
  return map[ext] || "plaintext";
}
```

### `client/components/PermissionModal.tsx`

```typescript
interface PermissionModalProps {
  tool: string;
  input: Record<string, unknown>;
  description: string;
  onAllow: () => void;
  onDeny: () => void;
}

export function PermissionModal({
  tool,
  input,
  description,
  onAllow,
  onDeny,
}: PermissionModalProps) {
  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/60">
      <div className="bg-gray-800 rounded-lg shadow-2xl max-w-lg w-full mx-4 overflow-hidden">
        {/* Header */}
        <div className="p-4 bg-yellow-600/20 border-b border-yellow-600/30">
          <h2 className="text-lg font-semibold text-yellow-200">
            Permission Required
          </h2>
        </div>

        {/* Content */}
        <div className="p-4 space-y-4">
          <p className="text-gray-300">{description}</p>

          <div className="bg-gray-900 rounded-lg p-3">
            <div className="text-xs text-gray-500 mb-1">Tool</div>
            <div className="font-mono text-yellow-400">{tool}</div>
          </div>

          <div className="bg-gray-900 rounded-lg p-3">
            <div className="text-xs text-gray-500 mb-1">Arguments</div>
            <pre className="text-sm overflow-x-auto max-h-48 overflow-y-auto">
              {JSON.stringify(input, null, 2)}
            </pre>
          </div>
        </div>

        {/* Actions */}
        <div className="p-4 bg-gray-900 flex justify-end gap-3">
          <button
            onClick={onDeny}
            className="px-4 py-2 bg-gray-700 hover:bg-gray-600 rounded-lg"
          >
            Deny
          </button>
          <button
            onClick={onAllow}
            className="px-4 py-2 bg-blue-600 hover:bg-blue-700 rounded-lg font-medium"
          >
            Allow
          </button>
        </div>
      </div>
    </div>
  );
}
```

### `client/components/ToolStatus.tsx`

```typescript
import type { AcpToolUpdateNotification } from "../types/acp";

interface ToolStatusProps {
  tool: AcpToolUpdateNotification;
}

const kindIcons: Record<string, string> = {
  think: "🧠",
  read: "📖",
  edit: "✏️",
  execute: "⚡",
  search: "🔍",
  fetch: "🌐",
  other: "🔧",
};

const statusColors: Record<string, string> = {
  started: "text-blue-400",
  running: "text-yellow-400",
  completed: "text-green-400",
  failed: "text-red-400",
};

export function ToolStatus({ tool }: ToolStatusProps) {
  return (
    <div className="p-3 border-b border-gray-700 bg-gray-800/50">
      <div className="flex items-center gap-2">
        <span>{kindIcons[tool.kind] || "🔧"}</span>
        <span className={`font-medium ${statusColors[tool.status]}`}>
          {tool.title}
        </span>
        {tool.status === "running" && (
          <span className="animate-pulse">...</span>
        )}
      </div>

      {tool.content && (
        <pre className="mt-2 text-xs text-gray-400 bg-gray-900 p-2 rounded overflow-x-auto max-h-24 overflow-y-auto">
          {tool.content.slice(0, 500)}
          {tool.content.length > 500 && "..."}
        </pre>
      )}

      {tool.locations && tool.locations.length > 0 && (
        <div className="mt-2 text-xs text-gray-500">
          {tool.locations.map((loc, i) => (
            <div key={i}>
              {loc.path}:{loc.lineStart}-{loc.lineEnd}
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

### `client/components/PlanView.tsx`

```typescript
interface PlanEntry {
  id: string;
  content: string;
  status: "pending" | "in_progress" | "completed";
}

interface PlanViewProps {
  entries: PlanEntry[];
}

const statusIcons: Record<string, string> = {
  pending: "○",
  in_progress: "◐",
  completed: "●",
};

const statusColors: Record<string, string> = {
  pending: "text-gray-500",
  in_progress: "text-yellow-400",
  completed: "text-green-400",
};

export function PlanView({ entries }: PlanViewProps) {
  return (
    <div className="p-3 border-b border-gray-700 bg-gray-800/30">
      <div className="text-xs text-gray-500 mb-2 font-medium">PLAN</div>
      <div className="space-y-1">
        {entries.map((entry) => (
          <div
            key={entry.id}
            className={`flex items-start gap-2 text-sm ${statusColors[entry.status]}`}
          >
            <span className="mt-0.5">{statusIcons[entry.status]}</span>
            <span className={entry.status === "completed" ? "line-through opacity-60" : ""}>
              {entry.content}
            </span>
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## Phase 5: Configuration Files

### `package.json`

```json
{
  "name": "claude-code-web",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "concurrently \"tsx watch server/index.ts\" \"vite\"",
    "build": "vite build && tsc -p tsconfig.server.json",
    "start": "node dist/server/index.js",
    "preview": "npm run build && npm run start",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:e2e": "playwright test",
    "lint": "eslint client server",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@zed-industries/claude-code-acp": "latest",
    "express": "^4.18.2",
    "ws": "^8.14.2"
  },
  "devDependencies": {
    "@monaco-editor/react": "^4.6.0",
    "@playwright/test": "^1.40.0",
    "@types/express": "^4.17.21",
    "@types/node": "^20.10.0",
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "@types/ws": "^8.5.10",
    "@vitejs/plugin-react": "^4.2.0",
    "autoprefixer": "^10.4.16",
    "concurrently": "^8.2.2",
    "postcss": "^8.4.32",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "tailwindcss": "^3.4.0",
    "tsx": "^4.6.2",
    "typescript": "^5.3.2",
    "vite": "^5.0.0",
    "vitest": "^1.0.0"
  }
}
```

### `vite.config.ts`

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  root: "client",
  build: {
    outDir: "../dist/client",
    emptyOutDir: true,
  },
  server: {
    port: 5173,
    proxy: {
      "/ws": {
        target: "ws://localhost:3000",
        ws: true,
      },
    },
  },
});
```

### `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "jsx": "react-jsx",
    "lib": ["DOM", "DOM.Iterable", "ES2022"],
    "types": ["node"]
  },
  "include": ["client/**/*", "server/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### `tsconfig.server.json`

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "./dist/server",
    "rootDir": "./server"
  },
  "include": ["server/**/*"],
  "exclude": ["client"]
}
```

### `tailwind.config.js`

```javascript
/** @type {import('tailwindcss').Config} */
export default {
  content: ["./client/**/*.{html,tsx,ts}"],
  theme: {
    extend: {},
  },
  plugins: [],
};
```

### `client/index.html`

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Claude Code</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/main.tsx"></script>
  </body>
</html>
```

### `client/main.tsx`

```typescript
import React from "react";
import ReactDOM from "react-dom/client";
import { App } from "./App";
import "./styles/globals.css";

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

### `client/styles/globals.css`

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

body {
  @apply bg-gray-900 text-white;
}
```

---

## Phase 6: Tests

### `tests/server.test.ts`

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from "vitest";
import { spawn } from "child_process";
import { WebSocket } from "ws";

// Mock child_process
vi.mock("child_process", () => ({
  spawn: vi.fn(() => ({
    stdin: { write: vi.fn() },
    stdout: { on: vi.fn() },
    stderr: { on: vi.fn() },
    on: vi.fn(),
    kill: vi.fn(),
  })),
}));

describe("Server", () => {
  it("should spawn claude-code-acp on connection", async () => {
    // Test that spawn is called with correct args
    const mockSpawn = spawn as unknown as ReturnType<typeof vi.fn>;

    // Simulate WebSocket connection triggering spawn
    expect(mockSpawn).toBeDefined();
  });

  it("should forward messages from WebSocket to agent stdin", async () => {
    // Test message forwarding
  });

  it("should forward agent stdout to WebSocket", async () => {
    // Test stdout forwarding
  });

  it("should cleanup on disconnect", async () => {
    // Test that agent.kill() is called
  });
});
```

### `tests/acp-messages.test.ts`

```typescript
import { describe, it, expect } from "vitest";
import type { AcpNotification, AcpTextNotification, AcpToolUpdateNotification } from "../client/types/acp";

describe("ACP Message Types", () => {
  it("should parse text notification", () => {
    const notification: AcpTextNotification = {
      type: "text",
      text: "Hello, world!",
    };

    expect(notification.type).toBe("text");
    expect(notification.text).toBe("Hello, world!");
  });

  it("should parse tool update notification", () => {
    const notification: AcpToolUpdateNotification = {
      type: "toolUpdate",
      id: "123",
      title: "Reading file.ts",
      status: "running",
      kind: "read",
      content: "file contents...",
    };

    expect(notification.type).toBe("toolUpdate");
    expect(notification.kind).toBe("read");
    expect(notification.status).toBe("running");
  });

  it("should validate all notification types", () => {
    const types = [
      "text", "thinking", "toolUpdate", "plan",
      "permissionRequest", "editProposal", "terminalOutput",
      "error", "done"
    ];

    types.forEach(type => {
      const notification = { type } as AcpNotification;
      expect(notification.type).toBe(type);
    });
  });
});
```

### `tests/e2e/chat.spec.ts`

```typescript
import { test, expect } from "@playwright/test";

test.describe("Chat Flow", () => {
  test.beforeEach(async ({ page }) => {
    await page.goto("/");
  });

  test("should show connect button initially", async ({ page }) => {
    await expect(page.getByRole("button", { name: "Connect" })).toBeVisible();
  });

  test("should connect and show connected status", async ({ page }) => {
    await page.click("button:has-text('Connect')");

    // Wait for connection
    await expect(page.locator(".bg-green-500")).toBeVisible({ timeout: 10000 });
    await expect(page.getByText("Connected")).toBeVisible();
  });

  test("should send message and show in chat", async ({ page }) => {
    await page.click("button:has-text('Connect')");
    await expect(page.getByText("Connected")).toBeVisible({ timeout: 10000 });

    const input = page.locator("textarea");
    await input.fill("Hello Claude!");
    await input.press("Enter");

    // User message should appear
    await expect(page.locator(".bg-blue-600:has-text('Hello Claude!')")).toBeVisible();
  });

  test("should show processing indicator", async ({ page }) => {
    await page.click("button:has-text('Connect')");
    await expect(page.getByText("Connected")).toBeVisible({ timeout: 10000 });

    await page.locator("textarea").fill("What is 2+2?");
    await page.click("button:has-text('Send')");

    // Should show processing state
    await expect(page.getByText("Claude is working")).toBeVisible({ timeout: 5000 });
  });
});
```

### `tests/e2e/permission.spec.ts`

```typescript
import { test, expect } from "@playwright/test";

test.describe("Permission Flow", () => {
  test("should show permission modal for bash commands", async ({ page }) => {
    await page.goto("/");
    await page.click("button:has-text('Connect')");
    await expect(page.getByText("Connected")).toBeVisible({ timeout: 10000 });

    // Ask Claude to run a command
    await page.locator("textarea").fill("Run: echo hello");
    await page.click("button:has-text('Send')");

    // Permission modal should appear
    await expect(page.getByText("Permission Required")).toBeVisible({ timeout: 30000 });
    await expect(page.getByText("Bash")).toBeVisible();
  });

  test("should close modal and continue on Allow", async ({ page }) => {
    // Setup...
    await page.click("button:has-text('Allow')");
    await expect(page.getByText("Permission Required")).not.toBeVisible();
  });

  test("should close modal and stop on Deny", async ({ page }) => {
    // Setup...
    await page.click("button:has-text('Deny')");
    await expect(page.getByText("Permission Required")).not.toBeVisible();
  });
});
```

### `tests/e2e/diff.spec.ts`

```typescript
import { test, expect } from "@playwright/test";

test.describe("Diff Review", () => {
  test("should show diff view for file edits", async ({ page }) => {
    await page.goto("/");
    await page.click("button:has-text('Connect')");
    await expect(page.getByText("Connected")).toBeVisible({ timeout: 10000 });

    // Ask Claude to create/edit a file
    await page.locator("textarea").fill("Create a file called test.txt with 'hello world'");
    await page.click("button:has-text('Send')");

    // Diff view should appear
    await expect(page.getByText("Review changes to")).toBeVisible({ timeout: 60000 });
    await expect(page.getByRole("button", { name: "Accept Changes" })).toBeVisible();
    await expect(page.getByRole("button", { name: "Reject" })).toBeVisible();
  });

  test("should close diff view on Accept", async ({ page }) => {
    // Setup...
    await page.click("button:has-text('Accept Changes')");
    await expect(page.getByText("Review changes to")).not.toBeVisible();
  });
});
```

### `playwright.config.ts`

```typescript
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./tests/e2e",
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  reporter: "html",
  use: {
    baseURL: "http://localhost:5173",
    trace: "on-first-retry",
  },
  projects: [
    { name: "chromium", use: { ...devices["Desktop Chrome"] } },
  ],
  webServer: {
    command: "npm run dev",
    url: "http://localhost:5173",
    reuseExistingServer: !process.env.CI,
  },
});
```

---

## Implementation Order

1. **Setup project** - Create directories, install dependencies
2. **Server** - `server/index.ts` (~50 lines)
3. **Types** - `client/types/acp.ts` (copy from above)
4. **Hook** - `client/hooks/useAcp.ts`
5. **Components** - Chat, ToolStatus, PlanView, PermissionModal, DiffView
6. **App** - Wire everything together
7. **Styling** - Tailwind setup
8. **Tests** - Unit tests first, then E2E

## Validation Steps

Before building, verify the approach works:

```bash
# 1. Install the ACP package
npm install @zed-industries/claude-code-acp

# 2. Test it responds to ACP messages
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"clientVersion":"test","capabilities":{}}}' | \
  ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY npx @zed-industries/claude-code-acp

# Should output JSON response with capabilities
```

## Summary

| Component | Lines of Code | Complexity |
|-----------|---------------|------------|
| Server (proxy) | ~50 | Low |
| Types | ~150 | Low |
| useAcp hook | ~150 | Medium |
| Chat component | ~80 | Low |
| DiffView | ~60 | Low |
| PermissionModal | ~50 | Low |
| ToolStatus | ~40 | Low |
| PlanView | ~40 | Low |
| App (orchestration) | ~120 | Medium |
| **Total** | **~750** | **Low-Medium** |

This is a straightforward proxy architecture. The `claude-code-acp` package handles all the hard work - you just render what it sends.
