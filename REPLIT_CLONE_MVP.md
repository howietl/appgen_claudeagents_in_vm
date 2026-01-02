# Replit Clone MVP - Simplified Architecture

**Core Insight:** Claude IS the developer. Users don't need an IDE - they just describe what they want, and Claude builds it.

## Simplified Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           USER'S BROWSER                                │
│  ┌─────────────────────────────────┐  ┌─────────────────────────────┐  │
│  │      Chat Interface             │  │     Preview Iframe          │  │
│  │  "Build me a todo app..."       │  │   (ngrok public URL)        │  │
│  │                                 │  │                             │  │
│  │  [Claude]: Creating files...    │  │   ┌─────────────────────┐   │  │
│  │  [Claude]: Running npm start... │  │   │  Live App Preview   │   │  │
│  │  [Claude]: ✓ App running at...  │  │   │                     │   │  │
│  └─────────────────────────────────┘  └───┴─────────────────────┴───┘  │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ WebSocket (SSE)
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         ORCHESTRATOR SERVER                             │
│                         (Next.js API Routes)                            │
│  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐   │
│  │ Session Manager   │  │ Message Router    │  │ VM Pool Manager   │   │
│  │ (simple auth)     │  │ (WebSocket hub)   │  │ (EKS controller)  │   │
│  └───────────────────┘  └───────────────────┘  └───────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ kubectl / k8s API
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           EKS CLUSTER                                   │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    BUILDER VM POD (per session)                  │   │
│  │  ┌─────────────────────────────────────────────────────────┐    │   │
│  │  │              CLAUDE AGENT (Agent SDK)                    │    │   │
│  │  │                                                          │    │   │
│  │  │  MCP Tools:                                              │    │   │
│  │  │  ├── filesystem (read/write/glob)                        │    │   │
│  │  │  ├── bash (run commands, npm, etc.)                      │    │   │
│  │  │  ├── browserbase (preview own work)  ◄── SELF-PREVIEW    │    │   │
│  │  │  └── emit_message (send updates to user)                 │    │   │
│  │  └─────────────────────────────────────────────────────────┘    │   │
│  │                          │                                       │   │
│  │  ┌───────────────────────┼───────────────────────────────────┐  │   │
│  │  │    WORKSPACE          │                                    │  │   │
│  │  │  /workspace/          ▼                                    │  │   │
│  │  │  ├── src/         ┌──────────┐      ┌──────────────────┐  │  │   │
│  │  │  ├── package.json │ Dev      │      │ ngrok / cloudflare│  │  │   │
│  │  │  └── ...          │ Server   │◄────►│ tunnel           │  │  │   │
│  │  │                   │ :3000    │      │ (public URL)     │  │  │   │
│  │  │                   └──────────┘      └────────┬─────────┘  │  │   │
│  │  └──────────────────────────────────────────────┼────────────┘  │   │
│  └─────────────────────────────────────────────────┼───────────────┘   │
└────────────────────────────────────────────────────┼───────────────────┘
                                                     │
                                                     ▼
                                            Public Internet
                                         (user can view app)
```

## Key Simplifications from Full Plan

| Full Plan | MVP |
|-----------|-----|
| Complex IDE with Monaco Editor | Just a chat interface |
| File tree UI, tabs, settings | Claude manages files directly |
| Multi-user collaboration (Yjs/CRDT) | Single user per session |
| Complex auth (OAuth, JWT refresh) | Simple session token |
| Separate microservices | Single Next.js app |
| PostgreSQL + Redis | SQLite or simple file storage |
| Preview panel with streaming | Just an iframe to ngrok URL |
| 7 implementation phases | 3 phases, ~2 weeks |

## Core Components

### 1. Frontend (Minimal Next.js)

```
/app
├── page.tsx              # Landing/login
├── workspace/[id]/
│   └── page.tsx          # Chat + Preview layout
├── api/
│   ├── sessions/         # Create/manage sessions
│   └── ws/               # WebSocket endpoint
└── components/
    ├── ChatPanel.tsx     # Message display + input
    └── PreviewFrame.tsx  # Iframe with ngrok URL
```

**That's it.** No Monaco, no file tree, no terminal emulator.

### 2. Orchestrator Server (Next.js API)

```typescript
// Simplified session flow
interface Session {
  id: string;
  userId: string;
  podName: string;
  podIP: string;
  ngrokUrl: string | null;
  status: 'starting' | 'ready' | 'stopped';
  createdAt: Date;
}

// API Routes
POST /api/sessions          // Create new builder VM
GET  /api/sessions/:id      // Get session status + ngrok URL
DELETE /api/sessions/:id    // Terminate VM
WS   /api/sessions/:id/ws   // Message stream from VM
```

### 3. Builder VM Pod (The Heart of MVP)

A single container running:
- **Claude Agent SDK** with custom MCP tools
- **Node.js/Python/etc runtime**
- **ngrok or cloudflared** for tunneling

```typescript
// agent/index.ts - Main agent entry point
import { Agent } from '@anthropic-ai/claude-agent-sdk';
import { filesystemTools } from './tools/filesystem';
import { bashTools } from './tools/bash';
import { browserbaseTool } from './tools/browserbase';
import { emitMessageTool } from './tools/emit-message';

