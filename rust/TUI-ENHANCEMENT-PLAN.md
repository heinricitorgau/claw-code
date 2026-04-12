# TUI Enhancement Plan — Claw Code (`rusty-claude-cli`)

## Overview

This document analyzes the current terminal user interface and proposes phased modifications to the existing REPL/prompt CLI, with the goal of improving observability and usability while preserving existing architecture and test coverage.

**Research question**: Which TUI improvements have the highest leverage on agent-mediated workflows, and what is the minimum structural change needed to enable them?

---

## 1. Current Architecture Analysis

### Crate map

| Crate | Purpose | Lines | TUI relevance |
|-------|---------|-------|---------------|
| `rusty-claude-cli` | Main binary: REPL loop, arg parsing, rendering, API bridge | ~3,600 | Primary TUI surface |
| `runtime` | Session, conversation loop, config, permissions, compaction | ~5,300 | Provides data/state |
| `api` | Anthropic HTTP client + SSE streaming | ~1,500 | Provides stream events |
| `commands` | Slash command metadata/parsing/help | ~470 | Drives command dispatch |
| `tools` | 18 built-in tool implementations | ~3,500 | Tool execution display |

### Current TUI components

> Note: The legacy prototype files `app.rs` and `args.rs` were removed on 2026-04-05.
> References below describe future extraction targets, not current tracked source files.

| Component | File | Current behavior | Assessment |
|-----------|------|------------------|------------|
| Input | `input.rs` (269 lines) | `rustyline`-based line editor with slash-command tab completion, Shift+Enter newline, history | solid |
| Rendering | `render.rs` (641 lines) | Markdown→terminal rendering (headings, lists, tables, code blocks with syntect highlighting, blockquotes), spinner widget | good |
| App/REPL loop | `main.rs` (3,159 lines) | The monolithic `LiveCli` struct: REPL loop, all slash command handlers, streaming output, tool call display, permission prompting, session management | monolithic |

### Key dependencies

- **crossterm 0.28** — terminal control (cursor, colors, clear)
- **pulldown-cmark 0.13** — Markdown parsing
- **syntect 5** — syntax highlighting
- **rustyline 15** — line editing with completion
- **serde_json** — tool I/O formatting

### Strengths

1. Clean rendering pipeline: Markdown rendering is well-structured with state tracking, table rendering, code highlighting
2. Rich tool display: tool calls get box-drawing borders (`╭─ name ─╮`), results show check/cross icons
3. Comprehensive slash commands: 15 commands covering model switching, permissions, sessions, config, diff, export
4. Session management: full persistence, resume, list, switch, compaction
5. Permission prompting: interactive Y/N approval for restricted tool calls
6. Thorough tests: every formatting function, every parse path has unit tests

### Known weaknesses and gaps

| # | Issue | Impact |
|---|-------|--------|
| 1 | `main.rs` is a 3,159-line monolith — all REPL logic, formatting, API bridging, session management, and tests in one file | High maintenance risk; blocks clean TUI additions |
| 2 | No alternate-screen / full-screen layout — everything is inline scrolling output | Limits spatial information density |
| 3 | No progress indicators beyond a single braille spinner — no streaming progress or token counts during generation | Token usage not visible during generation |
| 4 | No visual diff rendering — `/diff` dumps raw git diff text | Low readability for large diffs |
| 5 | No syntax highlighting in streamed output — markdown rendering only applies to tool results | Inconsistent visual treatment |
| 6 | No status bar / HUD — model, tokens, session info not visible during interaction | Requires slash commands for basic state inspection |
| 7 | No image/attachment preview — `SendUserMessage` resolves attachments but never displays them | Silent attachment handling |
| 8 | Streaming has artificial delay — `stream_markdown` sleeps 8ms per whitespace-delimited chunk | Unnecessary latency |
| 9 | No color theme customization — hardcoded `ColorTheme::default()` | Cannot adapt to light terminals |
| 10 | No resize handling — no terminal size awareness for wrapping, truncation, or layout | Layout breaks on narrow terminals |
| 11 | Historical dual app split — the repo previously carried a separate `CliApp` prototype alongside `LiveCli`; the prototype is gone, but the monolithic `main.rs` still needs extraction | Navigation confusion resolved, but extraction work remains |
| 12 | No pager for long outputs — `/status`, `/config`, `/memory` can overflow the viewport | Requires external piping |
| 13 | Tool results not collapsible — large bash outputs flood the screen | Context loss in multi-tool turns |
| 14 | No thinking/reasoning indicator — no visual distinction when model is in "thinking" mode | Indistinguishable from standard generation |
| 15 | No auto-complete for tool arguments — only slash command names complete | Manual path/model entry required |

---

## 2. Enhancement plan

### Phase 0: Structural cleanup (foundation)

**Goal**: break the monolith, establish the module structure for TUI work.

