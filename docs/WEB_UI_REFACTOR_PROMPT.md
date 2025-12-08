# Claude Code Web UI Refactoring Prompt

Use this prompt to refactor the `claude-code-acp` repository into a self-hosted web UI that provides a VS Code-like experience for Claude Code.

---

## PROMPT START

You are tasked with refactoring the `claude-code-acp` repository (an Agent Client Protocol adapter for Claude Code) into a **self-hosted web application** that provides a native IDE-like experience similar to the Claude Code VS Code extension.

## Project Overview

**Current State**: The repository implements an ACP (Agent Client Protocol) adapter that enables Claude Code to work with editors like Zed. It uses stdin/stdout JSON-RPC communication.

**Target State**: A web application with:
- WebSocket-based bidirectional streaming between browser and server
- React frontend with Monaco Editor for code editing and diff review
- Full Claude Code functionality including file operations, terminal, and tool permissions
- Real-time streaming of Claude's responses and tool operations

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              BROWSER                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │   Chat UI    │  │ Monaco Editor│  │  Diff View   │  │  Todo/Plan   │    │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘    │
│                              │                                               │
│                              │ React State + Hooks                           │
│                              ▼                                               │
│                     ┌─────────────────┐                                      │
│                     │ WebSocket Client│                                      │
│                     └────────┬────────┘                                      │
└──────────────────────────────┼──────────────────────────────────────────────┘
                               │ ws://localhost:3000
                               │ JSON-RPC 2.0
┌──────────────────────────────┼──────────────────────────────────────────────┐
│                     ┌────────▼────────┐           SERVER                     │
│                     │ WebSocket Server│                                      │
│                     └────────┬────────┘                                      │
│                              │                                               │
│  ┌───────────────────────────┼───────────────────────────────────────────┐  │
│  │                   ┌───────▼───────┐                                    │  │
│  │                   │ Session Manager│  (manages multiple sessions)      │  │
│  │                   └───────┬───────┘                                    │  │
│  │                           │                                            │  │
│  │           ┌───────────────┼───────────────┐                           │  │
│  │           ▼               ▼               ▼                           │  │
│  │   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                  │  │
│  │   │ClaudeAcpAgent│ │ClaudeAcpAgent│ │ClaudeAcpAgent│  (per session)   │  │
│  │   └──────┬───────┘ └──────────────┘ └──────────────┘                  │  │
│  │          │                                                             │  │
│  │          │ REUSED FROM EXISTING CODEBASE                              │  │
│  │          ▼                                                             │  │
│  │   ┌──────────────┐      ┌──────────────┐      ┌──────────────┐        │  │
│  │   │  MCP Server  │◄────►│    tools.ts   │◄────►│   utils.ts   │        │  │
│  │   └──────┬───────┘      └──────────────┘      └──────────────┘        │  │
│  │          │                                                             │  │
│  └──────────┼─────────────────────────────────────────────────────────────┘  │
│             │                                                                │
│             │ Claude Agent SDK                                               │
│             ▼                                                                │
│      ┌──────────────┐                                                        │
│      │ Claude Code  │                                                        │
│      │     CLI      │                                                        │
│      └──────────────┘                                                        │
└──────────────────────────────────────────────────────────────────────────────┘
```

## Directory Structure (Target)

```
claude-code-acp/
├── src/
│   ├── server/                    # NEW: Server-side code
│   │   ├── web-server.ts          # NEW: Express + WebSocket server
│   │   ├── session-manager.ts     # NEW: Manages browser sessions
│   │   ├── ws-transport.ts        # NEW: WebSocket transport layer
│   │   └── file-api.ts            # NEW: File system REST API
│   │
│   ├── client/                    # NEW: React frontend
│   │   ├── index.html
│   │   ├── main.tsx
│   │   ├── App.tsx
│   │   ├── components/
│   │   │   ├── Chat.tsx           # Chat message list and input
│   │   │   ├── Editor.tsx         # Monaco editor wrapper
│   │   │   ├── DiffView.tsx       # Diff review with accept/reject
│   │   │   ├── FileTree.tsx       # File explorer sidebar
│   │   │   ├── ToolProgress.tsx   # Current tool operation display
│   │   │   ├── PlanView.tsx       # Todo/plan list
│   │   │   ├── PermissionModal.tsx # Tool permission approval
│   │   │   └── Terminal.tsx       # Terminal output display
│   │   ├── hooks/
│   │   │   ├── useWebSocket.ts    # WebSocket connection hook
│   │   │   ├── useSession.ts      # Session management hook
│   │   │   └── useFileSystem.ts   # File operations hook
│   │   ├── types/
│   │   │   └── index.ts           # Shared TypeScript types
│   │   └── styles/
│   │       └── globals.css
│   │
│   ├── shared/                    # NEW: Shared types between client/server
│   │   └── protocol.ts            # WebSocket message types
│   │
│   ├── acp-agent.ts               # EXISTING: Keep, minor modifications
│   ├── mcp-server.ts              # EXISTING: Keep unchanged
│   ├── tools.ts                   # EXISTING: Keep unchanged
│   ├── utils.ts                   # EXISTING: Keep, add new utilities
│   ├── index.ts                   # EXISTING: Keep for CLI mode
│   └── lib.ts                     # EXISTING: Keep for library exports
│
├── tests/
│   ├── unit/
│   │   ├── session-manager.test.ts
│   │   ├── ws-transport.test.ts
│   │   └── protocol.test.ts
│   ├── integration/
│   │   ├── websocket.test.ts
│   │   ├── file-api.test.ts
│   │   └── claude-agent.test.ts
│   └── e2e/
│       ├── playwright.config.ts
│       ├── chat-flow.spec.ts
│       ├── diff-review.spec.ts
│       └── permission-flow.spec.ts
│
├── package.json
├── tsconfig.json
├── tsconfig.server.json           # NEW: Server TypeScript config
├── vite.config.ts                 # NEW: Vite config for client
├── vitest.config.ts               # EXISTING: Update for new tests
├── playwright.config.ts           # NEW: E2E test config
└── README.md
```

## Implementation Tasks

### Phase 1: Shared Protocol Types

Create `src/shared/protocol.ts` with all message types:

```typescript
// WebSocket JSON-RPC message types

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

// Client -> Server methods
export type ClientMethod =
  | "initialize"
  | "newSession"
  | "prompt"
  | "cancel"
  | "respondToPermission"
  | "respondToDiff"
  | "setModel"
  | "getModels"
  | "getSlashCommands";

