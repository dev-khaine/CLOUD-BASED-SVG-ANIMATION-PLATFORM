# CLOUD-BASED SVG ANIMATION PLATFORM
## Complete Engineering Design Handbook

**Version:** 1.0  
**Status:** Engineering Blueprint  
**Audience:** Senior Engineers, Architects, Product Teams  

---

## TABLE OF CONTENTS

1. Product Architecture
2. Graphics Engine Design
3. Animation Engine
4. Video Rendering System
5. Cloud Infrastructure
6. Data Models
7. Performance Engineering
8. Export Engine
9. Security
10. Codebase Structure
11. Development Roadmap
12. Optional AI Features

---

# SECTION 1: PRODUCT ARCHITECTURE

## 1.1 System Overview

The platform is a distributed, cloud-native application with four major subsystems:

```
┌────────────────────────────────────────────────────────────────────┐
│                         BROWSER CLIENT                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │  Vector      │  │  Animation   │  │  Preview / Playback      │  │
│  │  Editor UI   │  │  Timeline UI │  │  Engine (WebGL/Canvas)   │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────────────────────┘  │
│         │                 │                                         │
│  ┌──────▼─────────────────▼──────────────────────────────────────┐  │
│  │               Client State Manager (Zustand/Redux)            │  │
│  └──────────────────────────┬──────────────────────────────────┘   │
└─────────────────────────────┼──────────────────────────────────────┘
                              │ WebSocket + REST API
┌─────────────────────────────▼──────────────────────────────────────┐
│                          API GATEWAY                               │
│             (Kong / AWS API Gateway / nginx)                        │
└──┬────────────────────┬────────────────────────┬───────────────────┘
   │                    │                        │
┌──▼──────┐    ┌────────▼────────┐    ┌──────────▼──────────┐
│  Auth   │    │  Project API    │    │  Asset API           │
│ Service │    │  (CRUD/Collab)  │    │  (Upload/CDN)        │
└──┬──────┘    └────────┬────────┘    └──────────┬──────────┘
   │                    │                        │
   │           ┌────────▼────────┐    ┌──────────▼──────────┐
   │           │  Message Queue  │    │  Object Storage      │
   │           │  (RabbitMQ/SQS) │    │  (S3/GCS/R2)         │
   │           └────────┬────────┘    └─────────────────────┘
   │                    │
   │           ┌────────▼────────────────────────┐
   │           │     RENDER WORKER POOL           │
   │           │  ┌──────────┐  ┌──────────────┐  │
   │           │  │ Puppeteer│  │  FFmpeg      │  │
   │           │  │ Renderer │  │  Encoder     │  │
   │           │  └──────────┘  └──────────────┘  │
   │           └─────────────────────────────────┘
   │
┌──▼──────────────────────────────────────────────┐
│                  DATABASE LAYER                  │
│  ┌──────────────┐  ┌──────────┐  ┌────────────┐  │
│  │  PostgreSQL  │  │  Redis   │  │ Typesense  │  │
│  │  (Projects)  │  │  (Cache/ │  │ (Search)   │  │
│  │              │  │  Session)│  │            │  │
│  └──────────────┘  └──────────┘  └────────────┘  │
└─────────────────────────────────────────────────┘
```

## 1.2 Service Decomposition

The backend is split into six independently deployable services:

| Service | Responsibility | Technology |
|---------|---------------|------------|
| **auth-service** | Authentication, JWT, OAuth | Node.js + Passport.js |
| **project-service** | Project CRUD, versioning, collaboration | Node.js + PostgreSQL |
| **asset-service** | Upload, transform, CDN management | Node.js + Sharp |
| **render-service** | Export job management, queue | Node.js + BullMQ |
| **render-worker** | Headless rendering, FFmpeg encoding | Node.js + Puppeteer + FFmpeg |
| **gateway** | Rate limiting, routing, auth verification | nginx / Kong |

## 1.3 Client Architecture

The browser application is a single-page app built in React with the following structural separation:

```
Browser Application
│
├── UI Layer (React Components)
│   ├── Toolbar, Panels, Modals
│   ├── Timeline, Keyframe Editor
│   └── Properties Panel
│
├── Engine Layer (Pure TypeScript, no React)
│   ├── Document Model (scene graph)
│   ├── SVG Renderer (Canvas/WebGL hybrid)
│   ├── Animation Engine
│   └── History Manager (undo/redo)
│
├── Communication Layer
│   ├── REST API Client (axios)
│   ├── WebSocket Client (socket.io-client)
│   └── Autosave Manager
│
└── Worker Layer (Web Workers)
    ├── Layout Worker (heavy path calculations)
    ├── Export Worker (SVG serialization)
    └── Preview Worker (frame pre-rendering)
```

## 1.4 Real-Time Collaboration Architecture

Collaboration uses Operational Transformation (OT) or CRDT (Conflict-free Replicated Data Types). The recommended approach for an SVG editor is a CRDT-based model using **Yjs**:

```
Client A                 Server (Y-WebSocket)          Client B
   │                           │                          │
   │──── applyOperation(op) ──►│                          │
   │                           │──── broadcast(op) ──────►│
   │                           │                          │
   │◄─── ack + remote ops ─────│◄─── applyOperation(op) ──│
   │                           │                          │
   │   Local Y.Doc state       │   Persisted Y.Doc state  │   Local Y.Doc state
   │   stays in sync ──────────│──────────────────────────│── stays in sync
```

Each project document is a Yjs document where:
- `Y.Map` stores document metadata
- `Y.Array` stores the ordered layer list
- Each layer node is a `Y.Map` containing properties and nested children

---

# SECTION 2: GRAPHICS ENGINE DESIGN

## 2.1 Document Model

The document model is a scene graph — an n-ary tree of nodes. Each node is typed and carries its own data schema.

```typescript
// Core node type discriminated union
type NodeType =
  | 'document'
  | 'page'
  | 'group'
  | 'frame'
  | 'path'
  | 'rect'
  | 'ellipse'
  | 'polygon'
  | 'star'
  | 'text'
  | 'image'
  | 'mask'
  | 'boolean-op'
  | 'component'
  | 'component-instance'
  | 'symbol';

interface BaseNode {
  id: string;                     // Nanoid, e.g. "Kj3mP9x"
  type: NodeType;
  name: string;
  parentId: string | null;
  childIds: string[];             // Ordered child references
  visible: boolean;
  locked: boolean;
  opacity: number;                // 0.0 - 1.0
  blendMode: BlendMode;
  transform: Transform;           // 3x3 affine matrix (local space)
  effects: Effect[];              // Drop shadow, blur, etc.
  constraints: Constraints;       // Resize constraints for responsive layouts
}
```

The document uses a **flat map** structure rather than a nested tree. This makes undo/redo, serialization, and CRDT sync dramatically simpler:

```typescript
interface DocumentState {
  id: string;
  version: number;
  name: string;
  // Flat map of ALL nodes — the tree is reconstructed from parentId/childIds
  nodes: Record<string, SceneNode>;
  pages: string[];           // Ordered page IDs
  currentPageId: string;
  assets: AssetLibrary;
  colorStyles: Record<string, ColorStyle>;
  textStyles: Record<string, TextStyle>;
  effectStyles: Record<string, EffectStyle>;
}
```

### Reconstructing the Tree

```typescript
function buildTree(nodes: Record<string, SceneNode>, rootId: string): SceneNode {
  const node = nodes[rootId];
  return {
    ...node,
    children: node.childIds.map(id => buildTree(nodes, id)),
  };
}
```

## 2.2 Transform System

Every node has a local transform stored as a 2D affine matrix (6 components):

```
[ a  c  tx ]
[ b  d  ty ]
[ 0  0   1 ]
```

```typescript
interface Transform {
  a: number;  // Scale X / Rotation cos
  b: number;  // Skew Y / Rotation sin
  c: number;  // Skew X
  d: number;  // Scale Y
  tx: number; // Translate X
  ty: number; // Translate Y
}

// Utilities
const Transform = {
  identity(): Transform {
    return { a: 1, b: 0, c: 0, d: 1, tx: 0, ty: 0 };
  },

  multiply(m1: Transform, m2: Transform): Transform {
    return {
      a:  m1.a * m2.a  + m1.c * m2.b,
      b:  m1.b * m2.a  + m1.d * m2.b,
      c:  m1.a * m2.c  + m1.c * m2.d,
      d:  m1.b * m2.c  + m1.d * m2.d,
      tx: m1.a * m2.tx + m1.c * m2.ty + m1.tx,
      ty: m1.b * m2.tx + m1.d * m2.ty + m1.ty,
    };
  },

  // Get world transform by multiplying up the parent chain
  worldTransform(nodeId: string, nodes: Record<string, SceneNode>): Transform {
    const node = nodes[nodeId];
    if (!node.parentId) return node.transform;
    const parentWorld = Transform.worldTransform(node.parentId, nodes);
    return Transform.multiply(parentWorld, node.transform);
  },

  decompose(m: Transform): { tx: number; ty: number; sx: number; sy: number; rotation: number; skewX: number } {
    const sx = Math.sqrt(m.a * m.a + m.b * m.b);
    const sy = Math.sqrt(m.c * m.c + m.d * m.d);
    const rotation = Math.atan2(m.b, m.a);
    return { tx: m.tx, ty: m.ty, sx, sy, rotation, skewX: Math.atan2(m.c, m.d) - Math.PI / 2 - rotation };
  },
};
```

