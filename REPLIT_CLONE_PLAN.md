# Replit Clone Implementation Plan

A full-stack cloud IDE powered by Claude Agents SDK, running on EKS with Browserbase for live previews.

## High-Level System Architecture

```
                                    +---------------------------+
                                    |      Load Balancer        |
                                    |        (AWS ALB)          |
                                    +-------------+-------------+
                                                  |
                    +-----------------------------+-----------------------------+
                    |                             |                             |
          +---------v---------+        +----------v----------+       +----------v----------+
          |   Frontend CDN    |        |    API Gateway      |       |  WebSocket Gateway  |
          |   (CloudFront)    |        |    (Kong/AWS)       |       |    (Socket.io)      |
          +-------------------+        +----------+----------+       +----------+----------+
                    |                             |                             |
                    |              +--------------+--------------+              |
                    |              |              |              |              |
                    |    +---------v----+ +------v-------+ +----v---------+    |
                    |    | Auth Service | | Project Svc  | | Collab Svc   |    |
                    |    | (JWT/OAuth)  | | (CRUD)       | | (CRDT Sync)  |    |
                    |    +------+-------+ +------+-------+ +------+-------+    |
                    |           |               |                |             |
                    |           +-------+-------+-------+--------+             |
                    |                   |               |                      |
                    |          +--------v-------+  +----v----+                 |
                    |          |  PostgreSQL    |  | Redis   |                 |
                    |          |  (RDS)         |  | Cluster |                 |
                    |          +----------------+  +---------+                 |
                    |                                                          |
          +---------v--------------------+---------------------------+---------v---------+
          |                              |                           |                   |
    +-----v------+            +----------v----------+       +--------v--------+          |
    | Claude     |            | Execution           |       | Browserbase     |          |
    | Agent      |            | Orchestrator        |       | Integration     |          |
    | Service    |            | Service             |       | Service         |          |
    +-----+------+            +----------+----------+       +--------+--------+          |
          |                              |                           |                   |
          |                   +----------v----------+                |                   |
          |                   |   EKS Cluster       |                |                   |
          |                   |  +---------------+  |                |                   |
          |                   |  | Runtime Pods  |  |                |                   |
          |                   |  | (Isolated)    |  |                |                   |
          |                   |  +---------------+  |                |                   |
          |                   +---------------------+                |                   |
          |                              |                           |                   |
          +------------------------------+---------------------------+-------------------+
                                         |
                              +----------v----------+
                              |   Object Storage    |
                              |   (S3)              |
                              +---------------------+
```

## Core Technologies

| Technology | Purpose |
|------------|---------|
| **Claude Agents SDK** | AI-powered code generation, debugging, and assistance |
| **EKS (Kubernetes)** | Isolated container execution for user code |
| **Browserbase** | Headless browser automation for live web app previews |
| **Next.js 14** | Frontend web IDE application |
| **Monaco Editor** | Code editing with syntax highlighting and IntelliSense |
| **Yjs (CRDT)** | Real-time collaborative editing |
| **PostgreSQL** | Primary database for users, projects, files |
| **Redis** | Caching, pub/sub for collaboration scaling |

---

## Component Breakdown

### Frontend Application (Next.js 14+)

| Component | Responsibility |
|-----------|----------------|
| **Monaco Editor Integration** | Code editing with syntax highlighting, IntelliSense |
| **File Tree Component** | Hierarchical file/folder navigation, CRUD operations |
| **Terminal Emulator** | xterm.js-based terminal connected to execution pods |
| **Preview Panel** | Browserbase integration for live previews |
| **AI Assistant Panel** | Chat interface for Claude agent interactions |
| **Collaboration UI** | User presence, cursor awareness |
| **Project Dashboard** | Project listing, creation, forking, sharing |

### Backend Services

| Service | Responsibility |
|---------|----------------|
| **Authentication Service** | User registration, JWT/OAuth, session management |
| **Project Service** | Project CRUD, file system, versioning, sharing |
| **Collaboration Service** | WebSocket management, CRDT sync, presence |
| **Claude Agent Service** | Agent orchestration, tool management, context |
| **Execution Orchestrator** | Kubernetes pod lifecycle, terminal I/O routing |
| **Browserbase Service** | Browser session management, preview streaming |

---

## API Design

### REST Endpoints