// Server -> Client notifications
export type ServerNotification =
  | TextNotification
  | ToolUpdateNotification
  | PlanNotification
  | PermissionRequestNotification
  | DiffRequestNotification
  | TerminalOutputNotification
  | ErrorNotification
  | DoneNotification;

export interface TextNotification {
  type: "text";
  text: string;
  isThinking?: boolean;
}

export interface ToolUpdateNotification {
  type: "toolUpdate";
  toolUseId: string;
  title: string;
  kind: "think" | "read" | "edit" | "execute" | "search" | "fetch" | "other";
  content?: string;
  locations?: Array<{ path: string; lineStart: number; lineEnd: number }>;
  status: "running" | "completed" | "failed";
}

export interface PlanNotification {
  type: "plan";
  entries: Array<{
    content: string;
    status: "pending" | "in_progress" | "completed";
    activeForm: string;
  }>;
}

export interface PermissionRequestNotification {
  type: "permissionRequest";
  requestId: string;
  tool: string;
  input: Record<string, unknown>;
  description: string;
}

export interface DiffRequestNotification {
  type: "diffRequest";
  requestId: string;
  path: string;
  oldContent: string;
  newContent: string;
  description: string;
}

export interface TerminalOutputNotification {
  type: "terminalOutput";
  shellId: string;
  output: string;
  isBackground: boolean;
}

export interface ErrorNotification {
  type: "error";
  message: string;
  details?: string;
}

export interface DoneNotification {
  type: "done";
  sessionId: string;
  costUsd?: number;
  durationMs?: number;
  numTurns?: number;
}

// Session state
export interface SessionState {
  id: string;
  claudeSessionId: string;
  model: string;
  cwd: string;
  createdAt: Date;
  lastActivityAt: Date;
}

// Initialize params
export interface InitializeParams {
  cwd: string;
  capabilities?: {
    filesystem?: { read: boolean; write: boolean; roots: string[] };
    terminal?: { run: boolean; background: boolean };
  };
}

// Prompt params
export interface PromptParams {
  sessionId: string;
  content: string;
  attachments?: Array<{
    type: "file" | "image" | "selection";
    path?: string;
    content?: string;
    mimeType?: string;
  }>;
}
```

### Phase 2: WebSocket Transport Layer

Create `src/server/ws-transport.ts`:

```typescript
import { WebSocket } from "ws";
import { EventEmitter } from "events";
import type { JsonRpcRequest, JsonRpcResponse, JsonRpcNotification } from "../shared/protocol.js";

export class WebSocketTransport extends EventEmitter {
  private ws: WebSocket;
  private pendingRequests = new Map<string | number, {
    resolve: (result: unknown) => void;
    reject: (error: Error) => void;
  }>();

  constructor(ws: WebSocket) {
    super();
    this.ws = ws;
    this.setupListeners();
  }

  private setupListeners() {
    this.ws.on("message", (data) => {
      try {
        const message = JSON.parse(data.toString());
        this.handleMessage(message);
      } catch (err) {
        this.emit("error", new Error(`Failed to parse message: ${err}`));
      }
    });

    this.ws.on("close", () => this.emit("close"));
    this.ws.on("error", (err) => this.emit("error", err));
  }

  private handleMessage(message: JsonRpcRequest | JsonRpcResponse) {
    if ("method" in message) {
      // It's a request from the client
      this.emit("request", message);
    } else if ("id" in message) {
      // It's a response to our request
      const pending = this.pendingRequests.get(message.id);
      if (pending) {
        this.pendingRequests.delete(message.id);
        if (message.error) {
          pending.reject(new Error(message.error.message));
        } else {
          pending.resolve(message.result);
        }
      }
    }
  }

  sendNotification(method: string, params: unknown): void {
    if (this.ws.readyState === WebSocket.OPEN) {
      const notification: JsonRpcNotification = {
        jsonrpc: "2.0",
        method,
        params,
      };
      this.ws.send(JSON.stringify(notification));
    }
  }

  sendResponse(id: string | number, result?: unknown, error?: { code: number; message: string }): void {
    if (this.ws.readyState === WebSocket.OPEN) {
      const response: JsonRpcResponse = {
        jsonrpc: "2.0",
        id,
        ...(error ? { error } : { result }),
      };
      this.ws.send(JSON.stringify(response));
    }
  }

  async sendRequest(method: string, params?: unknown): Promise<unknown> {
    return new Promise((resolve, reject) => {
      const id = crypto.randomUUID();
      this.pendingRequests.set(id, { resolve, reject });

      const request: JsonRpcRequest = {
        jsonrpc: "2.0",
        id,
        method,
        params,
      };
      this.ws.send(JSON.stringify(request));

      // Timeout after 30 seconds
      setTimeout(() => {
        if (this.pendingRequests.has(id)) {
          this.pendingRequests.delete(id);
          reject(new Error("Request timeout"));
        }
      }, 30000);
    });
  }

  close(): void {
    this.ws.close();
  }

  get isOpen(): boolean {
    return this.ws.readyState === WebSocket.OPEN;
  }
}
```

### Phase 3: Session Manager

Create `src/server/session-manager.ts`:

```typescript
import { ClaudeAcpAgent } from "../acp-agent.js";
import { createMcpServer } from "../mcp-server.js";
import { WebSocketTransport } from "./ws-transport.js";
import type {
  SessionState,
  InitializeParams,
  PromptParams,
  ServerNotification,
} from "../shared/protocol.js";

export interface SessionManagerOptions {
  maxSessions?: number;
  sessionTimeout?: number; // ms
}

export class SessionManager {
  private sessions = new Map<string, Session>();
  private options: Required<SessionManagerOptions>;

  constructor(options: SessionManagerOptions = {}) {
    this.options = {
      maxSessions: options.maxSessions ?? 100,
      sessionTimeout: options.sessionTimeout ?? 30 * 60 * 1000, // 30 minutes
    };

    // Cleanup expired sessions periodically
    setInterval(() => this.cleanupExpiredSessions(), 60000);
  }

  async createSession(
    transport: WebSocketTransport,
    params: InitializeParams
  ): Promise<Session> {
    if (this.sessions.size >= this.options.maxSessions) {
      throw new Error("Maximum sessions reached");
    }

    const session = new Session(transport, params);
    await session.initialize();

    this.sessions.set(session.id, session);

    transport.on("close", () => {
      this.destroySession(session.id);
    });

    return session;
  }