## 2.3 Layer System

The layer system is built on top of the scene graph. Layers in the UI correspond to nodes in the document model. Key behaviors:

- **Stacking order** is determined by the `childIds` array (last = topmost in rendering)
- **Isolation** — groups and frames create new stacking contexts for blend modes
- **Clipping** — frames can clip their children to bounds
- **Pass-through blend mode** — groups with pass-through allow children to blend with layers below the group

```typescript
interface GroupNode extends BaseNode {
  type: 'group';
  // Pass-through means: composite children directly onto parent stacking context
  blendMode: BlendMode; // if 'pass-through', children composite independently
}

interface FrameNode extends BaseNode {
  type: 'frame';
  width: number;
  height: number;
  fills: Fill[];
  strokes: Stroke[];
  cornerRadius: number | CornerRadii;
  clips: boolean;        // Whether to clip children to frame bounds
  autolayout?: AutoLayout;
}
```

## 2.4 Shape System

```typescript
// Rectangle
interface RectNode extends BaseNode {
  type: 'rect';
  width: number;
  height: number;
  cornerRadius: number | CornerRadii; // Per-corner support
  fills: Fill[];
  strokes: Stroke[];
}

// Ellipse
interface EllipseNode extends BaseNode {
  type: 'ellipse';
  width: number;
  height: number;
  startAngle: number;  // For arcs / pie charts
  endAngle: number;
  innerRadius: number; // For donuts
  fills: Fill[];
  strokes: Stroke[];
}

// Generic path
interface PathNode extends BaseNode {
  type: 'path';
  data: string;          // SVG path data "M 0 0 C 10 10 20 10 30 0 Z"
  windingRule: 'nonzero' | 'evenodd';
  fills: Fill[];
  strokes: Stroke[];
  strokeWidth: number;
  strokeCap: StrokeCap;
  strokeJoin: StrokeJoin;
  dashArray: number[];
}
```

## 2.5 Path Editing Engine

The path editor is the most complex part of the graphics engine. It operates on the SVG path data string, parsed into an internal segment representation:

```typescript
type SegmentType = 'M' | 'L' | 'C' | 'Q' | 'A' | 'Z';

interface PathSegment {
  type: SegmentType;
  points: Point[];   // Control points for the segment
}

interface BezierPoint {
  anchor: Point;          // The on-curve anchor point
  handleIn: Point;        // Incoming tangent control point
  handleOut: Point;       // Outgoing tangent control point
  handleInLinked: boolean; // Mirror handles?
  handleOutLinked: boolean;
  cornerType: 'sharp' | 'smooth' | 'symmetric';
}

// Parse SVG path string into editable points
function parsePath(d: string): BezierPoint[] {
  const segments = new SVGPathParser(d).parse(); // Uses svgpath or similar
  return segments.map(segmentToBezierPoint);
}

// Serialize back to SVG path string
function serializePath(points: BezierPoint[]): string {
  return points.reduce((acc, point, i) => {
    if (i === 0) return `M ${point.anchor.x} ${point.anchor.y}`;
    const prev = points[i - 1];
    if (prev.handleOut.x === prev.anchor.x && prev.handleOut.y === prev.anchor.y &&
        point.handleIn.x === point.anchor.x && point.handleIn.y === point.anchor.y) {
      return acc + ` L ${point.anchor.x} ${point.anchor.y}`;
    }
    return acc + ` C ${prev.handleOut.x} ${prev.handleOut.y} ${point.handleIn.x} ${point.handleIn.y} ${point.anchor.x} ${point.anchor.y}`;
  }, '') + ' Z';
}
```

### Boolean Operations

Path boolean operations (union, subtract, intersect, exclude) are implemented using **Clipper.js** or **paper.js**:

```typescript
import { getPath, PathItem } from 'paper'; // Or use polybool / clipper

async function booleanOp(
  pathA: string,
  pathB: string,
  operation: 'union' | 'subtract' | 'intersect' | 'exclude'
): Promise<string> {
  const paper = await initPaperJS();
  const a = new paper.Path(pathA);
  const b = new paper.Path(pathB);
  const result = {
    union:     () => a.unite(b),
    subtract:  () => a.subtract(b),
    intersect: () => a.intersect(b),
    exclude:   () => a.exclude(b),
  }[operation]();
  return result.pathData;
}
```

## 2.6 Fill and Stroke System

```typescript
type Fill =
  | { type: 'solid'; color: Color; opacity: number }
  | { type: 'linear-gradient'; stops: GradientStop[]; transform: Transform }
  | { type: 'radial-gradient'; stops: GradientStop[]; cx: number; cy: number; r: number }
  | { type: 'image'; assetId: string; scaleMode: 'fill' | 'fit' | 'tile' | 'stretch'; opacity: number }
  | { type: 'pattern'; assetId: string; tileWidth: number; tileHeight: number };

interface Color {
  r: number; // 0-255
  g: number;
  b: number;
  a: number; // 0-1
}
```

## 2.7 Rendering Pipeline

The editor uses a **hybrid rendering strategy**:

- **Interaction layer**: SVG DOM (for hit-testing, selection handles, path editing)
- **Visual layer**: Canvas 2D or OffscreenCanvas (for high-performance compositing)
- **Preview layer**: WebGL (for blur, shadow effects at high zoom)

```
Scene Graph Change
        │
        ▼
   Dirty Flag Set (per node)
        │
        ▼
  RAF (requestAnimationFrame)
        │
        ▼
  ┌─────▼──────────────────────────────┐
  │        Renderer.render()            │
  │                                     │
  │  1. Traverse dirty nodes            │
  │  2. Composite bottom-up             │
  │  3. Apply effects (blur, shadow)    │
  │  4. Draw to OffscreenCanvas         │
  │  5. Blit to visible canvas          │
  └─────────────────────────────────────┘
```

---

# SECTION 3: ANIMATION ENGINE

## 3.1 Timeline Data Model

```typescript
interface AnimationTimeline {
  id: string;
  duration: number;       // Total duration in milliseconds
  fps: number;            // Target FPS for export (24, 30, 60)
  loop: boolean;
  tracks: AnimationTrack[];
}

interface AnimationTrack {
  id: string;
  nodeId: string;         // Which scene node this track targets
  property: AnimatableProperty;
  keyframes: Keyframe[];
  interpolation: InterpolationType; // Default between all keyframes
}

type AnimatableProperty =
  | 'transform.tx' | 'transform.ty'
  | 'transform.sx' | 'transform.sy'
  | 'transform.rotation'
  | 'opacity'
  | 'fill.0.color.r' | 'fill.0.color.g' | 'fill.0.color.b' | 'fill.0.color.a'
  | 'strokeWidth'
  | 'path.data'          // Morphing paths
  | string;              // Supports dot-path notation for any numeric leaf

interface Keyframe {
  id: string;
  time: number;           // Milliseconds from start
  value: AnimatableValue;
  easing: EasingFunction;
  selected?: boolean;     // UI state
}

type AnimatableValue = number | string | Color | Transform;

type EasingFunction =
  | { type: 'linear' }
  | { type: 'ease-in'; power: number }
  | { type: 'ease-out'; power: number }
  | { type: 'ease-in-out'; power: number }
  | { type: 'cubic-bezier'; x1: number; y1: number; x2: number; y2: number }
  | { type: 'spring'; mass: number; stiffness: number; damping: number }
  | { type: 'bounce'; amplitude: number; period: number }
  | { type: 'step'; count: number; direction: 'start' | 'end' };
```

## 3.2 Keyframe Interpolation

```typescript
function interpolateValue(
  fromKeyframe: Keyframe,
  toKeyframe: Keyframe,
  currentTime: number
): AnimatableValue {
  const t = (currentTime - fromKeyframe.time) / (toKeyframe.time - fromKeyframe.time);
  const easedT = applyEasing(fromKeyframe.easing, t);

  if (typeof fromKeyframe.value === 'number') {
    return lerp(fromKeyframe.value as number, toKeyframe.value as number, easedT);
  }
  if (isColor(fromKeyframe.value)) {
    return lerpColor(fromKeyframe.value, toKeyframe.value as Color, easedT);
  }
  if (isTransform(fromKeyframe.value)) {
    return lerpTransform(fromKeyframe.value, toKeyframe.value as Transform, easedT);
  }
  // Path morphing uses linear interpolation of path point positions
  if (typeof fromKeyframe.value === 'string') {
    return morphPaths(fromKeyframe.value, toKeyframe.value as string, easedT);
  }
  return fromKeyframe.value;
}

function applyEasing(easing: EasingFunction, t: number): number {
  switch (easing.type) {
    case 'linear': return t;
    case 'ease-in': return Math.pow(t, easing.power ?? 2);
    case 'ease-out': return 1 - Math.pow(1 - t, easing.power ?? 2);
    case 'ease-in-out': return t < 0.5
      ? 2 * Math.pow(t, 2)
      : 1 - Math.pow(-2 * t + 2, 2) / 2;
    case 'cubic-bezier': return cubicBezierEase(easing.x1, easing.y1, easing.x2, easing.y2, t);
    case 'spring': return springEase(easing.mass, easing.stiffness, easing.damping, t);
    case 'bounce': return bounceEase(t);
    default: return t;
  }
}
```