```
# Authentication
POST   /api/v1/auth/register
POST   /api/v1/auth/login
POST   /api/v1/auth/oauth/:provider

# Projects
GET    /api/v1/projects
POST   /api/v1/projects
GET    /api/v1/projects/:id
PUT    /api/v1/projects/:id
POST   /api/v1/projects/:id/fork

# Files
GET    /api/v1/projects/:id/files
GET    /api/v1/projects/:id/files/:path
PUT    /api/v1/projects/:id/files/:path

# Execution
POST   /api/v1/projects/:id/runtime
DELETE /api/v1/projects/:id/runtime
POST   /api/v1/projects/:id/runtime/exec

# AI Agent
POST   /api/v1/projects/:id/agent/message
GET    /api/v1/projects/:id/agent/history

# Preview
POST   /api/v1/projects/:id/preview
GET    /api/v1/projects/:id/preview/url
```

### WebSocket Events

```typescript
// Collaboration
'join-project'    { projectId, userId }
'yjs-update'      { update: Uint8Array }
'cursor-update'   { position, selection }
'presence'        { users: User[] }

// Terminal
'terminal-input'  { projectId, data: string }
'terminal-output' { data: string }

// AI Agent
'agent-message'   { projectId, message: string }
'agent-chunk'     { text: string }
'agent-tool-use'  { tool, input, output }
```

---

## Database Schema

```sql
-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255),
    name VARCHAR(255) NOT NULL,
    avatar_url TEXT,
    auth_provider VARCHAR(50) DEFAULT 'email',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Projects
CREATE TABLE projects (
    id UUID PRIMARY KEY,
    owner_id UUID REFERENCES users(id),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL,
    description TEXT,
    language VARCHAR(50) NOT NULL,
    is_public BOOLEAN DEFAULT FALSE,
    forked_from UUID REFERENCES projects(id),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(owner_id, slug)
);

-- Virtual File System
CREATE TABLE files (
    id UUID PRIMARY KEY,
    project_id UUID REFERENCES projects(id),
    path VARCHAR(1024) NOT NULL,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(20) NOT NULL,  -- 'file' or 'directory'
    content TEXT,
    s3_key VARCHAR(512),  -- For large files
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(project_id, path)
);

-- Agent Sessions
CREATE TABLE agent_sessions (
    id UUID PRIMARY KEY,
    project_id UUID REFERENCES projects(id),
    user_id UUID REFERENCES users(id),
    conversation_history JSONB DEFAULT '[]',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Runtimes
CREATE TABLE runtimes (
    id UUID PRIMARY KEY,
    project_id UUID REFERENCES projects(id),
    pod_name VARCHAR(255),
    status VARCHAR(50) DEFAULT 'pending',
    language VARCHAR(50) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## Infrastructure Requirements

### AWS Services

| Component | Service | Specification |
|-----------|---------|---------------|
| **Compute (Services)** | ECS Fargate or EKS | 2-4 services, auto-scaling |
| **Compute (Runtimes)** | EKS | Dedicated node group, c5.xlarge+ |
| **Database** | RDS PostgreSQL | db.r6g.large, Multi-AZ |
| **Cache/Pub-Sub** | ElastiCache Redis | cache.r6g.large, cluster mode |
| **Object Storage** | S3 | Standard tier |
| **CDN** | CloudFront | Global edge locations |
| **Load Balancer** | ALB | Application Load Balancer |

### EKS Runtime Pod Template

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: runtime-{project-id}
  namespace: user-runtimes
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
  - name: runtime
    image: replit-clone/runtime-{language}:latest
    resources:
      limits:
        memory: "1Gi"
        cpu: "1000m"
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
    ports:
    - containerPort: 3000  # Web preview
    - containerPort: 4000  # Terminal PTY
```

### Estimated Monthly Costs

| Service | Estimated Cost |
|---------|---------------|
| Claude API | $3,000-$5,000 |
| Browserbase | $500-$1,000 |
| AWS EKS | $500-$2,000 |
| RDS PostgreSQL | $300-$500 |
| ElastiCache Redis | $200-$400 |
| S3 + CloudFront | $100-$200 |
| **Total** | **$4,600-$9,100/month** |

---

## Implementation Phases

### Phase 1: Foundation (Weeks 1-4)
- Initialize monorepo (Turborepo)
- Set up CI/CD pipelines
- Provision AWS infrastructure (Terraform)
- Configure EKS cluster
- Implement Authentication Service
- Implement Project Service