  getSession(id: string): Session | undefined {
    return this.sessions.get(id);
  }

  async destroySession(id: string): Promise<void> {
    const session = this.sessions.get(id);
    if (session) {
      await session.destroy();
      this.sessions.delete(id);
    }
  }

  private cleanupExpiredSessions(): void {
    const now = Date.now();
    for (const [id, session] of this.sessions) {
      if (now - session.lastActivityAt.getTime() > this.options.sessionTimeout) {
        this.destroySession(id);
      }
    }
  }
}

export class Session {
  readonly id: string;
  readonly cwd: string;
  readonly createdAt: Date;
  lastActivityAt: Date;

  private transport: WebSocketTransport;
  private agent: ClaudeAcpAgent;
  private claudeSessionId: string | null = null;
  private pendingPermissions = new Map<string, (approved: boolean) => void>();
  private pendingDiffs = new Map<string, (approved: boolean) => void>();

  constructor(transport: WebSocketTransport, params: InitializeParams) {
    this.id = crypto.randomUUID();
    this.cwd = params.cwd;
    this.createdAt = new Date();
    this.lastActivityAt = new Date();
    this.transport = transport;

    this.agent = new ClaudeAcpAgent({
      log: (...args) => console.error("[Agent]", ...args),
    });

    this.setupTransportHandlers();
  }

  private setupTransportHandlers(): void {
    this.transport.on("request", async (request) => {
      this.lastActivityAt = new Date();

      try {
        const result = await this.handleRequest(request.method, request.params);
        this.transport.sendResponse(request.id, result);
      } catch (err) {
        this.transport.sendResponse(request.id, undefined, {
          code: -32000,
          message: err instanceof Error ? err.message : "Unknown error",
        });
      }
    });
  }

  async initialize(): Promise<{ sessionId: string; capabilities: unknown }> {
    const capabilities = await this.agent.initialize({
      clientVersion: "claude-code-web/1.0.0",
      capabilities: {
        filesystem: { read: true, write: true, roots: [this.cwd] },
        terminal: { run: true, background: true },
      },
    });

    return { sessionId: this.id, capabilities };
  }

  private async handleRequest(method: string, params: unknown): Promise<unknown> {
    switch (method) {
      case "newSession":
        return this.handleNewSession();

      case "prompt":
        return this.handlePrompt(params as PromptParams);

      case "cancel":
        return this.handleCancel();

      case "respondToPermission":
        return this.handlePermissionResponse(params as { requestId: string; approved: boolean });

      case "respondToDiff":
        return this.handleDiffResponse(params as { requestId: string; approved: boolean });

      case "getModels":
        return this.agent.getAvailableModels?.() ?? [];

      case "getSlashCommands":
        return this.agent.getAvailableSlashCommands?.() ?? [];

      default:
        throw new Error(`Unknown method: ${method}`);
    }
  }

  private async handleNewSession(): Promise<{ claudeSessionId: string }> {
    const result = await this.agent.newSession({ mcpServers: [] });
    this.claudeSessionId = result.sessionId;
    return { claudeSessionId: result.sessionId };
  }

  private async handlePrompt(params: PromptParams): Promise<{ done: boolean }> {
    if (!this.claudeSessionId) {
      throw new Error("No active Claude session");
    }

    // Convert prompt to Claude format
    const messages = [{
      role: "human" as const,
      content: [{ type: "text" as const, text: params.content }],
    }];

    // Add file attachments
    if (params.attachments) {
      for (const attachment of params.attachments) {
        if (attachment.type === "file" && attachment.path) {
          messages[0].content.push({
            type: "text" as const,
            text: `@${attachment.path}`,
          });
        }
      }
    }

    // Stream responses
    const generator = this.agent.prompt({
      sessionId: this.claudeSessionId,
      messages,
    });

    for await (const notification of generator) {
      const transformed = this.transformNotification(notification);
      if (transformed) {
        this.transport.sendNotification("notification", transformed);
      }
    }

    this.transport.sendNotification("notification", { type: "done", sessionId: this.id });
    return { done: true };
  }

  private transformNotification(notification: unknown): ServerNotification | null {
    // Transform ACP notifications to our web protocol format
    const n = notification as Record<string, unknown>;

    if (n.text) {
      return { type: "text", text: n.text as string };
    }

    if (n.toolUpdate) {
      const tu = n.toolUpdate as Record<string, unknown>;
      return {
        type: "toolUpdate",
        toolUseId: tu.id as string,
        title: tu.title as string,
        kind: tu.kind as ToolUpdateNotification["kind"],
        content: tu.content as string | undefined,
        locations: tu.locations as ToolUpdateNotification["locations"],
        status: "running",
      };
    }

    if (n.plan) {
      return {
        type: "plan",
        entries: n.plan as PlanNotification["entries"],
      };
    }

    // Handle permission requests by intercepting canUseTool
    if (n.permissionRequest) {
      const pr = n.permissionRequest as Record<string, unknown>;
      return {
        type: "permissionRequest",
        requestId: pr.id as string,
        tool: pr.tool as string,
        input: pr.input as Record<string, unknown>,
        description: pr.description as string,
      };
    }

    return null;
  }

  private async handleCancel(): Promise<{ cancelled: boolean }> {
    if (this.claudeSessionId) {
      await this.agent.cancel({ sessionId: this.claudeSessionId });
    }
    return { cancelled: true };
  }

  private handlePermissionResponse(params: { requestId: string; approved: boolean }): { resolved: boolean } {
    const callback = this.pendingPermissions.get(params.requestId);
    if (callback) {
      callback(params.approved);
      this.pendingPermissions.delete(params.requestId);
      return { resolved: true };
    }
    return { resolved: false };
  }

  private handleDiffResponse(params: { requestId: string; approved: boolean }): { resolved: boolean } {
    const callback = this.pendingDiffs.get(params.requestId);
    if (callback) {
      callback(params.approved);
      this.pendingDiffs.delete(params.requestId);
      return { resolved: true };
    }
    return { resolved: false };
  }

  // Called by modified agent when permission is needed
  async requestPermission(tool: string, input: Record<string, unknown>): Promise<boolean> {
    return new Promise((resolve) => {
      const requestId = crypto.randomUUID();
      this.pendingPermissions.set(requestId, resolve);

      this.transport.sendNotification("notification", {
        type: "permissionRequest",
        requestId,
        tool,
        input,
        description: `Allow ${tool}?`,
      });
    });
  }