**Rationale**: Phase 0 is a prerequisite for all subsequent phases. Without it, TUI additions require modifying an untestable monolith.

| Task | Description | Effort |
|------|-------------|--------|
| 0.1 | Extract `LiveCli` into `app.rs` — move the entire `LiveCli` struct, its impl, and helpers (`format_*`, `render_*`, session management) out of `main.rs` into focused modules: `app.rs` (core), `format.rs` (report formatting), `session_manager.rs` (session CRUD) | M |
| 0.2 | Keep the legacy `CliApp` removed — the old `CliApp` prototype has already been deleted; if any unique ideas remain valuable (e.g. stream event handler patterns), reintroduce them intentionally inside the active `LiveCli` extraction rather than restoring the old file wholesale | S |
| 0.3 | Extract `main.rs` arg parsing — the current `parse_args()` is still a hand-rolled parser in `main.rs`. If parsing is extracted later, do it into a newly-introduced module intentionally rather than reviving the removed prototype `args.rs` by accident | S |
| 0.4 | Create a `tui/` module — introduce `crates/rusty-claude-cli/src/tui/mod.rs` as the namespace for all new TUI components: `status_bar.rs`, `layout.rs`, `tool_panel.rs`, etc. | S |

### Phase 1: Status bar and live HUD

**Goal**: persistent information display during interaction.

| Task | Description | Effort |
|------|-------------|--------|
| 1.1 | Terminal-size-aware status line — use `crossterm::terminal::size()` to render a bottom-pinned status bar showing: model name, permission mode, session ID, cumulative token count, estimated cost | M |
| 1.2 | Live token counter — update the status bar in real-time as `AssistantEvent::Usage` and `AssistantEvent::TextDelta` events arrive during streaming | M |
| 1.3 | Turn duration timer — show elapsed time for the current turn (the `showTurnDuration` config already exists in Config tool but is not wired up) | S |
| 1.4 | Git branch indicator — display the current git branch in the status bar (already parsed via `parse_git_status_metadata`) | S |

### Phase 2: Enhanced streaming output

**Goal**: reduce artificial latency; make the response stream visually distinct during generation.

| Task | Description | Effort |
|------|-------------|--------|
| 2.1 | Live markdown rendering — buffer text deltas and incrementally render Markdown as it arrives. The existing `TerminalRenderer::render_markdown` can be adapted for incremental use | L |
| 2.2 | Thinking mode indicator — when extended thinking/reasoning is active, show a distinct animated indicator instead of the generic spinner | S |
| 2.3 | Streaming progress bar — add an optional horizontal progress indicator showing approximate completion (based on max_tokens vs. output_tokens so far) | M |
| 2.4 | Remove artificial stream delay — `stream_markdown` sleeps 8ms per chunk; for the main response stream this should be immediate or configurable | S |

### Phase 3: Tool call visualization

**Goal**: make tool execution legible and navigable.

| Task | Description | Effort |
|------|-------------|--------|
| 3.1 | Collapsible tool output — for results longer than N lines (configurable, default 15), show a truncated view with an expand hint; full output saved to file as fallback | M |
| 3.2 | Syntax-highlighted tool results — when tool results contain code (detected by tool name), apply syntect highlighting rather than plain text | M |
| 3.3 | Tool call timeline — for multi-tool turns, show a compact summary after all tool calls complete | S |
| 3.4 | Diff-aware `edit_file` display — when `edit_file` succeeds, show a colored unified diff of the change instead of a plain confirmation | M |
| 3.5 | Permission prompt enhancement — style the approval prompt with box drawing, color the tool name, show a one-line summary of what the tool will do | S |

### Phase 4: Enhanced slash commands and navigation

**Goal**: improve information display and add missing navigation capabilities.

| Task | Description | Effort |
|------|-------------|--------|
| 4.1 | Colored `/diff` output — parse the git diff and render it with red/green coloring for removals/additions | M |
| 4.2 | Pager for long outputs — when `/status`, `/config`, `/memory`, or `/diff` exceed terminal height, pipe through an internal pager (j/k/q) or external `$PAGER` | M |
| 4.3 | `/search` command — search conversation history by keyword | M |
| 4.4 | `/undo` command — undo the last file edit by restoring from `originalFile` data in `write_file`/`edit_file` tool results | M |
| 4.5 | Interactive session picker — replace text-based `/session list` with an interactive fuzzy-filterable list | L |
| 4.6 | Tab completion for tool arguments — extend `SlashCommandHelper` to complete file paths after `/export`, model names after `/model`, session IDs after `/session switch` | M |

### Phase 5: Color themes and configuration

**Goal**: allow the visual appearance to adapt to terminal environments.