**Deliverable:** Users can sign up and create/manage projects via API

### Phase 2: Web IDE Core (Weeks 5-8)
- Set up Next.js 14 application
- Integrate Monaco Editor
- Build file tree component
- Connect editor to Project Service
- Implement file save/load

**Deliverable:** Functional web IDE for editing code

### Phase 3: Code Execution (Weeks 9-12)
- Build runtime container images (Node, Python, TypeScript)
- Implement Execution Orchestrator Service
- Set up pod security policies
- Integrate xterm.js terminal
- Implement WebSocket terminal proxy

**Deliverable:** Users can run code and interact via terminal

### Phase 4: AI Agent Integration (Weeks 13-16)
- Set up Claude Agent SDK integration
- Implement agent session management
- Define custom tools:
  - `read_file`: Read project files
  - `write_file`: Modify project files
  - `run_command`: Execute in terminal
  - `search_code`: Search project
- Build AI assistant chat panel
- Implement streaming responses

**Deliverable:** Users can chat with AI that reads, writes, and runs code

### Phase 5: Real-Time Collaboration (Weeks 17-20)
- Set up Yjs server infrastructure
- Implement Collaboration Service with Socket.io
- Configure Redis pub/sub for scaling
- Integrate y-monaco for editor binding
- Add cursor awareness and presence

**Deliverable:** Multiple users can edit simultaneously

### Phase 6: Live Preview (Weeks 21-24)
- Implement Browserbase Integration Service
- Set up session management
- Build preview panel component
- Add hot reload integration
- Implement console log forwarding

**Deliverable:** Users see live preview of web applications

### Phase 7: Polish and Launch (Weeks 25-28)
- Implement caching strategies
- Add rate limiting
- Set up monitoring and alerting
- Performance optimization
- Onboarding flow
- Template gallery

**Deliverable:** Production-ready application

---

## Key Technical Decisions

### 1. CRDT (Yjs) for Collaboration
- Excellent Monaco Editor bindings
- Better offline support
- Well-maintained library

### 2. Pod-per-Project Execution
- Maximum isolation between users
- Predictable resource allocation
- Simpler security model
- Mitigate cold starts with pod pre-warming pool

### 3. Hybrid File Storage
- PostgreSQL for structure and small files
- S3 for large files (>1MB)
- Content hashing for deduplication

### 4. Claude Agent SDK
- Battle-tested agent loop
- Built-in tool management
- Context compaction features

### 5. Browserbase for Previews
- Managed infrastructure
- Built-in stealth features
- Scales with usage

---

## Directory Structure

```
/replit-clone
├── apps/
│   ├── web/                      # Next.js frontend
│   │   ├── app/                  # App Router pages
│   │   ├── components/
│   │   │   ├── editor/           # Monaco editor
│   │   │   ├── terminal/         # xterm.js
│   │   │   ├── file-tree/
│   │   │   ├── preview/
│   │   │   └── ai-assistant/
│   │   ├── hooks/
│   │   ├── lib/
│   │   └── stores/
│   │
│   └── api-gateway/
│
├── services/
│   ├── auth/
│   ├── projects/
│   ├── collaboration/
│   ├── agent/
│   ├── execution/
│   └── preview/
│
├── packages/
│   ├── shared/                   # Shared types
│   ├── database/                 # DB client and migrations
│   └── ui/                       # Shared UI components
│
├── infrastructure/
│   ├── terraform/                # AWS IaC
│   ├── kubernetes/
│   │   ├── base/
│   │   ├── overlays/
│   │   └── runtime-images/
│   └── scripts/
│
└── docs/
```

---

## Critical First Files to Create

1. **`/apps/web/components/editor/MonacoEditor.tsx`** - Core editor with Yjs binding
2. **`/services/agent/src/agentService.ts`** - Claude Agent SDK orchestration
3. **`/services/execution/src/podManager.ts`** - Kubernetes pod lifecycle
4. **`/services/collaboration/src/yjsProvider.ts`** - Yjs WebSocket provider
5. **`/infrastructure/kubernetes/runtime-images/Dockerfile.node`** - Base runtime image

---

## Security Considerations

- JWT with short expiry (15 min) + refresh tokens
- Kubernetes Pod Security Standards (Restricted)
- Network policies preventing cross-pod communication
- Encryption at rest and in transit
- Rate limiting and abuse protection
- Storage and execution quotas