  // Called for diff review
  async requestDiffApproval(path: string, oldContent: string, newContent: string): Promise<boolean> {
    return new Promise((resolve) => {
      const requestId = crypto.randomUUID();
      this.pendingDiffs.set(requestId, resolve);

      this.transport.sendNotification("notification", {
        type: "diffRequest",
        requestId,
        path,
        oldContent,
        newContent,
        description: `Review changes to ${path}`,
      });
    });
  }

  async destroy(): Promise<void> {
    // Reject all pending requests
    for (const [, callback] of this.pendingPermissions) {
      callback(false);
    }
    for (const [, callback] of this.pendingDiffs) {
      callback(false);
    }

    this.pendingPermissions.clear();
    this.pendingDiffs.clear();
    this.transport.close();
  }

  getState(): SessionState {
    return {
      id: this.id,
      claudeSessionId: this.claudeSessionId ?? "",
      model: "claude-sonnet-4-20250514",
      cwd: this.cwd,
      createdAt: this.createdAt,
      lastActivityAt: this.lastActivityAt,
    };
  }
}
```

### Phase 4: Web Server

Create `src/server/web-server.ts`:

```typescript
import express from "express";
import { createServer } from "http";
import { WebSocketServer } from "ws";
import path from "path";
import { fileURLToPath } from "url";
import { SessionManager } from "./session-manager.js";
import { WebSocketTransport } from "./ws-transport.js";
import { createFileApi } from "./file-api.js";
import { loadManagedSettings, applyEnvironmentSettings } from "../utils.js";

const __dirname = path.dirname(fileURLToPath(import.meta.url));

export interface WebServerOptions {
  port?: number;
  allowedRoots?: string[];
  staticDir?: string;
}

export async function startWebServer(options: WebServerOptions = {}): Promise<void> {
  const {
    port = parseInt(process.env.PORT || "3000", 10),
    allowedRoots = [process.cwd()],
    staticDir = path.join(__dirname, "../../dist/client"),
  } = options;

  // Load Claude Code settings
  const settings = loadManagedSettings();
  if (settings) {
    applyEnvironmentSettings(settings);
  }

  const app = express();
  app.use(express.json());

  // File API
  app.use("/api", createFileApi(allowedRoots));

  // Health check
  app.get("/api/health", (req, res) => {
    res.json({ status: "ok", timestamp: new Date().toISOString() });
  });

  // Serve static files (React app)
  app.use(express.static(staticDir));

  // SPA fallback
  app.get("*", (req, res) => {
    res.sendFile(path.join(staticDir, "index.html"));
  });

  const httpServer = createServer(app);
  const wss = new WebSocketServer({ server: httpServer });
  const sessionManager = new SessionManager();

  wss.on("connection", async (ws, req) => {
    console.error(`[WebSocket] New connection from ${req.socket.remoteAddress}`);

    const transport = new WebSocketTransport(ws);
    let session: Awaited<ReturnType<typeof sessionManager.createSession>> | null = null;

    transport.on("request", async (request) => {
      // Handle initialize specially - creates the session
      if (request.method === "initialize") {
        try {
          session = await sessionManager.createSession(transport, request.params);
          transport.sendResponse(request.id, {
            sessionId: session.id,
            capabilities: { filesystem: true, terminal: true },
          });
        } catch (err) {
          transport.sendResponse(request.id, undefined, {
            code: -32000,
            message: err instanceof Error ? err.message : "Failed to initialize",
          });
        }
      }
      // All other requests are handled by the session
    });

    transport.on("error", (err) => {
      console.error("[WebSocket] Error:", err);
    });

    transport.on("close", () => {
      console.error("[WebSocket] Connection closed");
    });
  });

  httpServer.listen(port, () => {
    console.error(`
╔═══════════════════════════════════════════════════════╗
║           Claude Code Web UI                          ║
║                                                       ║
║   Server running at: http://localhost:${port}            ║
║                                                       ║
║   Allowed roots: ${allowedRoots.join(", ")}
╚═══════════════════════════════════════════════════════╝
    `);
  });
}

// CLI entry point
if (import.meta.url === `file://${process.argv[1]}`) {
  startWebServer({
    port: parseInt(process.env.PORT || "3000", 10),
    allowedRoots: process.env.ALLOWED_ROOTS?.split(",") || [process.cwd()],
  });
}
```

### Phase 5: File API

Create `src/server/file-api.ts`:

```typescript
import { Router } from "express";
import fs from "fs/promises";
import path from "path";