| Task | Description | Effort |
|------|-------------|--------|
| 5.1 | Named color themes — add `dark` (current default), `light`, `solarized`, `catppuccin` themes, wired to the existing `Config` tool's `theme` setting | M |
| 5.2 | ANSI-256 / truecolor detection — detect terminal capabilities and fall back gracefully | M |
| 5.3 | Configurable spinner style — allow choosing between braille dots, bar, moon phases, etc. | S |
| 5.4 | Banner customization — make the ASCII art banner optional or configurable via settings | S |

### Phase 6: Full-screen TUI mode (stretch goal)

**Goal**: optional alternate-screen layout for workflows that benefit from spatial separation.

**Rationale**: full-screen mode is high effort and high risk; it should not be started until Phases 0–3 are stable.

| Task | Description | Effort |
|------|-------------|--------|
| 6.1 | Add `ratatui` dependency — introduce as an optional dependency behind a `full-tui` feature flag | S |
| 6.2 | Split-pane layout — top pane: conversation with scrollback; bottom pane: input area; right sidebar (optional): tool status/todo list | XL |
| 6.3 | Scrollable conversation view — navigate past messages with PgUp/PgDn, search within conversation | L |
| 6.4 | Keyboard shortcuts panel — `?` help overlay with all keybindings | M |
| 6.5 | Mouse support — click to expand tool results, scroll conversation, select text for copy | L |

---

## 3. Priority ordering

### Immediate (high impact, moderate effort)

| Priority | Item | Rationale |
|----------|------|-----------|
| 1 | Phase 0 | `main.rs` at 3,159 lines is the primary maintenance risk and blocks all clean TUI additions |
| 2 | Phase 1.1–1.2 | Token usage is the most frequently requested state information during generation |
| 3 | Phase 2.4 | Low effort; removes an artificial bottleneck with no upside |
| 4 | Phase 3.1 | Large bash outputs are the most common readability issue in multi-tool turns |

### Near-term

| Priority | Item |
|----------|------|
| 5 | Phase 2.1 — live markdown rendering |
| 6 | Phase 3.2 — syntax-highlighted tool results |
| 7 | Phase 3.4 — diff-aware edit display |
| 8 | Phase 4.1 — colored diff for `/diff` |

### Longer-term

| Priority | Item |
|----------|------|
| 9 | Phase 5 — color themes |
| 10 | Phase 4.2–4.6 — enhanced navigation and commands |
| 11 | Phase 6 — full-screen mode (evaluate after Phases 0–3 are stable) |

---

## 4. Architecture recommendations

### Module structure after Phase 0

```
crates/rusty-claude-cli/src/
├── main.rs              # Entrypoint, arg dispatch only (~100 lines)
├── args.rs              # CLI argument parsing (introduce intentionally, not revived from prototype)
├── app.rs               # LiveCli struct, REPL loop, turn execution
├── format.rs            # All report formatting (status, cost, model, permissions, etc.)
├── session_mgr.rs       # Session CRUD: create, resume, list, switch, persist
├── init.rs              # Repo initialization (unchanged)
├── input.rs             # Line editor (unchanged, minor extensions)
├── render.rs            # TerminalRenderer, Spinner (extended)
└── tui/
    ├── mod.rs           # TUI module root
    ├── status_bar.rs    # Persistent bottom status line
    ├── tool_panel.rs    # Tool call visualization (boxes, timelines, collapsible)
    ├── diff_view.rs     # Colored diff rendering
    ├── pager.rs         # Internal pager for long outputs
    └── theme.rs         # Color theme definitions and selection
```

### Design principles

| Principle | Rationale |
|-----------|-----------|
| Keep inline REPL as default; full-screen TUI is opt-in (`--tui` flag) | Avoids breaking existing workflows for users who do not need full-screen mode |
| All formatting functions take `&mut impl Write`, never assume stdout directly | Keeps everything testable without a terminal |
| Rendering must work incrementally, not by buffering the entire response | Streaming-first is a requirement for acceptable latency |
| Use `crossterm` for all terminal control — do not mix raw ANSI escape codes | The current startup banner does this; it needs to be corrected to avoid state conflicts |
| Feature-gate heavy dependencies — `ratatui` should be behind a `full-tui` feature flag | Avoids mandatory compilation cost for users who do not need Phase 6 |

---

## 5. Risk assessment

| Risk | Mitigation |
|------|-----------|
| Breaking the working REPL during Phase 0 refactor | Phase 0 is pure restructuring; existing test coverage is the safety net |
| Terminal compatibility issues (tmux, SSH, Windows) | Rely on crossterm's abstraction; test in degraded environments |
| Performance regression with rich rendering | Profile before/after; keep the fast path (raw streaming) always available |
| Scope creep into Phase 6 | Ship Phases 0–3 as a coherent unit before starting Phase 6 |
| Historical `app.rs` vs `main.rs` confusion | Keep the legacy prototype removed and avoid reintroducing a second app surface accidentally during extraction |

---

*Generated: 2026-03-31 | Workspace: `rust/` | Branch: `dev/rust`*