const agent = new Agent({
  model: 'claude-sonnet-4-20250514',
  tools: [
    ...filesystemTools,    // Read, write, glob files
    ...bashTools,          // Run shell commands
    browserbaseTool,       // Preview via headless browser
    emitMessageTool,       // Send updates to user
  ],
  systemPrompt: `You are a software developer building applications.
You have access to a workspace at /workspace.
When you make changes, verify them using the browserbase tool.
Use emit_message to keep the user updated on progress.`
});
```

## MCP Tools for Claude Agent

### 1. Filesystem Tools (Standard)
```typescript
// Read, write, list files in /workspace
```

### 2. Bash Tool (Standard)
```typescript
// Run npm install, npm start, etc.
```

### 3. Browserbase Tool (Critical for Self-Verification)
```typescript
// tools/browserbase.ts
import Browserbase from '@anthropic-ai/browserbase';

export const browserbaseTool = {
  name: 'preview_app',
  description: 'Take a screenshot of the running app to verify it works. Returns a base64 image.',
  input_schema: {
    type: 'object',
    properties: {
      url: {
        type: 'string',
        description: 'URL to preview (use the ngrok URL)'
      },
      action: {
        type: 'string',
        enum: ['screenshot', 'click', 'type', 'scroll'],
        description: 'Action to perform'
      },
      selector: {
        type: 'string',
        description: 'CSS selector for click/type actions'
      },
      text: {
        type: 'string',
        description: 'Text to type (for type action)'
      }
    },
    required: ['url']
  },
  async execute({ url, action = 'screenshot', selector, text }) {
    const browser = await Browserbase.connect();
    const page = await browser.newPage();
    await page.goto(url);

    if (action === 'click' && selector) {
      await page.click(selector);
    } else if (action === 'type' && selector && text) {
      await page.type(selector, text);
    }

    const screenshot = await page.screenshot({ encoding: 'base64' });
    await browser.close();

    return {
      type: 'image',
      data: screenshot,
      mediaType: 'image/png'
    };
  }
};
```

### 4. Emit Message Tool (User Communication)
```typescript
// tools/emit-message.ts
export const emitMessageTool = {
  name: 'emit_message',
  description: 'Send a message to the user. Use for status updates, questions, or sharing the preview URL.',
  input_schema: {
    type: 'object',
    properties: {
      type: {
        type: 'string',
        enum: ['status', 'url', 'error', 'question', 'complete'],
      },
      message: { type: 'string' },
      url: { type: 'string' }  // For 'url' type
    },
    required: ['type', 'message']
  },
  async execute({ type, message, url }) {
    // Send via WebSocket to orchestrator
    await fetch(process.env.ORCHESTRATOR_CALLBACK_URL, {
      method: 'POST',
      body: JSON.stringify({ type, message, url, sessionId: process.env.SESSION_ID })
    });
    return { success: true };
  }
};
```

## Message Flow: VM → User

```
┌─────────────────┐     HTTP POST      ┌─────────────────┐
│  Builder VM     │ ──────────────────►│  Orchestrator   │
│  (emit_message) │                    │  /api/callback  │
└─────────────────┘                    └────────┬────────┘
                                                │
                                                │ WebSocket push
                                                ▼
                                       ┌─────────────────┐
                                       │  User Browser   │
                                       │  (ChatPanel)    │
                                       └─────────────────┘
```

## Exposing Apps with ngrok/Cloudflare

Inside each builder VM pod:

```bash
# Option 1: ngrok (simpler, has free tier)
ngrok http 3000 --log=stdout > /tmp/ngrok.log &
# Parse public URL from log, emit to user

# Option 2: cloudflared (no account needed for quick tunnels)
cloudflared tunnel --url http://localhost:3000
# Gives you a *.trycloudflare.com URL
```

```typescript
// agent/tools/expose.ts
export const exposeTool = {
  name: 'expose_app',
  description: 'Expose the running app to the internet and get a public URL',
  input_schema: {
    type: 'object',
    properties: {
      port: { type: 'number', default: 3000 }
    }
  },
  async execute({ port = 3000 }) {
    // Start cloudflared tunnel
    const proc = spawn('cloudflared', ['tunnel', '--url', `http://localhost:${port}`]);

    // Parse URL from stderr (cloudflared outputs there)
    const url = await new Promise((resolve) => {
      proc.stderr.on('data', (data) => {
        const match = data.toString().match(/https:\/\/[^\s]+\.trycloudflare\.com/);
        if (match) resolve(match[0]);
      });
    });

    // Emit URL to user
    await emitMessage({ type: 'url', message: 'App is live!', url });

    return { url };
  }
};
```

## Kubernetes Pod Spec (Builder VM)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: builder-${SESSION_ID}
  namespace: builders
spec:
  containers:
  - name: builder
    image: appgen/builder:latest
    env:
    - name: SESSION_ID
      value: "${SESSION_ID}"
    - name: ORCHESTRATOR_CALLBACK_URL
      value: "https://orchestrator.example.com/api/callback"
    - name: ANTHROPIC_API_KEY
      valueFrom:
        secretKeyRef:
          name: anthropic-secrets
          key: api-key
    - name: BROWSERBASE_API_KEY
      valueFrom:
        secretKeyRef:
          name: browserbase-secrets
          key: api-key
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "2Gi"
        cpu: "2000m"
    ports:
    - containerPort: 3000  # Dev server
    - containerPort: 8080  # Agent HTTP interface
    volumeMounts:
    - name: workspace
      mountPath: /workspace
  volumes:
  - name: workspace
    emptyDir:
      sizeLimit: "1Gi"
```

