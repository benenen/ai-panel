# AI Panel Workbench Design

Date: 2026-04-21

## Goal

Replace the template Tauri + Leptos UI with a real terminal workbench inspired by the overall interaction model of `teminal-panel`, but adapted to this project and rendered on the right side with `xterm.js`.

The first version must be a usable product slice, not just a shell:

- Left sidebar for project management
- Right terminal area powered by `xterm.js`
- Real Tauri/Rust-backed shell or PTY sessions
- SQLite persistence
- Local and SSH projects
- Multiple terminals per project
- `tabs` / `panel` display modes

## Confirmed Product Decisions

### Scope

- Build a working version, not a mock UI.
- Support real terminal sessions.
- Support real project management.

### Persistence

- Use SQLite.

### Project / Terminal Model

- Each project can own multiple terminal sessions.
- The right side supports both `tabs` and `panel` display modes.

### Project Sources

- Support local directory projects.
- Support SSH projects.

### SSH Configuration

- SSH connections are managed inside the app.
- Do not depend on `~/.ssh/config`.
- Authentication for v1 is username + password only.

### Visual Direction

- Use the overall left-project / right-terminal framing, but do not closely clone the reference UI.
- Keep the page visually polished.
- Favor rounded, softer UI over a fully sharp IDE look.
- Keep structure editor-like, but not sterile.

### Confirmed Layout Constraints

- No `AI Panel` / `Projects` title text in the sidebar header.
- Avoid large summary cards such as `Current Project / Active / Mode / Sessions`.
- Keep the top area as a light context bar only.
- Keep tabs and terminal visually separate, not fused into one shell.
- Prefer icons over visible action labels where practical.
- Show meaning via tooltip on hover / focus.
- Keep the terminal as the dominant visual area.

## Page Structure

The page is a two-column workbench:

- Left: project sidebar
- Right: work area

The right work area is further divided into:

1. A light context bar
2. A separate terminal tabs bar with mode switch controls
3. The main terminal stage
4. A lightweight right-side action / info rail

### Sidebar

The sidebar is a project workspace, not a product landing panel.

It contains:

- Top icon controls only
- Search field
- Project list
- Per-project quick glyph actions
- Per-project terminal badges / affordances
- Footer icon actions for add-project and SSH settings

It does not contain:

- Branded product title blocks
- Section headers like `Projects`
- Heavy descriptive copy

### Context Bar

The context bar shows only the minimum useful context for the currently selected project:

- Project name
- Source type indicator
- Top-right icon actions

It should not become a summary dashboard.

### Tabs Bar

The tabs bar sits above the terminal stage and remains visually separate from it.

It contains:

- Active and inactive terminal tabs
- Clear inactive-tab affordance
- Icon-first display mode toggle

It must feel like a real terminal tab strip, but it must not be welded to the terminal container.

### Terminal Stage

The terminal stage is the visual and functional center of the product.

It hosts:

- `xterm.js`
- Terminal output
- Resize behavior
- Focus behavior
- Runtime status surfaces

### Right Rail

The right rail is intentionally lightweight.

It may show:

- Contextual status glyphs
- Quick actions
- Connection-related affordances

It must not become a heavy inspector.

## Component Boundaries

The frontend should be split into focused components instead of pushing everything into one `App`.

### `WorkbenchShell`

Responsibilities:

- Overall layout
- Composition of the main workbench regions

Not responsible for:

- Low-level terminal I/O
- Data persistence

### `ProjectSidebar`

Responsibilities:

- Render projects
- Handle project selection
- Handle sidebar icon actions
- Present per-project terminal affordances

Not responsible for:

- Terminal rendering
- Runtime PTY management

### `ProjectContextBar`

Responsibilities:

- Show the selected project context
- Surface top action icons

Not responsible for:

- Summary dashboards
- Session rendering

### `TerminalTabsBar`

Responsibilities:

- Render terminal tabs
- Switch active terminal
- Close tab
- Toggle `tabs` / `panel` mode

Not responsible for:

- PTY lifecycle
- xterm mounting

### `TerminalStage`

Responsibilities:

- Mount and own `xterm.js`
- Handle resize / focus
- Render terminal output
- Show session-level runtime states

Not responsible for:

- Global project selection
- SQLite persistence

### `ActionRail`

Responsibilities:

- Render lightweight right-side icons
- Surface secondary actions or states

### `TooltipLayer`

Responsibilities:

- Centralize tooltips
- Provide hover and focus semantics for icon-first controls

## State Model

State should be split by concern.

### UI State

Contains:

- Selected project
- Active terminal
- Current display mode
- Tooltip state
- Modal / popover visibility

### Project State

Contains:

- Project list
- SSH services
- Project-to-SSH relationships

Backed by SQLite.

### Runtime Session State

Contains in-memory session information:

- Active runtime terminal handles
- PTY lifecycle state
- Runtime connection state

This is not the same as persisted session metadata.

### Bridge Event State

Represents the event channel between Tauri/Rust and the frontend.

Core event types:

- `session_started`
- `session_output`
- `session_closed`
- `session_error`
- `project_changed`

## Database Design

SQLite should persist configuration and session metadata, not live process state.

### Table: `projects`

Columns:

- `id`
- `name`
- `kind` (`local` or `ssh`)
- `local_path`
- `ssh_service_id`
- `remote_path`
- `created_at`
- `updated_at`

### Table: `ssh_services`

Columns:

- `id`
- `name`
- `host`
- `port`
- `username`
- `password`
- `created_at`
- `updated_at`

### Table: `terminal_sessions`

Persisted metadata only:

- `id`
- `project_id`
- `title`
- `display_mode_scope`
- `sort_order`
- `last_active_at`
- `created_at`

### Optional Table: `app_state`

Useful for lightweight key/value persistence such as:

- last selected project
- last selected tab
- last display mode

This table is optional for v1.

## Data That Must Not Be Stored In SQLite

Do not persist:

- Live local PTY handles
- Live SSH connection handles
- The xterm screen buffer
- Streaming stdout / stderr state

These are runtime concerns and belong in Rust memory only.

## Rust Runtime Session Management

Rust should expose a centralized `SessionManager`.

Suggested responsibility:

- Map `session_id` to either local or SSH-backed runtime sessions
- Provide a unified interface to the frontend and commands layer

Suggested operations:

- `create_local_session`
- `create_ssh_session`
- `write_input`
- `resize`
- `close_session`
- `list_runtime_sessions`

The frontend should only care about `session_id`, not whether the session is local or SSH.

## Frontend / Backend Flow

### Create Local Project

1. User opens add-project flow
2. User chooses local path
3. App writes to SQLite `projects`
4. Sidebar refreshes immediately
5. User can open terminals under the project

### Create SSH Project

1. User creates an SSH service
2. User creates a project that references that SSH service
3. App writes to `ssh_services` and `projects`
4. No SSH connection is established yet
5. Connection is established only when a terminal is opened

### Open Terminal

1. User creates a terminal under the selected project
2. App writes `terminal_sessions` metadata
3. Rust creates the local or SSH-backed session
4. Rust returns `session_id`
5. Frontend attaches the tab to that session
6. Runtime output streams into `xterm.js`

### Switch Tabs / Panel

- This is UI state only.
- It must not recreate PTY sessions.
- The app only changes which session is shown or how multiple sessions are arranged.

### Rename / Close Terminal

- Renaming updates `terminal_sessions.title`
- Closing a tab:
  1. closes the Rust session
  2. removes the frontend tab
  3. removes the persisted session metadata

### Delete Project

Delete flow must:

- Confirm with the user
- Remove linked `terminal_sessions`
- Close linked runtime sessions
- Remove the project

Delete flow must not:

- Automatically delete shared `ssh_services`

## Recovery Strategy

On app restart:

- Recover persisted projects
- Recover persisted SSH services
- Recover persisted terminal session metadata
- Do not recover running PTY processes

Recovered terminal sessions should appear as re-openable session structure, not as magically restored live shells.

This is intentionally safer and smaller in scope for v1.

## Error Handling Boundaries

Handle these errors explicitly in v1:

### SQLite Failure

- Show toast or inline error
- Never fail silently

### Local PTY Startup Failure

- Keep the tab shell visible
- Mark it as failed
- Offer retry

### SSH Authentication Failure

- Surface as connection/auth error
- Do not show a blank terminal as if it succeeded

### SSH Disconnect

- Keep terminal content visible if possible
- Mark the session as disconnected

### Invalid Session

- If the frontend references a missing `session_id`, clean up the stale tab state

## Testing Boundaries

### Rust Repository Tests

Test:

- project CRUD
- SSH service CRUD
- terminal session metadata CRUD

### Rust Session Manager Tests

Test:

- create
- write
- resize
- close
- runtime cleanup

### Frontend State Tests

Test:

- project selection
- tab switching
- display mode switching
- error-state rendering
- tooltip behavior

### Manual Integration Verification

Validate:

- create local project
- create SSH project
- open local terminal
- open SSH terminal
- multiple tabs
- panel/tabs switching
- project deletion closes linked sessions

## Explicitly Out Of Scope For V1

- SSH agent auth
- SSH key file auth
- Auto-restoring live shell processes
- File tree sync
- Drag-and-drop tab sorting
- Heavy right-side inspector
- Rich terminal persistence

## Implementation Notes

- `xterm.js` is the terminal renderer on the frontend.
- Tauri/Rust owns real process and SSH lifecycles.
- SQLite owns configuration and session metadata.
- Leptos owns page state and UI composition.

That separation should remain strict during implementation.
