# ☁️ Cloud-Based SVG Animation Platform

> A browser-based vector design and animation tool — like Figma meets SVGator — where all rendering and video export runs entirely in the cloud. Users on weak devices can create high-quality animations and export them as MP4, GIF, Lottie, or Animated SVG.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.x-blue)](https://www.typescriptlang.org/)
[![Node.js](https://img.shields.io/badge/Node.js-20.x-green)](https://nodejs.org/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.29-blue)](https://kubernetes.io/)
[![pnpm](https://img.shields.io/badge/pnpm-workspace-orange)](https://pnpm.io/)

---

## 📑 Table of Contents

- [Overview](#overview)
- [Features](#features)
- [System Architecture](#system-architecture)
- [Graphics Engine](#graphics-engine)
- [Animation Engine](#animation-engine)
- [Cloud Rendering System](#cloud-rendering-system)
- [Cloud Infrastructure](#cloud-infrastructure)
- [Data Models](#data-models)
- [Performance Engineering](#performance-engineering)
- [Export Formats](#export-formats)
- [Security](#security)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Development Roadmap](#development-roadmap)
- [AI Features](#ai-features)
- [Tech Stack](#tech-stack)
- [Contributing](#contributing)

---

## Overview

This platform is a **distributed, cloud-native** SVG animation editor. Users design and animate SVG illustrations in the browser, then offload computationally expensive video export to a scalable pool of cloud render workers. No powerful local hardware required.

**Inspired by:**
| Tool | What we borrow |
|------|----------------|
| Figma | Document model, layer system, real-time collaboration |
| Adobe Illustrator | Path editing, boolean operations, fill/stroke system |
| SVGator | Timeline animation, keyframe editor, animated SVG export |
| After Effects | Track-based animation, easing curves, expression system |

---

## Features

### 🎨 Vector Editing
- Rect, ellipse, polygon, star, and freehand path tools
- Full Bézier path editor with anchor/handle manipulation
- Boolean operations (union, subtract, intersect, exclude)
- Gradient fills (linear, radial), image fills, pattern fills
- Drop shadow, blur, and blend mode effects
- Groups, frames, clipping masks, and component symbols

### 🎬 Animation
- Track-based timeline with keyframe editor
- Animatable properties: position, scale, rotation, opacity, color, path morph
- Easing presets: linear, ease-in/out, cubic-bezier, spring, bounce
- Path morphing between shapes
- Expression-based procedural animation
- Real-time preview playback in the browser

### ☁️ Cloud Rendering & Export
| Format | Engine | Notes |
|--------|--------|-------|
| **MP4** (H.264/H.265) | FFmpeg | Cloud render workers |
| **GIF** | FFmpeg (2-pass palette) | Cloud render workers |
| **WebM** (VP9) | FFmpeg | Supports transparency |
| **PNG Sequence** | Puppeteer | For video compositing |
| **Animated SVG** | Browser (CSS keyframes) | Instant, no cloud needed |
| **Lottie JSON** | Browser | Mobile app compatible |
| **SVG** | Browser | Static export |

### 👥 Collaboration
- Real-time multi-user editing via **Yjs CRDT**
- Cursor presence and live selection indicators
- Comment threads on canvas elements
- Team workspaces with owner/editor/viewer roles

### 🗂️ Project Management
- Autosave every 15 seconds
- Named version snapshots
- Asset library (images, fonts, SVG components)
- Public sharing with embed codes
- Template library

---

## System Architecture

```
┌──────────────────────────── BROWSER CLIENT ────────────────────────────────┐
│  ┌──────────────┐   ┌──────────────┐   ┌─────────────────────────────────┐  │
│  │  Vector      │   │  Animation   │   │  Preview / Playback Engine      │  │
│  │  Editor UI   │   │  Timeline UI │   │  (WebGL / Canvas 2D hybrid)     │  │
│  └──────┬───────┘   └──────┬───────┘   └─────────────────────────────────┘  │
│  ┌──────▼─────────────────▼──────────────────────────────────────────────┐  │
│  │                Client State Manager (Zustand + Immer)                  │  │
│  └──────────────────────────────────┬─────────────────────────────────────┘  │
└─────────────────────────────────────┼──────────────────────────────────────┘
                          WebSocket + REST API
┌─────────────────────────────────────▼──────────────────────────────────────┐
│                         API GATEWAY (Kong / nginx)                          │
└──────────────┬─────────────────────────────────────┬───────────────────────┘
   ┌───────────▼───────────┐                ┌────────▼──────────┐
   │  Auth + Project API   │                │  Asset API        │
   │  (Node.js services)   │                │  (Upload / CDN)   │
   └───────────┬───────────┘                └────────┬──────────┘
   ┌───────────▼─────────────────────────────────────▼──────────┐
   │         Message Queue  (BullMQ over Redis)                  │
   └─────────────────────────────┬──────────────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │   RENDER WORKER POOL    │
                    │  Puppeteer + FFmpeg     │
                    │  Kubernetes + KEDA      │
                    │  (scale-to-zero)        │
                    └─────────────────────────┘
```

### Microservices

| Service | Responsibility | Stack |
|---------|----------------|-------|
| `auth-service` | JWT, OAuth, sessions | Node.js + Passport.js |
| `project-service` | Project CRUD, versioning, collab | Node.js + PostgreSQL |
| `asset-service` | Upload, resize, CDN management | Node.js + Sharp |
| `render-service` | Export job submission & status | Node.js + BullMQ |
| `render-worker` | Headless rendering + FFmpeg encoding | Puppeteer + FFmpeg |
| `gateway` | Rate limiting, routing, auth | Kong / nginx |

### Client Architecture

The browser app is split into a strict **UI layer** (React) and **Engine layer** (pure TypeScript — zero React dependencies):

```
Browser Application
│
├── UI Layer          (React 18) — presents state, dispatches intent
├── Engine Layer      (TypeScript) — document model, renderer, animation
├── Worker Layer      (Web Workers via Comlink) — heavy path math, export
└── Communication     (REST + WebSocket) — API calls, autosave, collaboration
```

---

## Graphics Engine

### Document Model

The scene graph uses a **flat node map** (not nested objects). All nodes are stored by ID; parent/child relationships are maintained via `parentId` and `childIds` references. This makes undo/redo and CRDT sync dramatically simpler.

```typescript
interface DocumentState {
  id: string;
  nodes: Record<string, SceneNode>;   // Flat map — key design decision
  pages: string[];
  animation: AnimationTimeline;
  assets: AssetLibrary;
}
```

Supported node types: `rect` · `ellipse` · `path` · `polygon` · `star` · `text` · `image` · `group` · `frame` · `mask` · `component` · `component-instance`

### Transform System

Every node stores its transform as a **2D affine matrix** (6 values). World transforms are computed by multiplying matrices up the parent chain.

```typescript
interface Transform {
  a: number; b: number;    // Rotation / Scale X
  c: number; d: number;    // Skew / Scale Y
  tx: number; ty: number;  // Translation
}
```

### Path Editing

SVG path data is parsed into an internal `BezierPoint` representation with independent in/out handles per anchor. Operations (move anchor, move handle, toggle corner type, add/remove point) mutate this representation and serialize back to SVG path strings on commit.

Boolean operations (union, subtract, intersect, exclude) run in a **Web Worker** using **paper.js** to avoid blocking the UI.

### Rendering Pipeline

```
Scene Graph Mutation
        │
   Dirty flags set (per node)
        │
   requestAnimationFrame
        │
   ┌────▼───────────────────────────────┐
   │  1. Traverse only dirty subtrees   │
   │  2. Pull cached ImageBitmap        │
   │     if node is clean               │
   │  3. Re-render dirty nodes to       │
   │     OffscreenCanvas                │
   │  4. Composite bottom-up            │
   │  5. Apply effects (WebGL)          │
   │  6. Blit to visible <canvas>       │
   └────────────────────────────────────┘
```

> **Key principle:** The SVG DOM is used only as a transparent hit-test overlay. All pixels are rendered to `OffscreenCanvas` for performance.

---

## Animation Engine

### Timeline Data Model

Track-based animation — each track targets one property on one node using dot-path notation:

```typescript
interface AnimationTrack {
  nodeId: string;
  property: string;        // e.g. "transform.tx", "opacity", "fill.0.color.r"
  keyframes: Keyframe[];
}

interface Keyframe {
  time: number;            // ms from start
  value: number | Color | string;
  easing: EasingFunction;
}
```

### Supported Easing Functions

| Type | Parameters |
|------|-----------|
| `linear` | — |
| `ease-in` / `ease-out` / `ease-in-out` | `power: number` |
| `cubic-bezier` | `x1, y1, x2, y2` |
| `spring` | `mass, stiffness, damping` |
| `bounce` | `amplitude, period` |
| `step` | `count, direction` |

### Path Morphing

Morph between two shapes by interpolating each anchor point's position and handles. Paths with different point counts are resampled to match before morphing.

### Playback Engine

The playback engine uses `requestAnimationFrame` with precise `performance.now()` timing. It supports play, pause, seek, and loop. At each frame it evaluates all tracks and writes computed values directly to the scene graph nodes before triggering a re-render.

---

## Cloud Rendering System

### Render Flow

```
Browser                API Server                 Render Worker
   │                       │                           │
   │── POST /export ───────►│                           │
   │◄── { jobId } ─────────│── enqueue(job) ───────────►│
   │                       │                    ┌───────▼────────────────┐
   │── GET /jobs/:id ──────►│                    │ Launch Puppeteer       │
   │◄── { progress: 45% } ─│                    │ Load /render/:projectId│
   │                       │                    └───────┬────────────────┘
   │                       │                            │
   │                       │                    For each frame:
   │                       │                    seekTo(t) → screenshot.png
   │                       │                            │
   │                       │                    FFmpeg: PNGs → MP4/GIF
   │                       │                            │
   │                       │◄── complete(url) ──────────│
   │◄── { url: cdn/out.mp4}│
```

### Headless Renderer

The render worker loads a stripped-down headless version of the editor — no toolbars or UI chrome — that exposes `window.animationEngine` for programmatic frame seeking.

```typescript
for (let i = 0; i < frameCount; i++) {
  const t = startTime + (i / fps) * 1000;
  await page.evaluate((ms) => window.animationEngine.seekTo(ms), t);
  // Wait for 2 rAF cycles to ensure canvas is painted
  await page.evaluate(() =>
    new Promise(r => requestAnimationFrame(() => requestAnimationFrame(r))));
  await page.screenshot({ path: `frame_${i}.png` });
}
```

### FFmpeg Encoding

```bash
# MP4 — near-lossless H.264
ffmpeg -i frame_%06d.png -vcodec libx264 -crf 18 -preset slow \
       -pix_fmt yuv420p -movflags +faststart output.mp4

# GIF — two-pass palette for best quality
# Pass 1: generate optimal palette
ffmpeg -i frame_%06d.png -vf "palettegen=stats_mode=diff" palette.png
# Pass 2: encode with palette + dithering
ffmpeg -i frame_%06d.png -i palette.png \
       -lavfi "paletteuse=dither=bayer:bayer_scale=5" output.gif
```

### Job Queue

Jobs are queued in **BullMQ** (Redis-backed). Render workers use **KEDA** to autoscale based on queue depth — scaling to **zero pods when idle** to minimize cloud cost.

```yaml
# KEDA ScaledObject
minReplicaCount: 0        # Zero workers when no jobs
maxReplicaCount: 20       # Up to 20 parallel render workers
trigger:
  type: redis
  listLength: "3"          # 1 new worker per 3 queued jobs
```

---

## Cloud Infrastructure

### Kubernetes Layout

```
svgstudio namespace
│
├── Deployments
│   ├── project-api       (replicas: 3, HPA: 3–10)
│   ├── auth-api          (replicas: 2, HPA: 2–6)
│   ├── asset-api         (replicas: 2, HPA: 2–6)
│   └── render-worker     (replicas: 0–20, KEDA)
│
├── StatefulSets
│   ├── postgresql
│   └── redis
│
└── Services + Ingress
    └── nginx-ingress → API gateway → services
```

### Node Groups

| Node Group | Instance Type | Purpose |
|------------|--------------|---------|
| `api` | `t3.large` (2 vCPU / 8 GB) | API services, auto-scales 2–10 |
| `render` | `c6i.2xlarge` (8 vCPU / 16 GB) | Render workers, KEDA 0–50 |
| `render-gpu` *(optional)* | `g4dn.xlarge` (NVIDIA T4) | 4K H.265 encoding tier |

### GPU vs CPU Strategy

| Scenario | Strategy |
|----------|----------|
| SVG animation export (1080p) | CPU — Chromium headless |
| H.264 encoding | CPU — libx264 |
| H.265 / AV1 (4K tier) | GPU — nvenc |
| High concurrency (many jobs) | CPU — more cost-effective |

### Terraform (multi-cloud)

The infrastructure is defined in Terraform and works on AWS, GCP, or DigitalOcean by swapping the provider block. Environments (staging, production) use Terraform workspaces.

---

## Data Models

### PostgreSQL Schema (key tables)

```sql
-- Projects
CREATE TABLE projects (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID NOT NULL REFERENCES users(id),
  name          VARCHAR(255) NOT NULL,
  document_ref  VARCHAR(512),      -- S3 key (documents can be 1–50 MB)
  canvas_width  INT DEFAULT 1920,
  canvas_height INT DEFAULT 1080,
  duration_ms   INT DEFAULT 0,
  deleted_at    TIMESTAMPTZ        -- Soft delete
);

-- Version history / autosave
CREATE TABLE project_versions (
  project_id   UUID REFERENCES projects(id),
  version      INT NOT NULL,
  document_ref VARCHAR(512) NOT NULL,
  is_autosave  BOOLEAN DEFAULT TRUE,
  label        VARCHAR(255),        -- Named snapshots
  UNIQUE(project_id, version)
);

-- Render jobs
CREATE TABLE render_jobs (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID REFERENCES projects(id),
  status     VARCHAR(50) DEFAULT 'pending',
  format     VARCHAR(50) NOT NULL,
  settings   JSONB NOT NULL,
  output_url VARCHAR(512),
  progress   INT DEFAULT 0
);
```

### Document JSON

Project documents are stored as JSON in S3 (not in PostgreSQL) to keep the database lean. The document uses a flat node map:

```json
{
  "id": "proj_abc123",
  "schemaVersion": "1.0.0",
  "nodes": {
    "frame_1": {
      "id": "frame_1", "type": "frame",
      "parentId": "page_1",
      "childIds": ["group_logo", "text_title"],
      "transform": { "a":1,"b":0,"c":0,"d":1,"tx":0,"ty":0 }
    }
  },
  "animation": {
    "duration": 3000, "fps": 30, "loop": false,
    "tracks": [{
      "nodeId": "group_logo",
      "property": "transform.tx",
      "keyframes": [
        { "time": 0,    "value": -200, "easing": { "type": "ease-out", "power": 2 } },
        { "time": 600,  "value": 0,    "easing": { "type": "linear" } }
      ]
    }]
  }
}
```

---

## Performance Engineering

### Rendering Strategy

| Layer | Technology | Purpose |
|-------|-----------|---------|
| z=1 Canvas | `OffscreenCanvas` / Canvas 2D | All pixel rendering |
| z=2 SVG overlay | Transparent SVG | Hit-testing only |
| z=3 HTML | Absolute-positioned divs | Selection handles, UI chrome |

### Key Optimizations

**Dirty-rect rendering** — only nodes with changed properties are re-rendered each frame.

**Layer bitmap caching** — unchanged nodes are cached as `ImageBitmap` objects. Re-rendering a static layer costs zero GPU time.

**R-tree spatial index** — hit-testing with 1000+ objects stays O(log n).

```typescript
import RBush from 'rbush';
const tree = new RBush();
tree.insert({ minX, minY, maxX, maxY, nodeId });
const hits = tree.search({ minX: px, minY: py, maxX: px, maxY: py });
```

**Immer structural sharing** — undo/redo with 100 history entries stays memory-efficient because unchanged subtrees are shared, not cloned.

**Web Workers** — path boolean ops, auto-layout computation, and SVG/Lottie serialization run off the main thread via Comlink.

### Performance Targets

| Metric | Target |
|--------|--------|
| Editor time-to-interactive | < 2 seconds |
| Canvas render latency (60 fps) | < 16ms / frame |
| Undo / redo | < 5ms |
| Hit-test (1000 objects) | < 1ms |
| 30s 1080p MP4 render (CPU) | < 2 minutes |
| 30s 1080p MP4 render (GPU) | < 30 seconds |
| API response p95 | < 200ms |

---

## Export Formats

| Format | Where | Transparency | Use Case |
|--------|-------|-------------|----------|
| **SVG** | Browser | ✅ | Web, print, scalable static |
| **Animated SVG** | Browser | ✅ | Web animation, email |
| **Lottie JSON** | Browser | ✅ | Mobile apps (iOS/Android), web |
| **MP4** (H.264) | Cloud | ❌ | Social media, video platforms |
| **GIF** | Cloud | ✅ (1-bit) | Messaging, email |
| **WebM** (VP9) | Cloud | ✅ | Web video with alpha |
| **PNG Sequence** | Cloud | ✅ | Video compositing (Premiere, AE) |

---

## Security

### SVG Upload Sanitization
All uploaded SVGs are sanitized with **DOMPurify** before storage to remove scripts, event handlers, and external resource references (XSS, SSRF prevention).

```typescript
DOMPurify.sanitize(svg, {
  USE_PROFILES: { svg: true, svgFilters: true },
  FORBID_TAGS: ['script', 'foreignObject'],
  FORBID_ATTR: ['onload', 'onerror', 'href', 'xlink:href'],
});
```

### Asset Upload Validation
File types are detected from **magic bytes** (not extension or Content-Type). All raster images are re-encoded through **Sharp** to strip EXIF data and any embedded payloads.

### Render Worker Sandbox
- Runs as **non-root** with read-only root filesystem
- **Kubernetes NetworkPolicy** restricts egress to internal CDN only
- All external HTTP requests from Puppeteer are blocked at the `page.request` interception level

### Authorization
- JWT access tokens (15 min) + refresh token rotation
- Per-project role checks: `owner` / `editor` / `viewer`
- All S3 assets are private — served only via time-limited presigned URLs
- Rate limiting on all API endpoints

---

## Project Structure

```
svgstudio/
├── apps/
│   ├── web/                         # Main React editor application
│   │   └── src/
│   │       ├── components/
│   │       │   ├── Editor/
│   │       │   │   ├── Canvas/      # Canvas + SVG overlay layers
│   │       │   │   ├── Toolbar/     # Main + context toolbars
│   │       │   │   ├── Timeline/    # Keyframe editor UI
│   │       │   │   ├── Layers/      # Layer panel
│   │       │   │   └── Properties/  # Fill, stroke, transform panels
│   │       │   └── Export/          # Export modal + progress
│   │       ├── engine/              # Core engine (zero React deps)
│   │       │   ├── document/        # Scene graph, node factory
│   │       │   ├── renderer/        # Canvas 2D + WebGL
│   │       │   ├── animation/       # Playback, interpolation, easing
│   │       │   ├── tools/           # Select, Pen, Rect, Text, ...
│   │       │   ├── history/         # Undo/redo + operations
│   │       │   └── export/          # SVG, Lottie, AnimatedSVG exporters
│   │       ├── workers/             # Web Workers (Comlink)
│   │       ├── store/               # Zustand state stores
│   │       └── api/                 # REST + WebSocket clients
│   │
│   └── render-headless/             # Stripped editor for cloud rendering
│       └── src/
│           ├── RenderCanvas.tsx
│           └── renderBridge.ts      # Exposes window.animationEngine
│
├── services/
│   ├── auth-service/
│   ├── project-service/
│   ├── asset-service/
│   ├── render-service/              # Job queue API
│   └── render-worker/               # Puppeteer + FFmpeg worker
│       └── src/
│           ├── renderer.ts          # Headless frame capture
│           ├── encoder.ts           # FFmpeg pipeline
│           └── worker.ts            # BullMQ worker process
│
├── packages/
│   ├── shared-types/                # TypeScript types across all apps
│   └── design-system/               # Shared UI component library
│
├── infrastructure/
│   ├── terraform/                   # IaC — AWS / GCP / DigitalOcean
│   │   ├── modules/                 # eks, rds, elasticache, s3, cdn
│   │   └── environments/            # staging/, production/
│   └── kubernetes/
│       ├── deployments/
│       ├── services/
│       ├── ingress/
│       ├── keda/                    # KEDA ScaledObjects (autoscaling)
│       └── secrets/                 # Sealed Secrets
│
├── docker/
│   └── docker-compose.yml           # Full local dev stack
│
├── pnpm-workspace.yaml
└── turbo.json                       # Turborepo task graph
```

---

## Getting Started

### Prerequisites
- Node.js 20+
- pnpm 9+
- Docker + Docker Compose
- (Optional) `kubectl` + `terraform` for cloud deployment

### Local Development

```bash
# 1. Clone and install dependencies
git clone https://github.com/your-org/svgstudio.git
cd svgstudio
pnpm install

# 2. Start all backing services (PostgreSQL, Redis, MinIO)
docker compose up -d

# 3. Run database migrations
pnpm --filter project-service exec prisma migrate dev

# 4. Start all apps and services in parallel
pnpm dev
```

The editor will be available at `http://localhost:5173`.

### Environment Variables

Copy `.env.example` to `.env` in each service directory. Key variables:

```env
# project-service
DATABASE_URL=postgresql://user:pass@localhost:5432/svgstudio
REDIS_URL=redis://localhost:6379

# asset-service
S3_BUCKET=svgstudio-assets
S3_ENDPOINT=http://localhost:9000   # MinIO for local dev
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin

# render-worker
RENDER_BASE_URL=http://localhost:5174
PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium
WORKER_CONCURRENCY=2
```

### Running Tests

```bash
pnpm test              # All packages
pnpm --filter web test # Frontend only
pnpm test:e2e          # Playwright end-to-end tests
```

### Building for Production

```bash
pnpm build             # Build all apps and services

# Build and push Docker images
docker build -f docker/render-worker/Dockerfile -t registry/render-worker:latest .
docker push registry/render-worker:latest

# Deploy to Kubernetes
kubectl apply -f infrastructure/kubernetes/
```

---

## Development Roadmap

### ✅ Phase 1 — MVP (Months 1–3)
- [x] Monorepo setup (pnpm + Turborepo)
- [ ] Authentication (JWT + OAuth)
- [ ] Canvas 2D rendering engine
- [ ] Shape tools: rect, ellipse, path
- [ ] Selection + transform (move, resize, rotate)
- [ ] Layer panel + undo/redo
- [ ] Timeline + keyframe animation
- [ ] Cloud render workers (Puppeteer + FFmpeg)
- [ ] MP4 export

### 🔄 Phase 2 — Full Feature Set (Months 4–6)
- [ ] SVG path editor (bezier handles, pen tool)
- [ ] Boolean path operations
- [ ] Image import + asset library
- [ ] GIF + Animated SVG export
- [ ] Text tool + web font support
- [ ] Gradient fills + shadow/blur effects
- [ ] Groups, frames, clipping masks
- [ ] Component system (symbols)
- [ ] Lottie JSON export

### 🔜 Phase 3 — Collaboration & Scale (Months 7–9)
- [ ] Yjs real-time collaboration
- [ ] Team workspaces + permissions
- [ ] Comment system
- [ ] Kubernetes production deployment
- [ ] KEDA render autoscaling
- [ ] Multi-region CDN

### 🔜 Phase 4 — AI & Growth (Months 10–12)
- [ ] AI auto-animation from static SVG
- [ ] Text-to-SVG generation
- [ ] AI animation chat assistant
- [ ] Character rigging
- [ ] Stripe billing
- [ ] Plugin/extension API

---

## AI Features

The platform includes optional AI-powered capabilities built on top of the core animation engine:

### Auto-Animation
Analyses SVG element structure and generates a sensible animation timeline using GPT-4o with structured JSON output. Users choose a style (gentle, energetic, playful, corporate) and the AI populates keyframes.

### Smart Easing
Suggests easing functions based on the property being animated and context:
- Position entrances → spring easing
- Opacity fades-in → ease-out
- Color transitions → linear

### Animation Chat Assistant
An in-editor natural language interface:
```
User:  "Make the logo bounce when it enters the scene"
Agent: → Applies spring easing to logo Y-position entrance keyframes

User:  "Fade in the text 0.3 seconds after the logo finishes"
Agent: → Adds opacity track: t=logo_end+300 → t=logo_end+800, ease-out
```

### Character Rigging
Uses a vision model to detect body regions in SVG character illustrations and proposes a bone hierarchy. Supports inverse kinematics (IK) for natural limb movement.

---

## Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Frontend | React 18 + TypeScript | Ecosystem, concurrent rendering |
| State | Zustand + Immer | Lightweight, structural sharing for history |
| Build | Vite + Turborepo | Fast HMR, monorepo task caching |
| Canvas | Canvas 2D + OffscreenCanvas | Performance, fine-grained dirty rendering |
| Path ops | paper.js (Web Worker) | Mature boolean operations |
| Collaboration | Yjs + y-websocket | Industry-standard CRDT |
| API | Express + Fastify | Mature, flexible |
| ORM | Prisma | Type-safe, great DX |
| Database | PostgreSQL 16 | JSONB, rock-solid |
| Queue/Cache | Redis + BullMQ | Proven job queue |
| Storage | S3-compatible | AWS S3 / Cloudflare R2 / MinIO |
| Headless browser | Puppeteer + Chromium | Best SVG rendering fidelity |
| Video encoding | FFmpeg | Industry standard |
| Orchestration | Kubernetes + KEDA | Scale-to-zero workers |
| IaC | Terraform | Multi-cloud |
| Monitoring | Prometheus + Grafana | Standard observability |
| Error tracking | Sentry | Real-time alerting |

---

## Contributing

1. Fork the repo and create a feature branch (`git checkout -b feat/your-feature`)
2. Make your changes with tests
3. Run `pnpm lint && pnpm test` to verify
4. Submit a pull request with a clear description

Please read [CONTRIBUTING.md](CONTRIBUTING.md) for detailed guidelines.

---

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

---

<p align="center">
  Built with ☁️ cloud rendering · 🎨 vector graphics · 🎬 timeline animation
</p>