## 3.3 Playback System

```typescript
class PlaybackEngine {
  private timeline: AnimationTimeline;
  private nodes: Record<string, SceneNode>;
  private rafId: number | null = null;
  private startTime: number = 0;
  private currentTime: number = 0;
  private isPlaying: boolean = false;

  play() {
    this.isPlaying = true;
    this.startTime = performance.now() - this.currentTime;
    this.scheduleFrame();
  }

  pause() {
    this.isPlaying = false;
    if (this.rafId) cancelAnimationFrame(this.rafId);
  }

  seekTo(time: number) {
    this.currentTime = Math.max(0, Math.min(time, this.timeline.duration));
    this.applyTimeToScene(this.currentTime);
  }

  private scheduleFrame() {
    this.rafId = requestAnimationFrame((now) => {
      this.currentTime = now - this.startTime;

      if (this.currentTime >= this.timeline.duration) {
        if (this.timeline.loop) {
          this.startTime = now;
          this.currentTime = 0;
        } else {
          this.currentTime = this.timeline.duration;
          this.pause();
        }
      }

      this.applyTimeToScene(this.currentTime);
      if (this.isPlaying) this.scheduleFrame();
    });
  }

  private applyTimeToScene(time: number) {
    for (const track of this.timeline.tracks) {
      const value = this.evaluateTrack(track, time);
      if (value !== undefined) {
        this.setNodeProperty(track.nodeId, track.property, value);
      }
    }
    // Trigger re-render
    this.emitChange();
  }

  private evaluateTrack(track: AnimationTrack, time: number): AnimatableValue | undefined {
    const sorted = [...track.keyframes].sort((a, b) => a.time - b.time);
    if (sorted.length === 0) return undefined;
    if (time <= sorted[0].time) return sorted[0].value;
    if (time >= sorted[sorted.length - 1].time) return sorted[sorted.length - 1].value;

    for (let i = 0; i < sorted.length - 1; i++) {
      if (time >= sorted[i].time && time <= sorted[i + 1].time) {
        return interpolateValue(sorted[i], sorted[i + 1], time);
      }
    }
  }
}
```

## 3.4 Path Morphing

Path morphing requires that the source and destination paths have the same number of anchor points. If they don't match, the engine resamples the shorter path:

```typescript
function morphPaths(from: string, to: string, t: number): string {
  const fromPoints = parsePath(from);
  const toPoints   = parsePath(to);

  // Normalize point counts
  const normalized = normalizePointCounts(fromPoints, toPoints);

  // Interpolate each anchor and handle
  const morphed = normalized.from.map((fp, i) => ({
    anchor:    lerpPoint(fp.anchor,    normalized.to[i].anchor,    t),
    handleIn:  lerpPoint(fp.handleIn,  normalized.to[i].handleIn,  t),
    handleOut: lerpPoint(fp.handleOut, normalized.to[i].handleOut, t),
    cornerType: fp.cornerType,
    handleInLinked: fp.handleInLinked,
    handleOutLinked: fp.handleOutLinked,
  }));

  return serializePath(morphed);
}
```

## 3.5 Expression-Based Animation

For procedural animations, the engine supports a lightweight expression language:

```typescript
// Example: A node that oscillates based on time
// Property: transform.ty
// Expression: "sin(time * 2) * 50"

interface ExpressionKeyframe {
  type: 'expression';
  expression: string;
  variables: Record<string, string>; // "nodeRef.opacity", "time", "index"
}

class ExpressionEvaluator {
  evaluate(expr: string, context: ExpressionContext): number {
    // Uses a sandboxed math expression parser (mathjs or expr-eval)
    // context: { time, nodeId, index, width, height, ...nodeProperties }
    return mathjs.evaluate(expr, context);
  }
}
```

---

# SECTION 4: VIDEO RENDERING SYSTEM

## 4.1 Architecture Overview

Cloud rendering decouples frame generation from encoding:

```
Browser                   API Server              Render Workers
   │                          │                        │
   │── POST /export ──────────►│                        │
   │                          │── enqueue(job) ────────►│
   │◄── { jobId: "abc" } ─────│                        │
   │                          │                   ┌────▼────────────┐
   │── GET /export/abc/status─►│                   │  Pull job from  │
   │◄── { status: "pending" }─│                   │  queue (BullMQ) │
   │                          │                   └────┬────────────┘
   │                          │                        │
   │                          │                   ┌────▼────────────────────┐
   │                          │                   │  Launch Puppeteer        │
   │                          │                   │  browser in headless     │
   │                          │                   │  mode, load project      │
   │                          │                   └────┬────────────────────┘
   │                          │                        │
   │                          │                   ┌────▼───────────────────────────┐
   │                          │                   │  For each frame 0..N:          │
   │                          │                   │   1. timeline.seekTo(t)        │
   │                          │                   │   2. page.screenshot() → PNG   │
   │                          │                   │   3. Upload frame to S3        │
   │                          │                   └────┬───────────────────────────┘
   │                          │                        │
   │                          │                   ┌────▼───────────────────────────┐
   │                          │                   │  FFmpeg: encode frames to MP4   │
   │                          │                   │  or GIF or Lottie JSON          │
   │                          │                   └────┬───────────────────────────┘
   │                          │                        │
   │                          │◄── complete(jobId, url)│
   │◄── { status: "done",     │                        │
   │      url: "cdn.../out.mp4"}                        │
```

## 4.2 Render Job Schema

```typescript
interface RenderJob {
  id: string;
  projectId: string;
  userId: string;
  status: 'pending' | 'processing' | 'encoding' | 'done' | 'failed';
  createdAt: Date;
  startedAt?: Date;
  completedAt?: Date;
  settings: RenderSettings;
  outputUrl?: string;
  progress: number;       // 0-100
  error?: string;
  workerId?: string;
}

interface RenderSettings {
  format: 'mp4' | 'gif' | 'png-sequence' | 'webm' | 'animated-svg' | 'lottie';
  width: number;
  height: number;
  fps: number;
  quality: number;          // 1-100 (CRF for video, quality for GIF)
  startTime: number;        // ms
  endTime: number;          // ms
  transparentBackground: boolean;
  colorProfile: 'sRGB' | 'P3';
  codec?: 'h264' | 'h265' | 'vp9' | 'av1';
  bitrate?: string;         // "8M" for 8 Mbps
  audioTrackId?: string;
}
```

## 4.3 Headless Renderer

The render worker uses Puppeteer to load a special headless version of the editor:

```typescript
// render-worker/src/renderer.ts
import puppeteer from 'puppeteer';
import { RenderJob } from '@shared/types';

export async function renderJob(job: RenderJob): Promise<string[]> {
  const browser = await puppeteer.launch({
    headless: 'new',
    args: [
      '--no-sandbox',
      '--disable-setuid-sandbox',
      '--disable-dev-shm-usage',
      '--disable-gpu',       // CPU rendering
      '--window-size=3840,2160',
    ],
  });

  const page = await browser.newPage();
  await page.setViewport({ width: job.settings.width, height: job.settings.height, deviceScaleFactor: 1 });

  // Load the headless render page — a stripped-down version of the editor
  // that loads a project by ID, no UI, just the canvas
  await page.goto(`${RENDER_BASE_URL}/render/${job.projectId}?token=${job.renderToken}`, {
    waitUntil: 'networkidle0',
    timeout: 30000,
  });

  // Wait for document to fully load
  await page.waitForFunction('window.renderReady === true', { timeout: 15000 });

  const frameCount = Math.ceil(
    ((job.settings.endTime - job.settings.startTime) / 1000) * job.settings.fps
  );
  const framePaths: string[] = [];

  for (let frame = 0; frame < frameCount; frame++) {
    const timeMs = job.settings.startTime + (frame / job.settings.fps) * 1000;

    // Seek the animation engine to this time
    await page.evaluate((t) => window.animationEngine.seekTo(t), timeMs);

    // Wait for next paint
    await page.evaluate(() => new Promise(resolve => requestAnimationFrame(() => requestAnimationFrame(resolve))));

    const framePath = `/tmp/render/${job.id}/frame_${String(frame).padStart(6, '0')}.png`;
    await page.screenshot({ path: framePath, type: 'png', omitBackground: job.settings.transparentBackground });
    framePaths.push(framePath);

    // Report progress
    await updateJobProgress(job.id, Math.floor((frame / frameCount) * 80));
  }

  await browser.close();
  return framePaths;
}
```

## 4.4 FFmpeg Encoding Pipeline