export function createFileApi(allowedRoots: string[]): Router {
  const router = Router();

  function isPathAllowed(filePath: string): boolean {
    const resolved = path.resolve(filePath);
    return allowedRoots.some((root) => resolved.startsWith(path.resolve(root)));
  }

  // Read file
  router.get("/files", async (req, res) => {
    const filePath = req.query.path as string;

    if (!filePath) {
      return res.status(400).json({ error: "Path required" });
    }

    if (!isPathAllowed(filePath)) {
      return res.status(403).json({ error: "Access denied" });
    }

    try {
      const stat = await fs.stat(filePath);
      if (stat.isDirectory()) {
        return res.status(400).json({ error: "Path is a directory" });
      }

      const content = await fs.readFile(filePath, "utf-8");
      res.type("text/plain").send(content);
    } catch (err) {
      if ((err as NodeJS.ErrnoException).code === "ENOENT") {
        return res.status(404).json({ error: "File not found" });
      }
      res.status(500).json({ error: "Failed to read file" });
    }
  });

  // Write file
  router.post("/files", async (req, res) => {
    const { path: filePath, content } = req.body;

    if (!filePath || content === undefined) {
      return res.status(400).json({ error: "Path and content required" });
    }

    if (!isPathAllowed(filePath)) {
      return res.status(403).json({ error: "Access denied" });
    }

    try {
      await fs.writeFile(filePath, content, "utf-8");
      res.json({ success: true });
    } catch (err) {
      res.status(500).json({ error: "Failed to write file" });
    }
  });

  // List directory
  router.get("/files/list", async (req, res) => {
    const dirPath = req.query.path as string || allowedRoots[0];

    if (!isPathAllowed(dirPath)) {
      return res.status(403).json({ error: "Access denied" });
    }

    try {
      const entries = await fs.readdir(dirPath, { withFileTypes: true });
      const result = entries
        .filter((e) => !e.name.startsWith("."))
        .map((e) => ({
          name: e.name,
          path: path.join(dirPath, e.name),
          isDirectory: e.isDirectory(),
        }))
        .sort((a, b) => {
          if (a.isDirectory !== b.isDirectory) {
            return a.isDirectory ? -1 : 1;
          }
          return a.name.localeCompare(b.name);
        });

      res.json(result);
    } catch (err) {
      res.status(500).json({ error: "Failed to list directory" });
    }
  });

  // Get file tree (recursive)
  router.get("/files/tree", async (req, res) => {
    const rootPath = req.query.path as string || allowedRoots[0];
    const maxDepth = parseInt(req.query.depth as string || "3", 10);

    if (!isPathAllowed(rootPath)) {
      return res.status(403).json({ error: "Access denied" });
    }

    async function buildTree(dir: string, depth: number): Promise<any[]> {
      if (depth > maxDepth) return [];

      const entries = await fs.readdir(dir, { withFileTypes: true });
      const result = [];

      for (const entry of entries) {
        if (entry.name.startsWith(".") || entry.name === "node_modules") {
          continue;
        }

        const fullPath = path.join(dir, entry.name);
        const node: any = {
          name: entry.name,
          path: fullPath,
          isDirectory: entry.isDirectory(),
        };

        if (entry.isDirectory()) {
          node.children = await buildTree(fullPath, depth + 1);
        }

        result.push(node);
      }

      return result.sort((a, b) => {
        if (a.isDirectory !== b.isDirectory) {
          return a.isDirectory ? -1 : 1;
        }
        return a.name.localeCompare(b.name);
      });
    }

    try {
      const tree = await buildTree(rootPath, 0);
      res.json({ root: rootPath, children: tree });
    } catch (err) {
      res.status(500).json({ error: "Failed to build tree" });
    }
  });

  return router;
}
```

### Phase 6: React Frontend

Create the React application with these key components:

#### `src/client/App.tsx`
Main application component that orchestrates all UI components and manages global state.

#### `src/client/hooks/useWebSocket.ts`
```typescript
import { useCallback, useEffect, useRef, useState } from "react";
import type { JsonRpcRequest, ServerNotification } from "../../shared/protocol";

export interface UseWebSocketOptions {
  url: string;
  onNotification: (notification: ServerNotification) => void;
  onError?: (error: Error) => void;
  onClose?: () => void;
}

export function useWebSocket(options: UseWebSocketOptions) {
  const { url, onNotification, onError, onClose } = options;
  const [isConnected, setIsConnected] = useState(false);
  const wsRef = useRef<WebSocket | null>(null);
  const requestIdRef = useRef(0);
  const pendingRequestsRef = useRef<Map<number, { resolve: Function; reject: Function }>>(new Map());

  const connect = useCallback(() => {
    if (wsRef.current?.readyState === WebSocket.OPEN) return;

    const ws = new WebSocket(url);
    wsRef.current = ws;

    ws.onopen = () => setIsConnected(true);

    ws.onclose = () => {
      setIsConnected(false);
      onClose?.();
    };

    ws.onerror = () => {
      onError?.(new Error("WebSocket error"));
    };

    ws.onmessage = (event) => {
      try {
        const message = JSON.parse(event.data);

        if (message.method === "notification") {
          onNotification(message.params);
        } else if (message.id !== undefined) {
          const pending = pendingRequestsRef.current.get(message.id);
          if (pending) {
            pendingRequestsRef.current.delete(message.id);
            if (message.error) {
              pending.reject(new Error(message.error.message));
            } else {
              pending.resolve(message.result);
            }
          }
        }
      } catch (err) {
        console.error("Failed to parse message:", err);
      }
    };
  }, [url, onNotification, onError, onClose]);

  const disconnect = useCallback(() => {
    wsRef.current?.close();
    wsRef.current = null;
  }, []);

  const sendRequest = useCallback(<T = unknown>(method: string, params?: unknown): Promise<T> => {
    return new Promise((resolve, reject) => {
      if (!wsRef.current || wsRef.current.readyState !== WebSocket.OPEN) {
        reject(new Error("Not connected"));
        return;
      }

      const id = ++requestIdRef.current;
      pendingRequestsRef.current.set(id, { resolve, reject });

      const request: JsonRpcRequest = {
        jsonrpc: "2.0",
        id,
        method,
        params,
      };

      wsRef.current.send(JSON.stringify(request));

      // Timeout
      setTimeout(() => {
        if (pendingRequestsRef.current.has(id)) {
          pendingRequestsRef.current.delete(id);
          reject(new Error("Request timeout"));
        }
      }, 60000);
    });
  }, []);

  useEffect(() => {
    return () => {
      disconnect();
    };
  }, [disconnect]);

  return {
    isConnected,
    connect,
    disconnect,
    sendRequest,
  };
}
```

#### `src/client/components/DiffView.tsx`
```typescript
import { DiffEditor } from "@monaco-editor/react";

interface DiffViewProps {
  path: string;
  oldContent: string;
  newContent: string;
  onAccept: () => void;
  onReject: () => void;
}

export function DiffView({ path, oldContent, newContent, onAccept, onReject }: DiffViewProps) {
  const language = getLanguageFromPath(path);

  return (
    <div className="flex flex-col h-full">
      <div className="flex items-center justify-between p-2 bg-gray-800 border-b border-gray-700">
        <span className="text-sm font-mono">{path}</span>
        <div className="flex gap-2">
          <button
            onClick={onAccept}
            className="px-4 py-1.5 bg-green-600 hover:bg-green-700 rounded text-sm font-medium transition-colors"
          >
            Accept Changes
          </button>
          <button
            onClick={onReject}
            className="px-4 py-1.5 bg-red-600 hover:bg-red-700 rounded text-sm font-medium transition-colors"
          >
            Reject
          </button>
        </div>
      </div>
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
          }}
        />
      </div>
    </div>
  );
}

function getLanguageFromPath(path: string): string {
  const ext = path.split(".").pop()?.toLowerCase() || "";
  const languageMap: Record<string, string> = {
    ts: "typescript",
    tsx: "typescript",
    js: "javascript",
    jsx: "javascript",
    py: "python",
    rs: "rust",
    go: "go",
    java: "java",
    c: "c",
    cpp: "cpp",
    h: "c",
    hpp: "cpp",
    css: "css",
    scss: "scss",
    html: "html",
    json: "json",
    yaml: "yaml",
    yml: "yaml",
    md: "markdown",
    sql: "sql",
    sh: "shell",
    bash: "shell",
  };
  return languageMap[ext] || "plaintext";
}
```

#### `src/client/components/PermissionModal.tsx`
```typescript
interface PermissionModalProps {
  tool: string;
  input: Record<string, unknown>;
  description: string;
  onAllow: () => void;
  onDeny: () => void;
}

