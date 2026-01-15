# FastRender: Complete Analysis of an AI-Written Browser

## Executive Summary

FastRender is a production-quality browser rendering engine written primarily with AI assistance. It's not a demo or research project—it's a serious attempt to build a real browser that renders real pages correctly.

- **Language:** Rust (~1.38 million lines in the main crate alone)
- **Scope:** Complete browser: HTML parsing, CSS cascade, layout, paint, JavaScript execution, and desktop GUI
- **Development Period:** November 16, 2024 – January 14, 2025 (~60 days)
- **Total Commits:** ~29,300+
- **Peak Velocity:** 5,956 commits in a single day (January 13)

---

## Part 1: Architecture & Code Structure

### Repository Structure

```
cursor-full-ai-written-browser/
├── AGENTS.md           # Master instructions for AI agents working on the codebase
├── Cargo.toml          # Main workspace definition (~620 lines)
├── src/                # Main renderer crate (87 items, 1.38M lines)
│   ├── api.rs          # Main orchestration (1.26MB!)
│   ├── dom.rs          # DOM tree (663KB)
│   ├── dom2/           # vm-js backed DOM implementation
│   ├── layout/         # Layout algorithms (282K lines)
│   ├── paint/          # Painting/rasterization (208K lines)
│   ├── style/          # CSS cascade/computation (157K lines)
│   ├── js/             # JavaScript integration (213K lines)
│   ├── text/           # Text shaping/fonts (54K lines)
│   ├── ui/             # Browser chrome UI (137 items)
│   ├── ipc/            # Inter-process communication
│   ├── sandbox/        # OS-level sandboxing (Linux/macOS/Windows)
│   ├── bin/            # 38 binary entrypoints
│   └── ...
├── crates/             # Supporting crates
│   ├── fastrender-renderer/
│   ├── fastrender-ipc/
│   ├── fastrender-shmem/
│   ├── fastrender-yuv/
│   ├── js-wpt-dom-runner/
│   ├── win-sandbox/
│   └── libvpx-sys-bundled/   # VP9 codec
├── vendor/             # Vendored dependencies
│   ├── ecma-rs/        # JavaScript engine (vm-js)
│   ├── taffy/          # Flex/grid layout
│   ├── selectors/      # CSS selector matching
│   └── cmake/
├── docs/               # 90+ documentation files
├── instructions/       # AI workstream instructions
├── tests/              # Unit, integration, WPT, fixture tests
├── specs/              # Linked WHATWG/TC39/CSS specs
├── progress/           # Pageset accuracy tracking JSON
├── scripts/            # Build/test helper scripts
├── benches/            # Performance benchmarks (27 benchmarks)
├── tools/              # WebIDL processing, Unicode data
├── fuzz/               # CSS/selector fuzzing
└── xtask/              # Custom build commands
```

### Rendering Pipeline

The rendering pipeline follows the standard browser model:

```
HTML Parse → DOM Tree → CSS Parse → Cascade/Compute → Box Tree → Layout → Paint → Rasterize
```

**Key structures:**

| Stage | Structure | Description |
|-------|-----------|-------------|
| DOM | `DomNode`, `dom2/` | Traditional DOM and JavaScript-backed implementation |
| Styled Tree | `StyledNode` | Computed CSS values |
| Box Tree | `BoxTree`/`BoxNode` | Anonymous box fixup |
| Fragment Tree | `FragmentTree`/`FragmentNode` | Supports pagination/columns |
| Display List | Paint commands | tiny-skia rasterization |

### Multi-Process Architecture

The browser is moving toward Chromium-style process isolation:

| Process | Responsibility |
|---------|----------------|
| Browser process | Chrome UI, tab management |
| Renderer process | Sandboxed per-site content rendering |
| Network process | Isolated network stack |
| Site isolation | Per-origin renderer processes (OOPIF for cross-origin frames) |

**Platform-specific sandbox implementations:**

| Platform | Technologies |
|----------|--------------|
| Linux | Namespaces + Landlock + seccomp-bpf |
| macOS | Seatbelt (App Sandbox profiles) |
| Windows | AppContainer + Job Objects + restricted tokens |

### Key Technical Decisions

#### 1. Mega-Crate Architecture