```typescript
// render-worker/src/encoder.ts
import ffmpeg from 'fluent-ffmpeg';
import path from 'path';

export async function encodeToMP4(
  frameDir: string,
  outputPath: string,
  settings: RenderSettings
): Promise<void> {
  return new Promise((resolve, reject) => {
    ffmpeg()
      .input(path.join(frameDir, 'frame_%06d.png'))
      .inputFPS(settings.fps)
      .videoCodec(settings.codec === 'h265' ? 'libx265' : 'libx264')
      .outputOption('-crf', String(Math.floor(51 - (settings.quality / 100) * 51)))
      .outputOption('-preset', 'slow')           // Better compression
      .outputOption('-pix_fmt', 'yuv420p')       // Maximum compatibility
      .outputOption('-movflags', '+faststart')   // Web streaming optimization
      .size(`${settings.width}x${settings.height}`)
      .fps(settings.fps)
      .output(outputPath)
      .on('progress', (progress) => {
        updateJobProgress(/* job.id */ '...', 80 + progress.percent * 0.2);
      })
      .on('end', resolve)
      .on('error', reject)
      .run();
  });
}

export async function encodeToGIF(
  frameDir: string,
  outputPath: string,
  settings: RenderSettings
): Promise<void> {
  const paletteFile = `/tmp/palette_${Date.now()}.png`;

  // Two-pass GIF encoding for optimal quality
  // Pass 1: Generate optimal palette
  await new Promise<void>((resolve, reject) =>
    ffmpeg()
      .input(path.join(frameDir, 'frame_%06d.png'))
      .inputFPS(settings.fps)
      .videoFilter(`scale=${settings.width}:-1:flags=lanczos,palettegen=stats_mode=diff`)
      .output(paletteFile)
      .on('end', resolve).on('error', reject).run()
  );

  // Pass 2: Encode with palette
  await new Promise<void>((resolve, reject) =>
    ffmpeg()
      .input(path.join(frameDir, 'frame_%06d.png'))
      .inputFPS(settings.fps)
      .input(paletteFile)
      .complexFilter(`scale=${settings.width}:-1:flags=lanczos[x];[x][1:v]paletteuse=dither=bayer:bayer_scale=5:diff_mode=rectangle`)
      .output(outputPath)
      .on('end', resolve).on('error', reject).run()
  );
}
```

## 4.5 Job Queue Management

```typescript
// render-service/src/queue.ts
import { Queue, Worker, QueueEvents } from 'bullmq';
import IORedis from 'ioredis';

const connection = new IORedis({ host: process.env.REDIS_HOST, maxRetriesPerRequest: null });

export const renderQueue = new Queue('render-jobs', {
  connection,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 5000 },
    removeOnComplete: { age: 3600 },
    removeOnFail: { age: 86400 },
  },
});

// Add a new render job
export async function enqueueRenderJob(job: RenderJob): Promise<void> {
  await renderQueue.add(job.id, job, {
    priority: job.userId === 'pro' ? 1 : 10, // Pro users get priority
    jobId: job.id,
  });
}

// The worker that processes jobs
export const renderWorker = new Worker(
  'render-jobs',
  async (bullJob) => {
    const job: RenderJob = bullJob.data;
    try {
      await updateJobStatus(job.id, 'processing');
      const frames = await renderJob(job);
      await updateJobStatus(job.id, 'encoding');
      const outputUrl = await encodeAndUpload(job, frames);
      await updateJobStatus(job.id, 'done', outputUrl);
    } catch (err) {
      await updateJobStatus(job.id, 'failed', undefined, String(err));
      throw err;
    }
  },
  {
    connection,
    concurrency: parseInt(process.env.WORKER_CONCURRENCY ?? '2'),
    limiter: { max: 10, duration: 60000 }, // 10 jobs per minute per worker
  }
);
```

---

# SECTION 5: CLOUD INFRASTRUCTURE

## 5.1 Kubernetes Architecture

```yaml
# kubernetes/namespaces.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: svgstudio
---
# kubernetes/deployments/api.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: project-api
  namespace: svgstudio
spec:
  replicas: 3
  selector:
    matchLabels:
      app: project-api
  template:
    spec:
      containers:
        - name: project-api
          image: registry.example.com/project-api:latest
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1"
              memory: "1Gi"
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
---
# kubernetes/deployments/render-worker.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: render-worker
  namespace: svgstudio
spec:
  replicas: 2        # Base, will be scaled by KEDA
  selector:
    matchLabels:
      app: render-worker
  template:
    spec:
      containers:
        - name: render-worker
          image: registry.example.com/render-worker:latest
          resources:
            requests:
              cpu: "2"
              memory: "4Gi"
            limits:
              cpu: "4"
              memory: "8Gi"
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            capabilities:
              drop: ["ALL"]
              add: ["SYS_ADMIN"]  # Required for Chromium sandboxing
```

## 5.2 Render Worker Autoscaling with KEDA

KEDA (Kubernetes Event-Driven Autoscaling) scales render workers based on the queue depth:

```yaml
# kubernetes/keda/render-scaler.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: render-worker-scaler
  namespace: svgstudio
spec:
  scaleTargetRef:
    name: render-worker
  minReplicaCount: 0          # Scale to zero when idle
  maxReplicaCount: 20         # Max workers
  cooldownPeriod: 120         # Seconds before scaling down
  triggers:
    - type: redis
      metadata:
        address: redis-service:6379
        listName: "bull:render-jobs:wait"
        listLength: "3"        # One worker per 3 queued jobs
```

## 5.3 GPU vs CPU Rendering Strategy

| Scenario | Strategy | Reason |
|----------|----------|--------|
| SVG-based animation | CPU (Chromium) | SVG rendering is CPU-bound; GPU provides minimal benefit |
| Complex blur/effects | WebGL in Chromium | GPU accelerated within the browser |
| Video encoding (H.264) | CPU (libx264) | High quality, widely available |
| Video encoding (H.265/AV1) | GPU optional | `nvenc` on NVIDIA nodes for speed |
| High concurrency | CPU workers | More cost-effective at scale |
| 4K+ resolution | GPU encoding | Dramatically faster |

GPU node configuration for NVIDIA encoding:

```yaml
# kubernetes/gpu-nodes.yaml — for 4K render tier
apiVersion: v1
kind: Node
metadata:
  labels:
    accelerator: nvidia-tesla-t4
---
# Add to render-worker deployment for GPU tier:
resources:
  limits:
    nvidia.com/gpu: 1
env:
  - name: FFMPEG_HWACCEL
    value: "nvenc"
  - name: VIDEO_CODEC
    value: "h264_nvenc"
```

## 5.4 Container Images

```dockerfile
# docker/render-worker/Dockerfile
FROM node:20-slim

# Install Chrome dependencies
RUN apt-get update && apt-get install -y \
    chromium \
    ffmpeg \
    fonts-noto \
    fonts-noto-cjk \
    --no-install-recommends \
    && rm -rf /var/lib/apt/lists/*

# Set Chrome path for Puppeteer
ENV PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium
ENV PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true

WORKDIR /app
COPY package*.json .
RUN npm ci --only=production

COPY dist/ ./dist/

RUN useradd -m renderuser
USER renderuser

CMD ["node", "dist/worker.js"]
```

## 5.5 Infrastructure as Code (Terraform)

```hcl
# infrastructure/terraform/main.tf
# Works on AWS, GCP, or DigitalOcean with provider swap

module "kubernetes_cluster" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "svgstudio-prod"
  cluster_version = "1.29"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  eks_managed_node_groups = {
    # API nodes — regular CPU instances
    api = {
      min_size       = 2
      max_size       = 10
      instance_types = ["t3.large"]
    }

    # Render worker nodes — high CPU
    render = {
      min_size       = 0
      max_size       = 50
      instance_types = ["c6i.2xlarge"]   # 8 vCPU, 16 GB
      taints = [{
        key    = "workload"
        value  = "render"
        effect = "NO_SCHEDULE"
      }]
    }
  }
}

# S3 bucket for assets and rendered outputs
resource "aws_s3_bucket" "assets" {
  bucket = "svgstudio-assets-${var.environment}"
}

resource "aws_s3_bucket" "renders" {
  bucket = "svgstudio-renders-${var.environment}"
}

resource "aws_cloudfront_distribution" "cdn" {
  origin {
    domain_name = aws_s3_bucket.assets.bucket_regional_domain_name
    origin_id   = "S3-assets"
  }
  enabled = true
  # ... CDN config
}
```

## 5.6 Multi-Region Deployment

For production at scale, deploy across regions:

```
                        Global DNS (Route53 / Cloudflare)
                                      │
                  ┌───────────────────┼──────────────────┐
                  ▼                   ▼                   ▼
           US-East-1            EU-West-1            AP-Southeast-1
         ┌──────────┐         ┌──────────┐          ┌──────────┐
         │ API Pods │         │ API Pods │          │ API Pods │
         │ Render   │         │ Render   │          │ Render   │
         │ Workers  │         │ Workers  │          │ Workers  │
         └────┬─────┘         └────┬─────┘          └────┬─────┘
              │                    │                      │
         ┌────▼────────────────────▼──────────────────────▼────┐
         │              Global PostgreSQL (Aurora Global)        │
         │              Primary: US-East-1                       │
         │              Read Replicas: EU, AP                    │
         └───────────────────────────────────────────────────────┘
```

---

# SECTION 6: DATA MODELS

## 6.1 PostgreSQL Schema