export function PermissionModal({ tool, input, description, onAllow, onDeny }: PermissionModalProps) {
  return (
    <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
      <div className="bg-gray-800 rounded-lg shadow-xl max-w-lg w-full mx-4">
        <div className="p-4 border-b border-gray-700">
          <h2 className="text-lg font-semibold">Permission Required</h2>
        </div>

        <div className="p-4">
          <p className="text-gray-300 mb-4">{description}</p>

          <div className="bg-gray-900 rounded p-3 mb-4">
            <div className="text-sm text-gray-400 mb-1">Tool</div>
            <div className="font-mono text-yellow-400">{tool}</div>
          </div>

          <div className="bg-gray-900 rounded p-3">
            <div className="text-sm text-gray-400 mb-1">Arguments</div>
            <pre className="text-xs overflow-auto max-h-40">
              {JSON.stringify(input, null, 2)}
            </pre>
          </div>
        </div>

        <div className="p-4 border-t border-gray-700 flex justify-end gap-3">
          <button
            onClick={onDeny}
            className="px-4 py-2 bg-gray-700 hover:bg-gray-600 rounded transition-colors"
          >
            Deny
          </button>
          <button
            onClick={onAllow}
            className="px-4 py-2 bg-blue-600 hover:bg-blue-700 rounded transition-colors"
          >
            Allow
          </button>
        </div>
      </div>
    </div>
  );
}
```

## Testing Requirements

### Unit Tests

Create `tests/unit/session-manager.test.ts`:
```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";
import { SessionManager, Session } from "../../src/server/session-manager";
import { WebSocketTransport } from "../../src/server/ws-transport";

describe("SessionManager", () => {
  let sessionManager: SessionManager;

  beforeEach(() => {
    sessionManager = new SessionManager({ maxSessions: 5 });
  });

  describe("createSession", () => {
    it("should create a new session with valid params", async () => {
      const mockTransport = createMockTransport();
      const session = await sessionManager.createSession(mockTransport, {
        cwd: "/tmp/test",
      });

      expect(session.id).toBeDefined();
      expect(session.cwd).toBe("/tmp/test");
    });

    it("should reject when max sessions reached", async () => {
      const transports = Array(5).fill(null).map(() => createMockTransport());

      for (const transport of transports) {
        await sessionManager.createSession(transport, { cwd: "/tmp" });
      }

      await expect(
        sessionManager.createSession(createMockTransport(), { cwd: "/tmp" })
      ).rejects.toThrow("Maximum sessions reached");
    });
  });

  describe("destroySession", () => {
    it("should clean up session resources", async () => {
      const mockTransport = createMockTransport();
      const session = await sessionManager.createSession(mockTransport, {
        cwd: "/tmp/test",
      });

      await sessionManager.destroySession(session.id);

      expect(sessionManager.getSession(session.id)).toBeUndefined();
    });
  });
});

function createMockTransport(): WebSocketTransport {
  return {
    on: vi.fn(),
    sendNotification: vi.fn(),
    sendResponse: vi.fn(),
    close: vi.fn(),
    isOpen: true,
  } as unknown as WebSocketTransport;
}
```

Create `tests/unit/ws-transport.test.ts`:
```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";
import { WebSocket } from "ws";
import { WebSocketTransport } from "../../src/server/ws-transport";

describe("WebSocketTransport", () => {
  let mockWs: WebSocket;
  let transport: WebSocketTransport;
  let messageHandler: (data: string) => void;

  beforeEach(() => {
    mockWs = {
      readyState: WebSocket.OPEN,
      on: vi.fn((event, handler) => {
        if (event === "message") {
          messageHandler = handler;
        }
      }),
      send: vi.fn(),
      close: vi.fn(),
    } as unknown as WebSocket;

    transport = new WebSocketTransport(mockWs);
  });

  describe("sendNotification", () => {
    it("should send JSON-RPC notification", () => {
      transport.sendNotification("test", { foo: "bar" });

      expect(mockWs.send).toHaveBeenCalledWith(
        JSON.stringify({
          jsonrpc: "2.0",
          method: "test",
          params: { foo: "bar" },
        })
      );
    });

    it("should not send when socket is closed", () => {
      (mockWs as any).readyState = WebSocket.CLOSED;

      transport.sendNotification("test", {});

      expect(mockWs.send).not.toHaveBeenCalled();
    });
  });

  describe("message handling", () => {
    it("should emit request event for incoming requests", () => {
      const requestHandler = vi.fn();
      transport.on("request", requestHandler);

      messageHandler(JSON.stringify({
        jsonrpc: "2.0",
        id: 1,
        method: "test",
        params: {},
      }));

      expect(requestHandler).toHaveBeenCalledWith(expect.objectContaining({
        method: "test",
      }));
    });
  });
});
```

Create `tests/unit/protocol.test.ts`:
```typescript
import { describe, it, expect } from "vitest";
import type {
  TextNotification,
  ToolUpdateNotification,
  PermissionRequestNotification,
} from "../../src/shared/protocol";

describe("Protocol Types", () => {
  it("should validate TextNotification structure", () => {
    const notification: TextNotification = {
      type: "text",
      text: "Hello world",
      isThinking: false,
    };

    expect(notification.type).toBe("text");
    expect(notification.text).toBeDefined();
  });

  it("should validate ToolUpdateNotification structure", () => {
    const notification: ToolUpdateNotification = {
      type: "toolUpdate",
      toolUseId: "123",
      title: "Reading file",
      kind: "read",
      content: "file content",
      status: "running",
    };

    expect(notification.kind).toBe("read");
    expect(notification.status).toBe("running");
  });

  it("should validate PermissionRequestNotification structure", () => {
    const notification: PermissionRequestNotification = {
      type: "permissionRequest",
      requestId: "abc",
      tool: "Bash",
      input: { command: "ls -la" },
      description: "Execute shell command",
    };

    expect(notification.tool).toBe("Bash");
    expect(notification.input).toHaveProperty("command");
  });
});
```

### Integration Tests

Create `tests/integration/websocket.test.ts`:
```typescript
import { describe, it, expect, beforeAll, afterAll } from "vitest";
import { WebSocket } from "ws";
import { startWebServer } from "../../src/server/web-server";