Everything is in one crate (`fastrender`). This makes builds slow but keeps dependencies simple. The team has analyzed splitting it but decided the cost/benefit doesn't justify it yet.

#### 2. Vendored JavaScript Engine

Uses `vendor/ecma-rs/`—a custom JavaScript engine called vm-js:

- Full ECMAScript support (modules, async/await, generators)
- WebIDL bindings (auto-generated from spec IDL files)
- HTML script processing model (parser-blocking, defer, async, modules)
- Event loop with microtasks/macrotasks

#### 3. No Page-Specific Hacks

A core philosophy: *"No hostname checks, no magic numbers for one site."* If a page doesn't render correctly, the fix must be a proper spec implementation.

#### 4. Bounded Execution

All JavaScript execution is interruptible with fuel-based budgets. Memory allocations are bounded. Tests run under `timeout -k` to kill runaway processes.

#### 5. Vendored Layout Engine

Uses `vendor/taffy/` for flex/grid layout (fork of the Taffy layout library). Table/block/inline layout is custom.

### Build System

The project uses standard Cargo but with strict guardrails:

```bash
# NEVER run raw cargo commands - use the wrapper
bash scripts/cargo_agent.sh build --release --bin browser

# Always scope builds to specific targets
bash scripts/cargo_agent.sh test -p fastrender --lib

# Always use timeouts (processes can hang)
timeout -k 10 600 bash scripts/cargo_agent.sh test --test integration
```

**Key features:**

- `--features browser_ui` enables the desktop GUI (winit + wgpu + egui)
- `--features vmjs` enables JavaScript execution
- `--profile release-max` for final distribution (LTO, single codegen unit—extremely slow)

**Build times:**

| Build Type | Cold | Incremental |
|------------|------|-------------|
| Type check | 10-30s | ~1s |
| Release binary | ~90s | ~1s |
| Release-max (all targets) | 30-60 minutes | — |

### Testing Infrastructure

Comprehensive testing at every level:

| Test Type | Description |
|-----------|-------------|
| Unit tests | In `src/` alongside code (`cargo test --lib`) |
| Integration tests | Exactly 2 binaries (`integration.rs`, `allocation_failure.rs`) |
| Visual fixtures | HTML → PNG golden comparisons |
| WPT harness | Web Platform Tests (local subset) |
| Chrome comparison | Automated diff reports against Chrome renders |
| test262 | JavaScript parser conformance |
| Fuzzing | CSS parsing, selector matching |

Pageset accuracy is tracked in committed JSON files under `progress/pages/*.json`.

### Development Workflow

```bash
# Development workflow
bash scripts/cargo_agent.sh check -p fastrender          # Fast type-check
bash scripts/cargo_agent.sh test -p fastrender --lib     # Unit tests
bash scripts/cargo_agent.sh build --release --bin browser --features browser_ui

# Run the browser
./target/release/browser https://example.com

# Render a page to PNG
bash scripts/cargo_agent.sh run --release --bin fetch_and_render -- https://example.com out.png

# Compare against Chrome
bash scripts/cargo_agent.sh xtask fixture-chrome-diff
```

The `xtask` crate provides high-level commands: `pageset`, `update-goldens`, `render-page`, `fixture-chrome-diff`, etc.

---

## Part 2: Development History & Process

### Timeline & Velocity Overview

| Period | Days | Commits | Avg/Day | Focus |
|--------|------|---------|---------|-------|
| Nov 16-30 | 11 | 93 | 8 | Foundation ("Wave" tasks) |
| Dec 1-15 | 4 active | 32 | 8 | Major refactor pause |
| Dec 16-22 | 7 | 1,789 | 256 | CSS property explosion |
| Dec 23-31 | 9 | 1,473 | 164 | Performance, disk caching, pageset |
| Jan 1-7 | 7 | 3,899 | 557 | Browser UI, JS foundation |
| Jan 8-14 | 7 | 22,016 | 3,145 | Full acceleration (JS, video, multiprocess) |

**Peak days:**

- **Jan 13:** 5,956 commits (video/media, multiprocess, WebSocket)
- **Jan 11:** 4,892 commits (vm-js runtime, native codegen)
- **Jan 12:** 4,579 commits (vm-js generators, session persistence)

### Phase-by-Phase Development

#### Phase 1: Foundation (Nov 16-30)

**Pattern:** Structured "Wave" methodology