```sql
-- Projects table
CREATE TABLE projects (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id      UUID NOT NULL REFERENCES users(id),
  name         VARCHAR(255) NOT NULL,
  description  TEXT,
  thumbnail_url VARCHAR(512),
  is_public    BOOLEAN DEFAULT FALSE,
  is_template  BOOLEAN DEFAULT FALSE,
  created_at   TIMESTAMPTZ DEFAULT NOW(),
  updated_at   TIMESTAMPTZ DEFAULT NOW(),
  deleted_at   TIMESTAMPTZ,           -- Soft delete

  -- Document stored as JSONB for flexibility
  -- In production, large documents stored in S3 with reference here
  document_ref  VARCHAR(512),         -- S3 key for document JSON
  document_size INT,

  -- Metadata
  page_count   INT DEFAULT 1,
  duration_ms  INT DEFAULT 0,
  canvas_width INT DEFAULT 1920,
  canvas_height INT DEFAULT 1080
);

CREATE INDEX ON projects(user_id) WHERE deleted_at IS NULL;
CREATE INDEX ON projects(is_public, is_template);

-- Project versions for history / autosave
CREATE TABLE project_versions (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id   UUID NOT NULL REFERENCES projects(id),
  version      INT NOT NULL,
  document_ref VARCHAR(512) NOT NULL,    -- S3 key
  created_by   UUID REFERENCES users(id),
  created_at   TIMESTAMPTZ DEFAULT NOW(),
  is_autosave  BOOLEAN DEFAULT TRUE,
  label        VARCHAR(255),            -- Named snapshots

  UNIQUE(project_id, version)
);

-- Assets table
CREATE TABLE assets (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id      UUID NOT NULL REFERENCES users(id),
  project_id   UUID REFERENCES projects(id),  -- NULL = global asset
  name         VARCHAR(255) NOT NULL,
  type         VARCHAR(50) NOT NULL,   -- 'image', 'svg', 'font', 'audio'
  mime_type    VARCHAR(100),
  storage_key  VARCHAR(512) NOT NULL,  -- S3 key
  cdn_url      VARCHAR(512),
  file_size    INT,
  width        INT,
  height       INT,
  created_at   TIMESTAMPTZ DEFAULT NOW()
);

-- Render jobs
CREATE TABLE render_jobs (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id   UUID NOT NULL REFERENCES projects(id),
  user_id      UUID NOT NULL REFERENCES users(id),
  status       VARCHAR(50) DEFAULT 'pending',
  format       VARCHAR(50) NOT NULL,
  settings     JSONB NOT NULL,
  output_url   VARCHAR(512),
  error        TEXT,
  progress     INT DEFAULT 0,
  worker_id    VARCHAR(255),
  created_at   TIMESTAMPTZ DEFAULT NOW(),
  started_at   TIMESTAMPTZ,
  completed_at TIMESTAMPTZ
);

CREATE INDEX ON render_jobs(user_id, created_at DESC);
CREATE INDEX ON render_jobs(status) WHERE status IN ('pending', 'processing', 'encoding');

-- Collaboration sessions
CREATE TABLE collaborators (
  project_id   UUID NOT NULL REFERENCES projects(id),
  user_id      UUID NOT NULL REFERENCES users(id),
  role         VARCHAR(50) DEFAULT 'editor',  -- 'owner', 'editor', 'viewer'
  invited_at   TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY(project_id, user_id)
);
```

## 6.2 Document JSON Schema

The document is stored as a JSON file in object storage. Large documents can exceed 50 MB, so they are never stored inline in PostgreSQL:

```json
{
  "id": "proj_Kj3mP9",
  "version": 42,
  "schemaVersion": "1.0.0",
  "name": "Brand Animation",
  "pages": [
    {
      "id": "page_1",
      "name": "Scene 1",
      "width": 1920,
      "height": 1080,
      "background": { "type": "solid", "color": { "r": 255, "g": 255, "b": 255, "a": 1 } }
    }
  ],
  "currentPageId": "page_1",
  "nodes": {
    "frame_main": {
      "id": "frame_main",
      "type": "frame",
      "name": "Canvas",
      "parentId": "page_1",
      "childIds": ["group_logo", "text_headline"],
      "visible": true,
      "locked": false,
      "opacity": 1,
      "blendMode": "normal",
      "transform": { "a": 1, "b": 0, "c": 0, "d": 1, "tx": 0, "ty": 0 },
      "width": 1920,
      "height": 1080,
      "fills": [],
      "clips": true
    }
  },
  "animation": {
    "id": "timeline_1",
    "duration": 5000,
    "fps": 30,
    "loop": false,
    "tracks": [
      {
        "id": "track_1",
        "nodeId": "group_logo",
        "property": "transform.tx",
        "keyframes": [
          { "id": "kf_1", "time": 0, "value": -200, "easing": { "type": "ease-out", "power": 2 } },
          { "id": "kf_2", "time": 500, "value": 0, "easing": { "type": "linear" } }
        ]
      }
    ]
  },
  "assets": {
    "img_logo": {
      "id": "img_logo",
      "type": "image",
      "name": "Company Logo",
      "cdnUrl": "https://cdn.example.com/assets/img_logo.png",
      "width": 400,
      "height": 200
    }
  }
}
```

---

# SECTION 7: PERFORMANCE ENGINEERING

## 7.1 Canvas vs SVG Rendering

The editor uses a dual-layer rendering strategy:

```
┌───────────────────────────────────────────┐
│  HTML Layer (position: absolute, z: 3)    │
│  - Selection handles                       │
│  - Resize grips                            │
│  - Text cursor                             │
│  - Tooltips, snapping guides               │
└───────────────────────────────────────────┘
┌───────────────────────────────────────────┐
│  SVG Overlay (z: 2)                       │
│  - Hit testing target (transparent fill)   │
│  - Path editing control points            │
│  - Bezier handles                          │
└───────────────────────────────────────────┘
┌───────────────────────────────────────────┐
│  Canvas 2D / WebGL (z: 1)                 │
│  - Actual pixel rendering of document     │
│  - High performance compositing           │
│  - Cached layer bitmaps                   │
└───────────────────────────────────────────┘
```

**Why Canvas over SVG for rendering:**
- SVG DOM re-renders entire subtree on any property change
- Canvas allows fine-grained dirty-region updates
- Complex blend modes and effects are much faster on Canvas/WebGL
- Canvas handles thousands of objects without DOM overhead

## 7.2 Spatial Indexing for Hit Testing

With hundreds or thousands of objects, naive hit-testing is O(n). Use an R-tree (bounding box spatial index):

```typescript
import RBush from 'rbush';

interface IndexEntry {
  minX: number; minY: number;
  maxX: number; maxY: number;
  nodeId: string;
}

class SpatialIndex {
  private tree = new RBush<IndexEntry>();

  insert(nodeId: string, bounds: Rect) {
    this.tree.insert({
      minX: bounds.x, minY: bounds.y,
      maxX: bounds.x + bounds.width,
      maxY: bounds.y + bounds.height,
      nodeId,
    });
  }

  hitTest(point: Point): string[] {
    return this.tree.search({
      minX: point.x, minY: point.y,
      maxX: point.x, maxY: point.y,
    }).map(e => e.nodeId);
  }

  // Reindex after transform
  update(nodeId: string, oldBounds: Rect, newBounds: Rect) {
    this.tree.remove({ minX: oldBounds.x, minY: oldBounds.y, maxX: oldBounds.x + oldBounds.width, maxY: oldBounds.y + oldBounds.height, nodeId }, (a, b) => a.nodeId === b.nodeId);
    this.insert(nodeId, newBounds);
  }
}
```

## 7.3 Render Caching

For large documents, cache individual layer renders as ImageBitmap objects. Only re-render layers that have changed since the last frame:

```typescript
class LayerCache {
  private cache = new Map<string, { bitmap: ImageBitmap; contentHash: string }>();

  async getOrRender(
    nodeId: string,
    node: SceneNode,
    renderer: Renderer
  ): Promise<ImageBitmap> {
    const hash = computeNodeHash(node);       // Fast hash of all visual properties
    const cached = this.cache.get(nodeId);

    if (cached && cached.contentHash === hash) {
      return cached.bitmap;
    }

    // Re-render this layer to an OffscreenCanvas
    const offscreen = new OffscreenCanvas(node.width, node.height);
    await renderer.renderNode(node, offscreen);
    const bitmap = await createImageBitmap(offscreen);

    this.cache.set(nodeId, { bitmap, contentHash: hash });
    return bitmap;
  }

  invalidate(nodeId: string) {
    this.cache.delete(nodeId);
  }
}
```

## 7.4 Web Workers for Heavy Computation

```typescript
// src/workers/layout.worker.ts
import { expose } from 'comlink';

const LayoutWorker = {
  computeAutoLayout(frame: FrameNode): Record<string, Transform> {
    // Heavy auto-layout computation off the main thread
    // Returns new transforms for all children
    return computeFlexLayout(frame);
  },

  computePathBoolean(pathA: string, pathB: string, op: BooleanOp): string {
    // CPU-intensive path boolean operations
    return booleanOp(pathA, pathB, op);
  },

  flattenPath(path: string, tolerance: number): string {
    // Convert curves to line segments for export
    return flattenBeziers(path, tolerance);
  },
};

expose(LayoutWorker);

// Main thread usage:
// const worker = wrap<typeof LayoutWorker>(new Worker('./layout.worker'));
// const transforms = await worker.computeAutoLayout(frame);
```

## 7.5 Virtual Rendering for Large Documents

For documents with thousands of objects, virtualize the layer panel and only render objects within the viewport:

```typescript
class VirtualRenderer {
  private viewport: Rect;  // Current view in world coordinates
  private zoom: number;

  getVisibleNodes(nodes: Record<string, SceneNode>): SceneNode[] {
    // Quick reject using spatial index
    return this.spatialIndex
      .search({
        minX: this.viewport.x,
        minY: this.viewport.y,
        maxX: this.viewport.x + this.viewport.width,
        maxY: this.viewport.y + this.viewport.height,
      })
      .map(entry => nodes[entry.nodeId])
      .filter(Boolean);
  }
}
```