## Builder Container Dockerfile

```dockerfile
FROM node:20-slim

# Install common tools
RUN apt-get update && apt-get install -y \
    git curl python3 python3-pip \
    && rm -rf /var/lib/apt/lists/*

# Install cloudflared for tunneling
RUN curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 \
    -o /usr/local/bin/cloudflared && chmod +x /usr/local/bin/cloudflared

# Install agent
WORKDIR /agent
COPY package*.json ./
RUN npm install
COPY . .

# Workspace for user projects
RUN mkdir -p /workspace
WORKDIR /workspace

# Start agent
CMD ["node", "/agent/index.js"]
```

## Implementation Phases

### Phase 1: Core Loop (3-5 days)
- [ ] Set up Next.js app with chat UI
- [ ] Create builder container with Claude Agent SDK
- [ ] Implement basic MCP tools (filesystem, bash)
- [ ] Add emit_message tool for VM → user communication
- [ ] Local docker-compose for testing

**Deliverable:** Chat with Claude agent that can create and run code locally

### Phase 2: Preview & Tunnel (3-5 days)
- [ ] Integrate Browserbase MCP tool
- [ ] Add cloudflared/ngrok tunnel tool
- [ ] Implement preview iframe in frontend
- [ ] Add URL sharing when app is exposed

**Deliverable:** Claude can verify its own work and share live URLs

### Phase 3: EKS Deployment (3-5 days)
- [ ] Create EKS cluster with Terraform
- [ ] Build and push builder container image
- [ ] Implement pod lifecycle management in orchestrator
- [ ] Add session cleanup and timeouts
- [ ] Basic monitoring/logging

**Deliverable:** MVP deployed and usable

## Directory Structure (MVP)

```
/appgen-mvp
├── apps/
│   └── web/                      # Next.js frontend + API
│       ├── app/
│       │   ├── page.tsx          # Landing page
│       │   ├── workspace/[id]/
│       │   │   └── page.tsx      # Chat + Preview
│       │   └── api/
│       │       ├── sessions/     # Session CRUD
│       │       └── callback/     # VM message receiver
│       └── components/
│           ├── ChatPanel.tsx
│           └── PreviewFrame.tsx
│
├── builder/                      # Builder VM container
│   ├── Dockerfile
│   ├── package.json
│   ├── index.ts                  # Agent entry point
│   └── tools/
│       ├── filesystem.ts
│       ├── bash.ts
│       ├── browserbase.ts
│       ├── expose.ts
│       └── emit-message.ts
│
├── infrastructure/
│   ├── terraform/                # EKS setup
│   └── k8s/
│       └── builder-pod.yaml
│
└── docker-compose.yaml           # Local dev
```

## Cost Estimate (MVP)

| Service | Monthly Cost |
|---------|-------------|
| Claude API (moderate usage) | $500-1,000 |
| Browserbase (basic tier) | $100-200 |
| EKS (small cluster) | $150-300 |
| Cloudflare Tunnels | Free |
| **Total MVP** | **~$750-1,500/month** |

Much cheaper than the full plan's $4,600-9,100/month!

---

## Example User Flow

1. **User:** "Build me a todo app with React"

2. **Claude (in VM):**
   - Creates React project with Vite
   - Writes components (TodoList, TodoItem, AddTodo)
   - Runs `npm install && npm run dev`
   - Exposes via cloudflared → gets URL
   - **Uses Browserbase to screenshot and verify UI works**
   - Emits: "✓ App running at https://xyz.trycloudflare.com"

3. **User sees:** Chat messages + live preview iframe

4. **User:** "Add a dark mode toggle"

5. **Claude (in VM):**
   - Edits App.tsx to add theme context
   - Adds toggle button
   - **Uses Browserbase to verify dark mode works**
   - Emits: "✓ Dark mode added, try clicking the toggle!"

---

## What's NOT in MVP (Future Features)

- User accounts / persistent projects
- Multiple files visible in UI
- Real-time code editing
- Collaboration
- Custom domains
- Database integrations
- Production deployment (just dev server)