Each commit was a discrete task with explicit identifiers:

```
W1.R01: CSS 2.1 Visual Formatting Model research
W1.T03: Length, LengthUnit, and LengthOrAuto Types
W2.T04: Layout Constraints
W3.T08: FlexFormattingContext with Taffy integration
W4.T12: Inline Formatting Context
W5.T06: Canvas wrapper for tiny-skia 2D graphics
W6.T03: Implement public API for FastRender
```

**Key milestone:** Nov 29—complete rendering pipeline working:
```
964d6d10b Review project completion and overall status (#93)
```

#### Phase 2: Refactor Pause (Dec 1-8)

Only 3 commits in 8 days—major architectural review:

```
11ee74c03 Refactor codebase for better architecture and quality (#94)
f813e0c90 Review codebase for production readiness (#95)
a2de88288 Improvements to rendering (#96)
```

This created 6 planning docs under `docs/refactor/`. The project recognized structural issues before scaling.

#### Phase 3: CSS Property Explosion (Dec 16-22)

**Pattern:** Rapid feature iteration with immediate testing

Dec 17 (771 commits) shows the pattern:

- **Morning:** Add pseudo-classes (`:placeholder-shown`, `:read-only`, `:checked`)
- **Midday:** Add transforms, clip, overflow properties
- **Afternoon:** Test on real sites (wikipedia, yahoo, nasa)
- **Evening:** Fix issues discovered on real sites

Real site testing began:

```
d7b79ffc2 Note wikipedia render success
330ecaff9 Note wikipedia all-white render
23b61be3f Log findings from cached wikipedia HTML (display:none overlays)
```

**Challenge discovered:** Many sites render blank because:

1. Content hidden behind JS-only overlays
2. Missing CSS property support
3. Need for noscript fallback handling

#### Phase 4: Christmas Week Performance Push (Dec 23-31)

**Pattern:** Optimization and caching

Key themes: disk cache for pageset runs, web font loading optimization, performance profiling, multi-script font fallback.

```
56b17d603 feat: honor HTTP cache freshness for disk caches
f8b33be63 feat: enable disk cache for pageset runs
919e8f004 feat: add workload-aware auto layout parallelism
```

**Christmas Day (Dec 25):** 127 commits including:

```
e631a28f6 Fix border-image parsing
bd96ac4d8 Parse cursor image lists
f22b6abc7 Handle selector keys in negations
```

#### Phase 5: Agent Integration Pattern (Dec 25-26)

**Discovery:** Parallel AI agents working simultaneously

A distinct pattern emerges:

```
b0924c608 Integrate agent-72: Add :has cascade microbenchmark
68e1d16d3 Integrate agent-73: Inline SVG document CSS CDATA regression test
85d9d63a8 Integrate agent-78: Support ::first-line/::first-letter
21d8b274e Integrate agent-79: Fix feTile origin + subregion masking
9f689976f Integrate agent-80: multicol fragmentation + pagination fixes
```

This explains the high commit counts—multiple AI agents working in parallel, with their changes being merged by numbered agent ID.

#### Phase 6: Browser UI & JavaScript (Jan 1-9)

**Pattern:** Feature acceleration with JS engine integration

**Jan 1 (649 commits):** Performance + prefetching:

```
5a5604989 perf(css): avoid double-copying custom property raw values
4d0be5493 feat: prefetch more HTML subresources
4534428ab perf(css): memoize parsed property values with thread-local LRU
```

**Jan 8 (858 commits):** Browser UI skeleton:

```
cb162e22d feat: add feature-gated browser UI deps and browser bin
11832cc8d feat(browser_ui): add winit/wgpu/egui browser skeleton
336414541 feat: add BrowserDocument for multi-frame cached rendering
```

**Jan 9 (2,025 commits):** JavaScript event loop:

```
6ec05cf4f feat(js): deterministic clocked event loop with HTML timers
86ea5e55e feat(js): add classic script scheduler with async/defer ordering
5d0efec18 feat: add DOM Events core (Event/EventTarget dispatch + listeners)
```

#### Phase 7: Full Acceleration (Jan 10-14)

**Pattern:** vm-js integration, video, multiprocess

**Jan 10 (1,579 commits):** ecma-rs vendoring:

```
e26c5ce15 chore(js): vendor ecma-rs and update paths
ba04c8d1b feat(browser_ui): per-tab page zoom with viewport↔DPR mapping
1e5430ab8 feat(browser_ui): reopen closed tab (Ctrl/Cmd+Shift+T)
```

**Jan 11 (4,892 commits):** Native runtime experiments:

```
f79a3691f fix(js): use real VmHost for property gets
28e6c0d10 paint: support off-canvas filter/backdrop layers
327786d6a fix(runtime-native): dedupe gc::align_up and stabilize tests
```

**Jan 13 (5,956 commits):** Video/audio + multiprocess:

```
13fe439ca feat: add bundled libvpx sys crate (vendored build)
71c11d23f feat: add WebM demuxer (VP9+Opus) with seek support
bb769bf91 feat: add Linux seccomp-bpf renderer sandbox
```

### Key Challenges Observed

#### 1. Blank Page Renders (Dec 17-18)

Sites rendering as solid white:

```
330ecaff9 Note wikipedia all-white render
ec41435b4 Note news.ycombinator.com still paints blank despite layout
216f91a00 Note nationalgeographic.com blank render
```

**Root causes identified:**

- JavaScript-only content with no noscript fallback
- CSS `display:none` overlays covering content
- Missing CSS property implementations

#### 2. Merge Conflicts from Parallel Development

Throughout the project, especially after agent integration:

```
53d3523ff Remove SCRATCHPAD to avoid conflicts
6bd024f1f Remove scratch_notes to reduce conflicts
```

**Solution:** Removed scratchpad files entirely.

#### 3. Performance vs Correctness Balance

Many commits show dual focus:

```
ccbbce5ac Reuse flex pass cache with adjusted measurement styles
8bbf54e0f Cache media query evaluation reuse
```

But always correctness first:

```
a7d2186b4 Clean up codebase: fix doc tests, remove debug prints, and dead code
```

#### 4. vm-js Integration Complexity (Jan 8-14)

Heavy churn around JavaScript engine integration:

```
e73d17a30 fix: sync vm-js integration + UI navigation messages
670112511 fix(js): adapt fastrender to vm-js API changes
ab0c0a323 fix: restore builds after JS/dom2 force_async refactors
```

#### 5. Build Stability Under Rapid Change

Frequent "restore build" commits indicate aggressive parallelism:

```
b432fc2a2 fix: resolve merge conflicts and restore build
f80c57a92 fix: restore cargo test compilation
```

### Development Patterns Identified

#### 1. Scratchpad Note-Taking (Removed Later)

Early development used inline notes:

```
b074845d8 Update scratchpad
07352e634 Note latest commit status in scratchpad
```

Later removed due to merge conflicts.

#### 2. Real-Site Testing Loop

Consistent pattern:

1. Add feature
2. Test on pageset sites
3. Note failures in commit message
4. Fix and test again

```
654aae55f Add discord/weather/bbc targets to fetch_pages
d7b79ffc2 Note wikipedia render success
6839491e0 Note example.com render check
```

#### 3. Agent-Numbered Work Streams

Parallel agents working on independent tasks:

```
Integrate agent-72, agent-73, agent-78, agent-79...
```

#### 4. Regression-First Development

Almost every feature commit paired with test:

```
fa1d3f245 Add :placeholder-shown pseudo-class support
 → test immediately follows
```

#### 5. Clippy/Lint Days

Periodic cleanup sprints (Dec 19-20 especially):

```
dfd544f5b clippy: avoid unnecessary or_else in flex inline percentage base
7c445e819 Remove clippy allow on Containment::with_flags
```

### Commit Distribution by Type

Based on systematic sampling:

| Type | Estimated % | Notes |
|------|-------------|-------|
| `fix:` | ~34% | Correctness-first culture |
| `test:` | ~20% | Strong regression discipline |
| `feat:` | ~15% | New capabilities |
| `perf:` | ~8% | Optimization work |
| `docs:` | ~8% | Documentation |
| `chore:` | ~5% | Maintenance |
| `refactor:` | ~4% | Structure improvements |
| Progress notes | ~6% | "Note X renders", scratchpad updates |

### Technical Decisions Over Time

#### Decisions Made and Kept