## 7.6 Undo/Redo with Structural Sharing

Use Immer for immutable state updates with structural sharing, keeping memory usage manageable:

```typescript
import { produce, enableMapSet } from 'immer';
enableMapSet();

class HistoryManager {
  private past: DocumentState[] = [];
  private future: DocumentState[] = [];
  private current: DocumentState;
  private MAX_HISTORY = 100;

  apply(operation: Operation): void {
    const next = produce(this.current, (draft) => {
      applyOperation(draft, operation);
    });
    this.past.push(this.current);
    if (this.past.length > this.MAX_HISTORY) {
      this.past.shift();  // Drop oldest
    }
    this.future = [];     // Clear redo stack
    this.current = next;
  }

  undo(): DocumentState | undefined {
    if (this.past.length === 0) return;
    this.future.push(this.current);
    this.current = this.past.pop()!;
    return this.current;
  }

  redo(): DocumentState | undefined {
    if (this.future.length === 0) return;
    this.past.push(this.current);
    this.current = this.future.pop()!;
    return this.current;
  }
}
```

---

# SECTION 8: EXPORT ENGINE

## 8.1 SVG Export

The SVG exporter serializes the scene graph to a clean, optimized SVG file:

```typescript
class SVGExporter {
  export(doc: DocumentState, pageId: string, options: SVGExportOptions): string {
    const page = this.getPage(doc, pageId);
    const nodes = doc.nodes;

    const svgEl = this.createSVGRoot(page, options);
    this.appendDefs(svgEl, doc, options);
    this.appendNodes(svgEl, page.childIds, nodes, options);

    if (options.optimize) {
      return this.optimize(svgEl.outerHTML, options);
    }
    return svgEl.outerHTML;
  }

  private appendNodes(parent: Element, childIds: string[], nodes: Record<string, SceneNode>, opts: SVGExportOptions) {
    for (const id of childIds) {
      const node = nodes[id];
      if (!node.visible) continue;

      const el = this.nodeToSVGElement(node, nodes, opts);
      if (el) parent.appendChild(el);
    }
  }

  private nodeToSVGElement(node: SceneNode, nodes: Record<string, SceneNode>, opts: SVGExportOptions): Element | null {
    switch (node.type) {
      case 'rect':    return this.rectToSVG(node as RectNode);
      case 'ellipse': return this.ellipseToSVG(node as EllipseNode);
      case 'path':    return this.pathToSVG(node as PathNode);
      case 'text':    return this.textToSVG(node as TextNode, opts);
      case 'group':   return this.groupToSVG(node as GroupNode, nodes, opts);
      case 'image':   return this.imageToSVG(node as ImageNode, opts);
      default:        return null;
    }
  }
}
```

## 8.2 Animated SVG Export

Animated SVG embeds CSS animations or SMIL animations. CSS is preferred for browser compatibility:

```typescript
class AnimatedSVGExporter extends SVGExporter {
  exportAnimated(doc: DocumentState, pageId: string, options: AnimatedSVGOptions): string {
    const svg = super.export(doc, pageId, options);
    const cssAnimations = this.generateCSSAnimations(doc.animation, doc.nodes);
    return this.embedCSS(svg, cssAnimations);
  }

  private generateCSSAnimations(timeline: AnimationTimeline, nodes: Record<string, SceneNode>): string {
    let css = '';

    for (const track of timeline.tracks) {
      const node = nodes[track.nodeId];
      if (!node) continue;

      const keyframesCSS = this.trackToKeyframeCSS(track, timeline.duration);
      const animationName = `anim_${track.nodeId}_${track.property.replace(/\./g, '_')}`;

      css += `@keyframes ${animationName} {\n${keyframesCSS}}\n`;
      css += `#${track.nodeId} { animation: ${animationName} ${timeline.duration}ms ${timeline.loop ? 'infinite' : '1'} forwards; }\n`;
    }

    return css;
  }

  private trackToKeyframeCSS(track: AnimationTrack, totalDuration: number): string {
    return track.keyframes.map(kf => {
      const percent = (kf.time / totalDuration * 100).toFixed(2);
      const prop = this.animatablePropertyToCSS(track.property);
      return `  ${percent}% { ${prop}: ${this.valueToCSS(kf.value, track.property)}; animation-timing-function: ${this.easingToCSS(kf.easing)}; }\n`;
    }).join('');
  }
}
```

## 8.3 Lottie JSON Export

Lottie is a JSON-based animation format consumed by lottie-player and mobile SDKs:

```typescript
class LottieExporter {
  export(doc: DocumentState, pageId: string): LottieAnimation {
    const timeline = doc.animation;

    return {
      v: '5.9.0',         // Lottie spec version
      fr: timeline.fps,   // Frame rate
      ip: 0,              // In point (first frame)
      op: Math.ceil(timeline.duration / 1000 * timeline.fps), // Out point
      w: doc.pages[0].width,
      h: doc.pages[0].height,
      nm: doc.name,
      ddd: 0,             // 3D: off
      assets: this.exportAssets(doc),
      layers: this.exportLayers(doc, pageId),
    };
  }

  private exportLayers(doc: DocumentState, pageId: string): LottieLayer[] {
    const page = doc.nodes[pageId];
    return page.childIds
      .map(id => this.nodeToLottieLayer(doc.nodes[id], doc))
      .filter(Boolean) as LottieLayer[];
  }
}
```

## 8.4 PNG Sequence Export

PNG sequences are used for compositing in video editors:

```typescript
// In the render worker:
async function exportPNGSequence(job: RenderJob): Promise<string[]> {
  const frames = await renderJob(job); // Already renders PNG frames
  const zipPath = `/tmp/render/${job.id}/frames.zip`;
  await zipDirectory(`/tmp/render/${job.id}`, zipPath, '*.png');
  return [zipPath];
}
```

---

# SECTION 9: SECURITY

## 9.1 SVG Sanitization

SVG files from users are a significant security risk. SVG can contain executable JavaScript, external resource references, and XXE (XML External Entity) attacks:

```typescript
import DOMPurify from 'isomorphic-dompurify';
import { XMLParser } from 'fast-xml-parser';

export function sanitizeSVGUpload(svgContent: string): string {
  // Step 1: Remove all script elements, event handlers, and external refs
  const clean = DOMPurify.sanitize(svgContent, {
    USE_PROFILES: { svg: true, svgFilters: true },
    FORBID_TAGS: ['script', 'use', 'foreignObject'],
    FORBID_ATTR: [
      'onload', 'onerror', 'onclick', 'onmouseover',
      'href',        // Prevent external resource loading
      'xlink:href',
    ],
    FORCE_BODY: false,
  });

  // Step 2: Validate it's well-formed SVG
  const parser = new XMLParser({ ignoreAttributes: false });
  try {
    parser.parse(clean);
  } catch {
    throw new Error('Invalid SVG structure after sanitization');
  }

  // Step 3: Size limit check
  if (clean.length > 10 * 1024 * 1024) {  // 10 MB
    throw new Error('SVG file too large');
  }

  return clean;
}
```

## 9.2 Render Sandbox

Each render worker runs in an isolated environment:

```yaml
# Render worker pod security:
securityContext:
  runAsNonRoot: true
  readOnlyRootFilesystem: true   # Can't write except to /tmp
  allowPrivilegeEscalation: false
  seccompProfile:
    type: RuntimeDefault

# Network policy: render workers can ONLY reach:
# - The internal asset CDN (to load project assets)
# - The internal S3 endpoint (to write output)
# No external network access
```

Puppeteer runs with the `--no-network` flag for projects that don't need external assets:

```typescript
await page.setRequestInterception(true);
page.on('request', (req) => {
  const url = req.url();
  // Only allow requests to our own CDN
  if (url.startsWith(process.env.ASSET_CDN_BASE)) {
    req.continue();
  } else {
    req.abort('blockedbyclient');
  }
});
```

## 9.3 Asset Upload Security

```typescript
// asset-service/src/upload.ts
import fileType from 'file-type';
import sharp from 'sharp';

const ALLOWED_MIME_TYPES = new Set([
  'image/png', 'image/jpeg', 'image/webp', 'image/gif',
  'image/svg+xml', 'font/woff', 'font/woff2',
]);