describe("WebSocket Integration", () => {
  let server: any;
  const port = 3099;

  beforeAll(async () => {
    server = await startWebServer({ port, allowedRoots: ["/tmp"] });
  });

  afterAll(() => {
    server?.close();
  });

  it("should accept WebSocket connections", async () => {
    const ws = new WebSocket(`ws://localhost:${port}`);

    await new Promise<void>((resolve, reject) => {
      ws.on("open", () => resolve());
      ws.on("error", reject);
      setTimeout(() => reject(new Error("Connection timeout")), 5000);
    });

    expect(ws.readyState).toBe(WebSocket.OPEN);
    ws.close();
  });

  it("should handle initialize request", async () => {
    const ws = new WebSocket(`ws://localhost:${port}`);

    await new Promise<void>((resolve) => ws.on("open", resolve));

    const response = await new Promise<any>((resolve) => {
      ws.on("message", (data) => {
        resolve(JSON.parse(data.toString()));
      });

      ws.send(JSON.stringify({
        jsonrpc: "2.0",
        id: 1,
        method: "initialize",
        params: { cwd: "/tmp" },
      }));
    });

    expect(response.id).toBe(1);
    expect(response.result).toHaveProperty("sessionId");

    ws.close();
  });
});
```

Create `tests/integration/file-api.test.ts`:
```typescript
import { describe, it, expect, beforeAll, afterAll, beforeEach } from "vitest";
import fs from "fs/promises";
import path from "path";
import request from "supertest";
import express from "express";
import { createFileApi } from "../../src/server/file-api";

describe("File API Integration", () => {
  const testDir = "/tmp/claude-code-web-test";
  let app: express.Application;

  beforeAll(async () => {
    await fs.mkdir(testDir, { recursive: true });
    app = express();
    app.use(express.json());
    app.use("/api", createFileApi([testDir]));
  });

  afterAll(async () => {
    await fs.rm(testDir, { recursive: true, force: true });
  });

  beforeEach(async () => {
    // Clean test directory
    const files = await fs.readdir(testDir);
    for (const file of files) {
      await fs.rm(path.join(testDir, file), { recursive: true });
    }
  });

  describe("GET /api/files", () => {
    it("should read existing file", async () => {
      const testFile = path.join(testDir, "test.txt");
      await fs.writeFile(testFile, "Hello World");

      const response = await request(app)
        .get("/api/files")
        .query({ path: testFile });

      expect(response.status).toBe(200);
      expect(response.text).toBe("Hello World");
    });

    it("should return 404 for non-existent file", async () => {
      const response = await request(app)
        .get("/api/files")
        .query({ path: path.join(testDir, "nonexistent.txt") });

      expect(response.status).toBe(404);
    });

    it("should return 403 for paths outside allowed roots", async () => {
      const response = await request(app)
        .get("/api/files")
        .query({ path: "/etc/passwd" });

      expect(response.status).toBe(403);
    });
  });

  describe("POST /api/files", () => {
    it("should write file", async () => {
      const testFile = path.join(testDir, "new.txt");

      const response = await request(app)
        .post("/api/files")
        .send({ path: testFile, content: "New content" });

      expect(response.status).toBe(200);

      const content = await fs.readFile(testFile, "utf-8");
      expect(content).toBe("New content");
    });
  });

  describe("GET /api/files/list", () => {
    it("should list directory contents", async () => {
      await fs.writeFile(path.join(testDir, "file1.txt"), "");
      await fs.writeFile(path.join(testDir, "file2.txt"), "");
      await fs.mkdir(path.join(testDir, "subdir"));

      const response = await request(app)
        .get("/api/files/list")
        .query({ path: testDir });

      expect(response.status).toBe(200);
      expect(response.body).toHaveLength(3);
      expect(response.body[0].isDirectory).toBe(true); // subdir first
    });
  });
});
```

### E2E Tests

Create `tests/e2e/playwright.config.ts`:
```typescript
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./tests/e2e",
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: "html",
  use: {
    baseURL: "http://localhost:3000",
    trace: "on-first-retry",
  },
  projects: [
    {
      name: "chromium",
      use: { ...devices["Desktop Chrome"] },
    },
  ],
  webServer: {
    command: "npm run dev",
    url: "http://localhost:3000",
    reuseExistingServer: !process.env.CI,
    timeout: 120000,
  },
});
```

Create `tests/e2e/chat-flow.spec.ts`:
```typescript
import { test, expect } from "@playwright/test";

test.describe("Chat Flow", () => {
  test.beforeEach(async ({ page }) => {
    await page.goto("/");
  });

  test("should connect to WebSocket server", async ({ page }) => {
    await page.click("button:has-text('Connect')");

    await expect(page.locator("text=Connected")).toBeVisible({ timeout: 10000 });
  });

  test("should send message and receive response", async ({ page }) => {
    await page.click("button:has-text('Connect')");
    await expect(page.locator("text=Connected")).toBeVisible({ timeout: 10000 });

    const input = page.locator("textarea[placeholder*='Ask Claude']");
    await input.fill("Hello, Claude!");
    await input.press("Enter");

    // Wait for user message to appear
    await expect(page.locator(".message:has-text('Hello, Claude!')")).toBeVisible();

    // Wait for assistant response (may take a while)
    await expect(page.locator(".message.assistant")).toBeVisible({ timeout: 60000 });
  });

  test("should display tool operations", async ({ page }) => {
    await page.click("button:has-text('Connect')");
    await expect(page.locator("text=Connected")).toBeVisible({ timeout: 10000 });

    const input = page.locator("textarea[placeholder*='Ask Claude']");
    await input.fill("List the files in the current directory");
    await input.press("Enter");

    // Should show tool operation
    await expect(page.locator("[data-testid='tool-progress']")).toBeVisible({ timeout: 30000 });
  });
});
```

Create `tests/e2e/diff-review.spec.ts`:
```typescript
import { test, expect } from "@playwright/test";