1. **Single mega-crate** — Never split despite 1.38M lines
2. **tiny-skia for rasterization** — Pure Rust, no system deps
3. **Vendored Taffy** — Frozen flex/grid engine
4. **vm-js custom engine** — Not V8 or SpiderMonkey
5. **No page-specific hacks** — Spec compliance enforced

#### Decisions Evolved

1. Scratchpad files → Removed (merge conflict issues)
2. Note-taking in commits → Moved to progress JSON files
3. ecma-rs as submodule → Vendored (Jan 10)

#### Late Additions (Jan 13)

- **libvpx bundled** — VP9 video decoding
- **WebM demuxer** — Audio/video container
- **seccomp-bpf sandbox** — Linux security
- **Landlock filesystem sandbox** — Linux security

---

## Part 3: AI-Assisted Development Methodology

### Workstreams

The project is organized into parallel workstreams (from `AGENTS.md`):

- **Capability buildout:** Implementing spec primitives (CSS/layout algorithms)
- **Pageset page loop:** Fixing real pages one-by-one
- **Browser chrome:** Tabs, navigation, address bar
- **Browser responsiveness:** Frame rate, input latency
- **JavaScript engine/DOM/Web APIs:** Four separate workstreams

### Non-Negotiables

- No page-specific hacks
- No deviating-spec behavior
- No post-layout pixel nudging
- No panics in production
- JavaScript must be bounded (no infinite loops crashing the browser)

### Philosophy

The `docs/philosophy.md` reads like hard-won engineering wisdom:

- *"Correct pixels are the product"*
- *"90% accuracy, 10% performance"*
- *"Data-driven: inject, trace, collect, understand, systematize"*
- *"No vanity work"* — changes must improve accuracy or eliminate crashes
- *"Budgets are boundaries, not strategy"*

### What Worked

1. **Parallel agent architecture** — Multiple agents on independent tasks
2. **Numbered agent tracking** — Clear provenance of changes
3. **Structured early phases** — Wave methodology for foundation
4. **Evidence-driven loop** — Chrome comparison, pageset tracking

### Challenges of Scale

1. **Merge conflicts** — Required removing shared scratch files
2. **Build stability** — Frequent "restore build" commits
3. **API churn** — vm-js integration required many adaptations

### Velocity Observations

- **Peak sustained rate:** ~5,000 commits/day (Jan 11-13)
- That's approximately **one commit every 17 seconds** on peak days
- Suggests multiple agents running in parallel, each producing focused commits

---

## Summary & Assessment

### Strengths

1. **Excellent documentation:** 90+ markdown files covering architecture, philosophy, workflows, debugging, security. The docs are clearly written from real experience.

2. **Rigorous safety:** Timeouts everywhere, memory limits, no unbounded operations. This is what production browser development looks like.

3. **Spec-first approach:** Refusing to add hacks is painful but correct for long-term maintainability.

4. **Parallel workstreams:** Good organization for AI agents or multiple contributors.

5. **Evidence-based workflow:** Chrome comparison reports, pageset accuracy tracking, visual diffs—everything is measurable.

6. **Security-conscious:** Sandboxing on all platforms, site isolation, IPC boundaries.

### Areas of Interest

1. **Mega-crate challenges:** 1.38M lines in one crate makes compilation slow and IDE tooling struggle. The team acknowledges this but hasn't split yet.

2. **Vendored JS engine:** Using a custom vm-js instead of V8/SpiderMonkey is ambitious. It's a massive undertaking to match production browser JS performance.

3. **Browser UI in egui:** Currently the chrome is egui-based. There's a workstream to eventually render chrome with FastRender itself ("renderer-chrome").

4. **Still experimental:** JavaScript integration is labeled "experimental." Many Web APIs are incomplete.

### Final Observations

FastRender is an impressive project—a serious attempt to build a modern browser engine from scratch with AI assistance. The codebase shows engineering maturity: rigorous safety, comprehensive testing, spec-first development, and excellent documentation.

The unique aspect is the `AGENTS.md` and `instructions/` directory—these are explicit instructions for AI coding assistants, making this perhaps **the first browser built primarily through AI-human collaboration at this scale**.

The development pattern confirms a highly parallel AI-assisted workflow where multiple agents work on independent features, with frequent integration and build stabilization. The methodical start, strategic pause for refactoring, and explosive growth phase demonstrate thoughtful project management despite the unprecedented velocity.