export async function validateAndProcessUpload(
  buffer: Buffer,
  originalName: string,
  userId: string
): Promise<ProcessedAsset> {
  // 1. Detect actual file type (not just extension)
  const detected = await fileType.fromBuffer(buffer);
  if (!detected || !ALLOWED_MIME_TYPES.has(detected.mime)) {
    throw new SecurityError('File type not allowed');
  }

  // 2. Size limits
  if (buffer.length > 50 * 1024 * 1024) {  // 50 MB
    throw new ValidationError('File too large');
  }

  // 3. For images: reprocess through sharp to strip metadata and validate
  if (detected.mime.startsWith('image/') && detected.mime !== 'image/svg+xml') {
    const processed = await sharp(buffer)
      .withMetadata({ exif: {} })  // Strip EXIF
      .toBuffer();
    return { buffer: processed, mime: detected.mime };
  }

  // 4. For SVG: sanitize
  if (detected.mime === 'image/svg+xml') {
    const cleaned = sanitizeSVGUpload(buffer.toString('utf-8'));
    return { buffer: Buffer.from(cleaned), mime: 'image/svg+xml' };
  }

  return { buffer, mime: detected.mime };
}
```

## 9.4 Authentication & Authorization

```typescript
// Middleware: verify project access
export async function requireProjectAccess(
  req: Request,
  res: Response,
  next: NextFunction,
  requiredRole: 'viewer' | 'editor' | 'owner' = 'viewer'
) {
  const { projectId } = req.params;
  const userId = req.user!.id;

  // Project owner
  const project = await db.projects.findOne({ where: { id: projectId } });
  if (!project) return res.status(404).json({ error: 'Not found' });
  if (project.userId === userId) return next();  // Owner has all access

  // Collaborator
  const collab = await db.collaborators.findOne({ where: { projectId, userId } });
  if (!collab) return res.status(403).json({ error: 'Forbidden' });

  const roleLevels: Record<string, number> = { viewer: 0, editor: 1, owner: 2 };
  if (roleLevels[collab.role] >= roleLevels[requiredRole]) return next();

  return res.status(403).json({ error: 'Insufficient permissions' });
}
```

---

# SECTION 10: CODEBASE STRUCTURE

```
svgstudio/
│
├── apps/
│   ├── web/                          # Main browser application
│   │   ├── src/
│   │   │   ├── components/           # React UI components
│   │   │   │   ├── Editor/
│   │   │   │   │   ├── Canvas/
│   │   │   │   │   │   ├── CanvasView.tsx
│   │   │   │   │   │   ├── SelectionOverlay.tsx
│   │   │   │   │   │   └── GridOverlay.tsx
│   │   │   │   │   ├── Toolbar/
│   │   │   │   │   │   ├── MainToolbar.tsx
│   │   │   │   │   │   └── ContextToolbar.tsx
│   │   │   │   │   ├── Timeline/
│   │   │   │   │   │   ├── TimelinePanel.tsx
│   │   │   │   │   │   ├── TrackRow.tsx
│   │   │   │   │   │   ├── KeyframeHandle.tsx
│   │   │   │   │   │   └── PlayControls.tsx
│   │   │   │   │   ├── Layers/
│   │   │   │   │   │   ├── LayerPanel.tsx
│   │   │   │   │   │   └── LayerRow.tsx
│   │   │   │   │   └── Properties/
│   │   │   │   │       ├── PropertiesPanel.tsx
│   │   │   │   │       ├── FillEditor.tsx
│   │   │   │   │       ├── StrokeEditor.tsx
│   │   │   │   │       └── TransformEditor.tsx
│   │   │   │   ├── Export/
│   │   │   │   │   ├── ExportModal.tsx
│   │   │   │   │   └── RenderProgress.tsx
│   │   │   │   └── common/
│   │   │   │       ├── ColorPicker.tsx
│   │   │   │       ├── NumberInput.tsx
│   │   │   │       └── Slider.tsx
│   │   │   │
│   │   │   ├── engine/               # Core engine (no React dependencies)
│   │   │   │   ├── document/
│   │   │   │   │   ├── DocumentModel.ts
│   │   │   │   │   ├── NodeFactory.ts
│   │   │   │   │   └── DocumentSerializer.ts
│   │   │   │   ├── renderer/
│   │   │   │   │   ├── Renderer.ts
│   │   │   │   │   ├── CanvasRenderer.ts
│   │   │   │   │   ├── WebGLRenderer.ts
│   │   │   │   │   └── LayerCache.ts
│   │   │   │   ├── animation/
│   │   │   │   │   ├── PlaybackEngine.ts
│   │   │   │   │   ├── Interpolator.ts
│   │   │   │   │   ├── EasingFunctions.ts
│   │   │   │   │   └── ExpressionEvaluator.ts
│   │   │   │   ├── tools/
│   │   │   │   │   ├── SelectTool.ts
│   │   │   │   │   ├── PenTool.ts
│   │   │   │   │   ├── RectTool.ts
│   │   │   │   │   ├── EllipseTool.ts
│   │   │   │   │   └── TextTool.ts
│   │   │   │   ├── history/
│   │   │   │   │   ├── HistoryManager.ts
│   │   │   │   │   └── operations/
│   │   │   │   │       ├── MoveOperation.ts
│   │   │   │   │       ├── ResizeOperation.ts
│   │   │   │   │       └── AddNodeOperation.ts
│   │   │   │   └── export/
│   │   │   │       ├── SVGExporter.ts
│   │   │   │       ├── AnimatedSVGExporter.ts
│   │   │   │       └── LottieExporter.ts
│   │   │   │
│   │   │   ├── workers/              # Web Workers
│   │   │   │   ├── layout.worker.ts
│   │   │   │   ├── export.worker.ts
│   │   │   │   └── preview.worker.ts
│   │   │   │
│   │   │   ├── store/                # State management (Zustand)
│   │   │   │   ├── document.store.ts
│   │   │   │   ├── animation.store.ts
│   │   │   │   ├── ui.store.ts
│   │   │   │   └── collaboration.store.ts
│   │   │   │
│   │   │   ├── api/                  # API client
│   │   │   │   ├── client.ts
│   │   │   │   ├── projects.api.ts
│   │   │   │   ├── assets.api.ts
│   │   │   │   └── render.api.ts
│   │   │   │
│   │   │   └── pages/
│   │   │       ├── Editor.tsx
│   │   │       ├── Dashboard.tsx
│   │   │       ├── Auth.tsx
│   │   │       └── render/
│   │   │           └── [projectId].tsx  # Headless render target
│   │   │
│   │   ├── public/
│   │   ├── package.json
│   │   └── vite.config.ts
│   │
│   └── render-headless/              # Stripped editor for cloud rendering
│       ├── src/
│       │   ├── RenderCanvas.tsx
│       │   └── renderBridge.ts       # Exposes window.animationEngine
│       └── vite.config.ts
│
├── services/
│   ├── auth-service/
│   │   ├── src/
│   │   │   ├── routes/
│   │   │   ├── middleware/
│   │   │   └── index.ts
│   │   └── Dockerfile
│   │
│   ├── project-service/
│   │   ├── src/
│   │   │   ├── routes/
│   │   │   ├── repositories/
│   │   │   ├── services/
│   │   │   └── index.ts
│   │   └── Dockerfile
│   │
│   ├── asset-service/
│   │   ├── src/
│   │   │   ├── routes/
│   │   │   ├── processors/
│   │   │   │   ├── imageProcessor.ts
│   │   │   │   └── svgSanitizer.ts
│   │   │   └── index.ts
│   │   └── Dockerfile
│   │
│   ├── render-service/
│   │   ├── src/
│   │   │   ├── routes/
│   │   │   ├── queue/
│   │   │   └── index.ts
│   │   └── Dockerfile
│   │
│   └── render-worker/
│       ├── src/
│       │   ├── renderer.ts
│       │   ├── encoder.ts
│       │   ├── uploader.ts
│       │   └── worker.ts
│       └── Dockerfile
│
├── packages/
│   ├── shared-types/                 # Shared TypeScript types
│   │   ├── src/
│   │   │   ├── document.types.ts
│   │   │   ├── animation.types.ts
│   │   │   ├── render.types.ts
│   │   │   └── index.ts
│   │   └── package.json
│   │
│   └── design-system/               # Shared UI components
│       ├── src/
│       │   ├── tokens.ts
│       │   └── components/
│       └── package.json
│
├── infrastructure/
│   ├── terraform/
│   │   ├── modules/
│   │   │   ├── eks/
│   │   │   ├── rds/
│   │   │   ├── elasticache/
│   │   │   └── s3/
│   │   ├── environments/
│   │   │   ├── staging/
│   │   │   └── production/
│   │   └── main.tf
│   │
│   └── kubernetes/
│       ├── namespaces.yaml
│       ├── deployments/
│       ├── services/
│       ├── ingress/
│       ├── configmaps/
│       ├── secrets/           # Sealed Secrets
│       ├── hpa/               # Horizontal Pod Autoscaler
│       └── keda/              # KEDA ScaledObjects
│
├── docker/
│   ├── docker-compose.yml          # Local development stack
│   ├── docker-compose.test.yml
│   └── .env.example
│
├── scripts/
│   ├── seed-db.ts
│   ├── migrate.ts
│   └── deploy.sh
│
├── docs/
│   ├── architecture/
│   ├── api/                        # OpenAPI specs
│   └── runbooks/
│
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
│
├── package.json                    # Monorepo root (pnpm workspaces)
├── pnpm-workspace.yaml
└── turbo.json                      # Turborepo build orchestration
```

---

# SECTION 11: DEVELOPMENT ROADMAP

## Phase 1: MVP (Months 1–3)

### Sprint 1–2: Foundation (Weeks 1–4)
- [ ] Initialize monorepo (pnpm + Turborepo)
- [ ] Set up Vite + React + TypeScript for web app
- [ ] Set up Express + Prisma for project-service
- [ ] PostgreSQL schema + migrations
- [ ] Redis setup
- [ ] Basic JWT authentication
- [ ] Docker Compose for local development

**Deliverable:** Authenticated user can log in and see empty dashboard.

### Sprint 3–4: Core Editor (Weeks 5–8)
- [ ] Canvas rendering engine (Canvas 2D)
- [ ] Scene graph document model
- [ ] Shape tools: rect, ellipse, path
- [ ] Selection tool with move/resize
- [ ] Transform system (position, scale, rotation)
- [ ] Basic layer panel
- [ ] Fill and stroke properties panel
- [ ] Undo/redo with Immer history

**Deliverable:** User can draw and manipulate basic shapes.

### Sprint 5–6: Animation (Weeks 9–12)
- [ ] Timeline panel UI
- [ ] Keyframe creation and editing
- [ ] Linear interpolation
- [ ] Playback engine (play/pause/seek)
- [ ] Easing presets (ease-in, ease-out, ease-in-out)
- [ ] Property animation (position, opacity, scale, rotation)

**Deliverable:** User can create simple animations with keyframes.

### Sprint 7–8: Cloud Export (Weeks 13–16)
- [ ] Render worker (Puppeteer + FFmpeg, Dockerized)
- [ ] BullMQ job queue
- [ ] Render job API (submit, status, download)
- [ ] MP4 export
- [ ] Export modal with progress indicator
- [ ] S3 upload for rendered output

**Deliverable:** User can render and download an MP4 animation.

## Phase 2: Full Feature Set (Months 4–6)

### Month 4
- [ ] SVG path editor (bezier handles, pen tool)
- [ ] Path boolean operations
- [ ] Image import and embedding
- [ ] Asset library panel
- [ ] GIF export
- [ ] Animated SVG export

### Month 5
- [ ] Text tool with font support
- [ ] Gradient fills
- [ ] Drop shadow and blur effects
- [ ] Groups and frames
- [ ] Mask system
- [ ] Component system (reusable symbols)

### Month 6
- [ ] Lottie JSON export
- [ ] PNG sequence export
- [ ] Project versioning (named saves)
- [ ] Project sharing (public links)
- [ ] Template library
- [ ] Keyboard shortcuts system

## Phase 3: Collaboration & Scale (Months 7–9)

- [ ] Yjs real-time collaboration
- [ ] Cursor presence (see other users)
- [ ] Comment system
- [ ] Team workspaces
- [ ] Permission system (owner/editor/viewer)
- [ ] Kubernetes deployment
- [ ] KEDA render autoscaling
- [ ] CDN for assets (CloudFront)
- [ ] Multi-region deployment
- [ ] Performance profiling and optimization pass

## Phase 4: Polish & Growth (Months 10–12)

- [ ] AI features (see Section 12)
- [ ] Mobile-responsive editor (tablet support)
- [ ] Plugin/extension system
- [ ] Public API for integrations
- [ ] Analytics dashboard for users
- [ ] Subscription billing (Stripe)
- [ ] SOC 2 compliance preparation

---

# SECTION 12: OPTIONAL AI FEATURES

## 12.1 Auto-Animation from Static SVG

Given a static SVG illustration, the AI generates a sensible animation:

```typescript
interface AutoAnimateRequest {
  svgContent: string;
  style: 'gentle' | 'energetic' | 'playful' | 'corporate';
  duration: number;
}