test.describe("Diff Review", () => {
  test("should show diff view when file changes are proposed", async ({ page }) => {
    await page.goto("/");
    await page.click("button:has-text('Connect')");
    await expect(page.locator("text=Connected")).toBeVisible({ timeout: 10000 });

    const input = page.locator("textarea[placeholder*='Ask Claude']");
    await input.fill("Create a file called test.txt with the content 'Hello World'");
    await input.press("Enter");

    // Wait for diff view to appear
    const diffView = page.locator("[data-testid='diff-view']");
    await expect(diffView).toBeVisible({ timeout: 60000 });

    // Should show accept/reject buttons
    await expect(page.locator("button:has-text('Accept')")).toBeVisible();
    await expect(page.locator("button:has-text('Reject')")).toBeVisible();
  });

  test("should accept changes when Accept is clicked", async ({ page }) => {
    // Setup and trigger diff view...

    await page.click("button:has-text('Accept')");

    // Diff view should close
    await expect(page.locator("[data-testid='diff-view']")).not.toBeVisible();
  });

  test("should reject changes when Reject is clicked", async ({ page }) => {
    // Setup and trigger diff view...

    await page.click("button:has-text('Reject')");

    // Diff view should close
    await expect(page.locator("[data-testid='diff-view']")).not.toBeVisible();

    // Should show rejection message in chat
    await expect(page.locator("text=rejected")).toBeVisible();
  });
});
```

Create `tests/e2e/permission-flow.spec.ts`:
```typescript
import { test, expect } from "@playwright/test";

test.describe("Permission Flow", () => {
  test("should show permission modal for dangerous operations", async ({ page }) => {
    await page.goto("/");
    await page.click("button:has-text('Connect')");
    await expect(page.locator("text=Connected")).toBeVisible({ timeout: 10000 });

    const input = page.locator("textarea[placeholder*='Ask Claude']");
    await input.fill("Run the command: rm -rf /tmp/test-dir");
    await input.press("Enter");

    // Permission modal should appear
    const modal = page.locator("[data-testid='permission-modal']");
    await expect(modal).toBeVisible({ timeout: 30000 });

    // Should show tool name and arguments
    await expect(modal.locator("text=Bash")).toBeVisible();
    await expect(modal.locator("text=rm -rf")).toBeVisible();
  });

  test("should continue execution when permission is granted", async ({ page }) => {
    // Setup and trigger permission modal...

    await page.click("button:has-text('Allow')");

    // Modal should close
    await expect(page.locator("[data-testid='permission-modal']")).not.toBeVisible();

    // Execution should continue
    await expect(page.locator("[data-testid='tool-progress']")).toBeVisible();
  });

  test("should stop execution when permission is denied", async ({ page }) => {
    // Setup and trigger permission modal...

    await page.click("button:has-text('Deny')");

    // Modal should close
    await expect(page.locator("[data-testid='permission-modal']")).not.toBeVisible();

    // Should show denial message
    await expect(page.locator("text=denied")).toBeVisible();
  });
});
```

## Configuration Files

### `vite.config.ts`
```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import path from "path";

export default defineConfig({
  plugins: [react()],
  root: "src/client",
  build: {
    outDir: "../../dist/client",
    emptyOutDir: true,
  },
  server: {
    port: 5173,
    proxy: {
      "/api": "http://localhost:3000",
      "/ws": {
        target: "ws://localhost:3000",
        ws: true,
      },
    },
  },
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "src/client"),
      "@shared": path.resolve(__dirname, "src/shared"),
    },
  },
});
```

### `tsconfig.server.json`
```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src",
    "module": "NodeNext",
    "moduleResolution": "NodeNext"
  },
  "include": ["src/server/**/*", "src/shared/**/*", "src/*.ts"],
  "exclude": ["src/client/**/*"]
}
```

### Updated `package.json`
```json
{
  "name": "claude-code-web",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "concurrently \"tsx watch src/server/web-server.ts\" \"vite\"",
    "build": "vite build && tsc -p tsconfig.server.json",
    "start": "node dist/server/web-server.js",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    "lint": "eslint src tests",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@anthropic-ai/claude-agent-sdk": "latest",
    "@modelcontextprotocol/sdk": "^1.0.0",
    "diff": "^5.0.0",
    "express": "^4.18.2",
    "ws": "^8.14.2",
    "zod": "^3.22.0"
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
    "concurrently": "^8.2.2",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "supertest": "^6.3.3",
    "tailwindcss": "^3.4.0",
    "tsx": "^4.6.2",
    "typescript": "^5.3.2",
    "vite": "^5.0.0",
    "vitest": "^1.0.0"
  }
}
```

## Implementation Order

1. **Phase 1**: Create `src/shared/protocol.ts` with all type definitions
2. **Phase 2**: Create `src/server/ws-transport.ts` - WebSocket transport layer
3. **Phase 3**: Create `src/server/session-manager.ts` - Session management
4. **Phase 4**: Create `src/server/file-api.ts` - File system REST API
5. **Phase 5**: Create `src/server/web-server.ts` - Express + WebSocket server
6. **Phase 6**: Create React frontend in `src/client/`
7. **Phase 7**: Write unit tests in `tests/unit/`
8. **Phase 8**: Write integration tests in `tests/integration/`
9. **Phase 9**: Write E2E tests in `tests/e2e/`
10. **Phase 10**: Update configuration files and documentation

## Key Modifications to Existing Files

### `src/acp-agent.ts`
- Add event emitter for permission requests to support async UI approval
- Export `ClaudeAcpAgent` class for direct instantiation (already done via lib.ts)
- No changes to core logic needed

### `src/mcp-server.ts`
- No changes needed - reuse as-is

### `src/tools.ts`
- No changes needed - reuse as-is

### `src/utils.ts`
- Add any new utility functions needed by the web server

## Success Criteria

1. **Functional Requirements**:
   - [ ] User can connect to the web UI and start a Claude Code session
   - [ ] User can send messages and receive streaming responses
   - [ ] User can review and accept/reject file diffs
   - [ ] User can approve/deny tool permission requests
   - [ ] User can view and edit files in Monaco editor
   - [ ] User can see the todo/plan list as Claude works
   - [ ] User can cancel ongoing operations

2. **Testing Requirements**:
   - [ ] Unit test coverage > 80% for server code
   - [ ] All integration tests pass
   - [ ] All E2E tests pass in CI
   - [ ] No TypeScript errors (`npm run typecheck`)
   - [ ] No ESLint errors (`npm run lint`)

3. **Performance Requirements**:
   - [ ] WebSocket connection established in < 1 second
   - [ ] Message latency < 100ms
   - [ ] UI remains responsive during long operations

## PROMPT END

---

Save this prompt and use it to guide the implementation of your web UI. The prompt is self-contained and includes all necessary details for a complete implementation.