// API call to a fine-tuned model or GPT-4V
async function autoAnimate(req: AutoAnimateRequest): Promise<AnimationTimeline> {
  // 1. Parse SVG into scene graph
  const doc = parseSVGToDocument(req.svgContent);

  // 2. Send to AI with structured output instructions
  const response = await openai.chat.completions.create({
    model: 'gpt-4o',
    messages: [
      {
        role: 'system',
        content: `You are an animation designer. Given an SVG document structure, generate animation keyframes in our JSON format.
        Output ONLY valid JSON matching the AnimationTimeline schema.`
      },
      {
        role: 'user',
        content: `Animate this SVG in a "${req.style}" style over ${req.duration}ms.
        Document structure: ${JSON.stringify(doc.nodes, null, 2)}`
      }
    ],
    response_format: { type: 'json_object' },
  });

  return JSON.parse(response.choices[0].message.content!) as AnimationTimeline;
}
```

## 12.2 AI SVG Generation from Text Prompt

```typescript
// Integration with vector-capable AI models
async function generateSVGFromPrompt(prompt: string): Promise<string> {
  // Option A: Recraft AI — specialized vector generation API
  const response = await fetch('https://external.api.recraft.ai/v1/images/generate', {
    method: 'POST',
    headers: { Authorization: `Bearer ${RECRAFT_API_KEY}` },
    body: JSON.stringify({ prompt, style: 'vector_illustration', response_format: 'svg' }),
  });
  const { svg } = await response.json();

  // Sanitize before returning
  return sanitizeSVGUpload(svg);
}
```

## 12.3 Smart Easing Suggestions

Given the animated property and context, suggest appropriate easing curves:

```typescript
// Uses a lightweight ML model or rule-based system
function suggestEasing(property: AnimatableProperty, context: EasingContext): EasingFunction {
  // Heuristic rules trained on design principles
  if (property === 'opacity' && context.direction === 'in') {
    return { type: 'ease-out', power: 2 };
  }
  if (property.includes('transform') && context.isEntrance) {
    return { type: 'spring', mass: 1, stiffness: 300, damping: 20 };
  }
  if (property.includes('color')) {
    return { type: 'linear' };  // Color always linear
  }
  return { type: 'ease-in-out', power: 2 };
}
```

## 12.4 Character Rigging Assistance

For character animation, the AI detects body parts in an SVG illustration and proposes a rig:

```typescript
interface DetectedRig {
  bones: Bone[];
  skinWeights: Record<string, SkinWeight[]>; // NodeId -> weights per bone
}

interface Bone {
  id: string;
  name: string;        // 'torso', 'left_arm', 'head', etc.
  nodeIds: string[];   // SVG elements belonging to this bone
  parentBoneId?: string;
  pivotPoint: Point;
}

// Uses GPT-4V to analyze SVG structure and image
async function detectCharacterRig(svgContent: string, previewImageBase64: string): Promise<DetectedRig> {
  const response = await openai.chat.completions.create({
    model: 'gpt-4o',
    messages: [{
      role: 'user',
      content: [
        { type: 'text', text: 'Analyze this character SVG and propose a bone rig. Return JSON matching the DetectedRig schema.' },
        { type: 'image_url', image_url: { url: `data:image/png;base64,${previewImageBase64}` } },
      ]
    }],
    response_format: { type: 'json_object' },
  });
  return JSON.parse(response.choices[0].message.content!);
}
```

## 12.5 AI Animation Assistant (Chat Interface)

An in-editor chat assistant that can modify animations through natural language:

```
User: "Make the logo bounce when it enters the scene"
Assistant: Applying spring animation to the logo entrance...
           → Modifies keyframes on group_logo with spring easing
           → Sets amplitude and period for a bounce effect

User: "Make the text fade in 0.5 seconds after the logo"
Assistant: → Adds opacity track for text_headline
           → Keyframes: t=500 opacity:0 → t=1000 opacity:1

User: "Loop the background gradient animation"
Assistant: → Sets timeline.loop = true for the gradient tracks
```

This is implemented as a function-calling agent with access to document-modification tools that directly modify the AnimationTimeline and DocumentState through the same operations used by the editor UI.

---

# APPENDIX A: TECHNOLOGY STACK SUMMARY

| Layer | Technology | Rationale |
|-------|------------|-----------|
| Frontend Framework | React 18 + TypeScript | Ecosystem, concurrent rendering |
| State Management | Zustand | Lightweight, less boilerplate than Redux |
| Build Tool | Vite | Fast HMR, excellent TS support |
| Monorepo | pnpm + Turborepo | Fast, efficient disk usage |
| Canvas Rendering | Canvas 2D + OffscreenCanvas | Performance, fine-grained control |
| Path Math | paper.js (worker) | Mature boolean ops library |
| Collaboration | Yjs + y-websocket | Industry standard CRDT |
| API Framework | Express + Fastify | Mature, flexible |
| ORM | Prisma | Type-safe, excellent DX |
| Database | PostgreSQL 16 | JSONB for documents, rock solid |
| Cache / Queue | Redis + BullMQ | Proven combination |
| Object Storage | S3-compatible (AWS/R2/MinIO) | Universal, cheap |
| Headless Browser | Puppeteer + Chromium | SVG rendering accuracy |
| Video Encoding | FFmpeg | Industry standard |
| Container Orchestration | Kubernetes + KEDA | Autoscaling render workers |
| IaC | Terraform | Multi-cloud, state management |
| CI/CD | GitHub Actions | Integrated, free for open repos |
| Monitoring | Prometheus + Grafana | Standard observability stack |
| Error Tracking | Sentry | Real-time error alerting |

---

# APPENDIX B: KEY PERFORMANCE TARGETS

| Metric | Target |
|--------|--------|
| Editor time-to-interactive | < 2 seconds |
| Canvas render latency (60fps) | < 16ms per frame |
| Undo/redo operation | < 5ms |
| Autosave interval | 15 seconds |
| Render queue time (peak) | < 30 seconds wait |
| 30-second 1080p MP4 render | < 2 minutes |
| 30-second 1080p MP4 render (GPU) | < 30 seconds |
| Asset upload (10MB) | < 5 seconds |
| API response p95 | < 200ms |

---

*End of Engineering Design Handbook — SVGStudio Cloud Platform v1.0*
