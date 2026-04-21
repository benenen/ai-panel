# AI Panel Workbench Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the template Leptos + Tauri app with a usable terminal workbench that supports SQLite-backed local and SSH projects, multiple terminal sessions per project, `tabs` / `panel` display modes, and a right-side `xterm.js` terminal stage.

**Architecture:** Keep persistence, runtime sessions, and UI state strictly separated. SQLite stores configuration and terminal-session metadata; Rust owns live PTY and SSH runtime handles; Leptos owns layout, selection state, and `xterm.js` mounting. The frontend talks to Rust only through Tauri commands plus a narrow session event channel.

**Tech Stack:** Leptos CSR, Tauri 2, SQLite (`rusqlite`), local PTY runtime, SSH password auth runtime, `xterm.js`, wasm-bindgen interop, cargo tests, trunk.

---

## File Map

### Frontend files

- Modify: `src/app.rs`
- Modify: `src/main.rs`
- Modify: `styles.css`
- Modify: `index.html`
- Create: `src/models.rs`
- Create: `src/state.rs`
- Create: `src/bridge.rs`
- Create: `src/components/mod.rs`
- Create: `src/components/workbench_shell.rs`
- Create: `src/components/project_sidebar.rs`
- Create: `src/components/project_context_bar.rs`
- Create: `src/components/terminal_tabs_bar.rs`
- Create: `src/components/terminal_stage.rs`
- Create: `src/components/action_rail.rs`
- Create: `src/components/tooltip_layer.rs`
- Create: `src/xterm.rs`
- Create: `public/vendor/xterm/xterm.min.js`
- Create: `public/vendor/xterm/xterm.min.css`
- Create: `public/vendor/xterm/xterm-addon-fit.min.js`
- Create: `public/xterm-bridge.js`

### Backend files

- Modify: `src-tauri/Cargo.toml`
- Modify: `src-tauri/src/lib.rs`
- Create: `src-tauri/src/models/mod.rs`
- Create: `src-tauri/src/models/project.rs`
- Create: `src-tauri/src/models/ssh_service.rs`
- Create: `src-tauri/src/models/terminal_session.rs`
- Create: `src-tauri/src/db/mod.rs`
- Create: `src-tauri/src/db/schema.rs`
- Create: `src-tauri/src/db/projects.rs`
- Create: `src-tauri/src/db/ssh_services.rs`
- Create: `src-tauri/src/db/terminal_sessions.rs`
- Create: `src-tauri/src/runtime/mod.rs`
- Create: `src-tauri/src/runtime/session_manager.rs`
- Create: `src-tauri/src/runtime/local.rs`
- Create: `src-tauri/src/runtime/ssh.rs`
- Create: `src-tauri/src/commands/mod.rs`
- Create: `src-tauri/src/commands/projects.rs`
- Create: `src-tauri/src/commands/ssh_services.rs`
- Create: `src-tauri/src/commands/terminals.rs`
- Create: `src-tauri/tests/repository.rs`
- Create: `src-tauri/tests/session_manager.rs`

### Optional cleanup / support files

- Modify: `.gitignore` to ignore `.superpowers/`

---

### Task 1: Add Shared Domain Models And Frontend State

**Files:**
- Create: `src/models.rs`
- Create: `src/state.rs`
- Modify: `src/app.rs`
- Test: `src/state.rs`

- [ ] **Step 1: Write the failing frontend state tests**

```rust
#[test]
fn workbench_state_tracks_selected_project_and_active_terminal() {
    let mut state = WorkbenchState::default();
    state.select_project("project-1".into());
    state.set_active_terminal("project-1".into(), "session-1".into());

    assert_eq!(state.selected_project_id.as_deref(), Some("project-1"));
    assert_eq!(state.active_terminal_id.as_deref(), Some("session-1"));
}

#[test]
fn workbench_state_switches_between_tabs_and_panel_modes() {
    let mut state = WorkbenchState::default();
    assert_eq!(state.display_mode, DisplayMode::Tabs);

    state.toggle_display_mode();
    assert_eq!(state.display_mode, DisplayMode::Panel);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test -p ai-panel-ui workbench_state -- --nocapture`
Expected: FAIL because `WorkbenchState` and `DisplayMode` do not exist yet.

- [ ] **Step 3: Write minimal shared model and UI state implementation**

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum ProjectKind {
    Local,
    Ssh,
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum DisplayMode {
    Tabs,
    Panel,
}

#[derive(Debug, Default, Clone)]
pub struct WorkbenchState {
    pub selected_project_id: Option<String>,
    pub active_terminal_id: Option<String>,
    pub display_mode: DisplayMode,
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test -p ai-panel-ui workbench_state -- --nocapture`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/models.rs src/state.rs src/app.rs
git commit -m "feat: add frontend workbench state"
```

---

### Task 2: Add SQLite Schema And Repository Layer

**Files:**
- Modify: `src-tauri/Cargo.toml`
- Create: `src-tauri/src/models/mod.rs`
- Create: `src-tauri/src/models/project.rs`
- Create: `src-tauri/src/models/ssh_service.rs`
- Create: `src-tauri/src/models/terminal_session.rs`
- Create: `src-tauri/src/db/mod.rs`
- Create: `src-tauri/src/db/schema.rs`
- Create: `src-tauri/src/db/projects.rs`
- Create: `src-tauri/src/db/ssh_services.rs`
- Create: `src-tauri/src/db/terminal_sessions.rs`
- Test: `src-tauri/tests/repository.rs`

- [ ] **Step 1: Write failing repository tests for projects, SSH services, and terminal-session metadata**

```rust
#[test]
fn inserts_and_reads_local_project() {
    let repo = test_repository();
    let created = repo.projects().create(CreateProject {
        name: "alpha-api".into(),
        kind: ProjectKind::Local,
        local_path: Some("~/workspace/alpha".into()),
        ssh_service_id: None,
        remote_path: None,
    }).unwrap();

    let loaded = repo.projects().list().unwrap();
    assert_eq!(loaded.len(), 1);
    assert_eq!(loaded[0].id, created.id);
}

#[test]
fn inserts_and_reads_ssh_service() {
    let repo = test_repository();
    let created = repo.ssh_services().create(CreateSshService {
        name: "prod".into(),
        host: "example.com".into(),
        port: 22,
        username: "root".into(),
        password: "secret".into(),
    }).unwrap();

    let loaded = repo.ssh_services().list().unwrap();
    assert_eq!(loaded[0].id, created.id);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test -p ai-panel --test repository -- --nocapture`
Expected: FAIL because repository modules and models are missing.

- [ ] **Step 3: Add dependencies and implement schema + repositories**

```toml
[dependencies]
rusqlite = { version = "0.32", features = ["bundled"] }
uuid = { version = "1", features = ["v4", "serde"] }
time = { version = "0.3", features = ["formatting", "macros"] }
```

```rust
pub fn migrate(conn: &Connection) -> rusqlite::Result<()> {
    conn.execute_batch(
        r#"
        CREATE TABLE IF NOT EXISTS projects (...);
        CREATE TABLE IF NOT EXISTS ssh_services (...);
        CREATE TABLE IF NOT EXISTS terminal_sessions (...);
        "#,
    )
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test -p ai-panel --test repository -- --nocapture`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src-tauri/Cargo.toml src-tauri/src/models src-tauri/src/db src-tauri/tests/repository.rs
git commit -m "feat: add sqlite repositories for workbench data"
```

---

### Task 3: Add Tauri Commands For Projects, SSH Services, And Session Metadata

**Files:**
- Modify: `src-tauri/src/lib.rs`
- Create: `src-tauri/src/commands/mod.rs`
- Create: `src-tauri/src/commands/projects.rs`
- Create: `src-tauri/src/commands/ssh_services.rs`
- Create: `src-tauri/src/commands/terminals.rs`
- Modify: `src-tauri/src/db/mod.rs`
- Test: `src-tauri/tests/repository.rs`

- [ ] **Step 1: Write a failing command-level test for project creation and listing**

```rust
#[test]
fn create_project_command_persists_project() {
    let app = test_app_state();
    let created = create_project(
        app.clone(),
        CreateProjectPayload {
            name: "alpha-api".into(),
            kind: "local".into(),
            local_path: Some("~/workspace/alpha".into()),
            ssh_service_id: None,
            remote_path: None,
        },
    ).unwrap();

    let projects = list_projects(app).unwrap();
    assert_eq!(projects[0].id, created.id);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test -p ai-panel create_project_command_persists_project -- --nocapture`
Expected: FAIL because command modules and app state wiring do not exist.

- [ ] **Step 3: Implement app state and Tauri commands**

```rust
#[derive(Clone)]
pub struct AppState {
    pub repository: Arc<Repository>,
    pub sessions: Arc<SessionManager>,
}

#[tauri::command]
pub fn create_project(
    state: tauri::State<'_, AppState>,
    payload: CreateProjectPayload,
) -> Result<ProjectDto, String> {
    state.repository.projects().create(payload.try_into()?).map(Into::into).map_err(to_string)
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test -p ai-panel create_project_command_persists_project -- --nocapture`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/lib.rs src-tauri/src/commands src-tauri/tests/repository.rs
git commit -m "feat: expose tauri commands for project and ssh data"
```

---

### Task 4: Add Local PTY Runtime And Session Manager

**Files:**
- Modify: `src-tauri/Cargo.toml`
- Create: `src-tauri/src/runtime/mod.rs`
- Create: `src-tauri/src/runtime/session_manager.rs`
- Create: `src-tauri/src/runtime/local.rs`
- Modify: `src-tauri/src/commands/terminals.rs`
- Test: `src-tauri/tests/session_manager.rs`

- [ ] **Step 1: Write failing tests for local session lifecycle**

```rust
#[test]
fn local_session_manager_creates_writes_and_closes_session() {
    let manager = SessionManager::new();
    let session_id = manager.create_local_session(CreateLocalSession {
        working_dir: ".".into(),
        shell: None,
    }).unwrap();

    manager.write_input(&session_id, "echo hello\n").unwrap();
    manager.resize(&session_id, 120, 40).unwrap();
    manager.close_session(&session_id).unwrap();

    assert!(manager.get(&session_id).is_none());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test -p ai-panel --test session_manager local_session_manager_creates_writes_and_closes_session -- --nocapture`
Expected: FAIL because session manager does not exist.

- [ ] **Step 3: Add PTY dependency and implement local runtime**

```toml
[dependencies]
portable-pty = "0.8"
crossbeam-channel = "0.5"
```

```rust
pub struct SessionManager {
    sessions: Mutex<HashMap<String, RuntimeSession>>,
}

pub fn create_local_session(&self, request: CreateLocalSession) -> Result<String, SessionError> {
    // spawn local PTY, start read pump, register handle
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test -p ai-panel --test session_manager local_session_manager_creates_writes_and_closes_session -- --nocapture`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src-tauri/Cargo.toml src-tauri/src/runtime src-tauri/src/commands/terminals.rs src-tauri/tests/session_manager.rs
git commit -m "feat: add local pty session manager"
```

---

### Task 5: Add SSH Password Session Runtime

**Files:**
- Modify: `src-tauri/Cargo.toml`
- Create: `src-tauri/src/runtime/ssh.rs`
- Modify: `src-tauri/src/runtime/session_manager.rs`
- Modify: `src-tauri/src/commands/terminals.rs`
- Test: `src-tauri/tests/session_manager.rs`

- [ ] **Step 1: Write a failing SSH runtime unit test around request validation and session registration**

```rust
#[test]
fn ssh_session_request_requires_existing_service() {
    let manager = SessionManager::new();
    let err = manager.create_ssh_session(CreateSshSession {
        ssh_service: None,
        remote_path: "/srv/app".into(),
    }).unwrap_err();

    assert!(err.to_string().contains("ssh service"));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test -p ai-panel --test session_manager ssh_session_request_requires_existing_service -- --nocapture`
Expected: FAIL because SSH session creation path does not exist.

- [ ] **Step 3: Add SSH runtime implementation**

```toml
[dependencies]
ssh2 = "0.9"
```

```rust
pub fn create_ssh_session(&self, request: CreateSshSession) -> Result<String, SessionError> {
    // connect, authenticate with username/password, open shell channel, register session
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test -p ai-panel --test session_manager ssh_session_request_requires_existing_service -- --nocapture`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src-tauri/Cargo.toml src-tauri/src/runtime/ssh.rs src-tauri/src/runtime/session_manager.rs src-tauri/src/commands/terminals.rs src-tauri/tests/session_manager.rs
git commit -m "feat: add ssh password terminal sessions"
```

---

### Task 6: Add Session Event Bridge To The Frontend

**Files:**
- Modify: `src-tauri/src/commands/terminals.rs`
- Modify: `src-tauri/src/lib.rs`
- Create: `src/bridge.rs`
- Create: `src/models.rs` additions for DTOs/events
- Test: `src/state.rs`

- [ ] **Step 1: Write a failing frontend state test for applying session events**

```rust
#[test]
fn session_output_event_appends_to_terminal_buffer() {
    let mut state = WorkbenchState::default();
    state.ensure_terminal("session-1".into(), "alpha-api".into());
    state.apply_event(AppEvent::SessionOutput {
        session_id: "session-1".into(),
        chunk: "hello".into(),
    });

    assert_eq!(state.terminals["session-1"].buffer, "hello");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test -p ai-panel-ui session_output_event_appends_to_terminal_buffer -- --nocapture`
Expected: FAIL because event application logic does not exist.

- [ ] **Step 3: Implement the bridge and event models**

```rust
pub enum AppEvent {
    SessionStarted { session_id: String, project_id: String },
    SessionOutput { session_id: String, chunk: String },
    SessionClosed { session_id: String },
    SessionError { session_id: String, message: String },
}
```

```rust
pub async fn invoke<T: Serialize, R: DeserializeOwned>(cmd: &str, payload: &T) -> Result<R, JsValue> {
    // thin wrapper over window.__TAURI__.core.invoke
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test -p ai-panel-ui session_output_event_appends_to_terminal_buffer -- --nocapture`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/bridge.rs src/models.rs src/state.rs src-tauri/src/commands/terminals.rs src-tauri/src/lib.rs
git commit -m "feat: add frontend bridge for session events"
```

---

### Task 7: Add xterm.js Bridge And Terminal Stage

**Files:**
- Modify: `index.html`
- Modify: `styles.css`
- Create: `public/vendor/xterm/xterm.min.js`
- Create: `public/vendor/xterm/xterm.min.css`
- Create: `public/vendor/xterm/xterm-addon-fit.min.js`
- Create: `public/xterm-bridge.js`
- Create: `src/xterm.rs`
- Create: `src/components/terminal_stage.rs`
- Modify: `src/components/mod.rs`
- Modify: `src/app.rs`

- [ ] **Step 1: Write a failing minimal frontend test for terminal mounting state**

```rust
#[test]
fn terminal_view_state_tracks_mounted_session() {
    let mut state = WorkbenchState::default();
    state.mount_terminal_view("session-1".into());
    assert_eq!(state.mounted_terminal_id.as_deref(), Some("session-1"));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test -p ai-panel-ui terminal_view_state_tracks_mounted_session -- --nocapture`
Expected: FAIL because terminal mount state does not exist.

- [ ] **Step 3: Implement JS bridge + `xterm.js` stage**

```javascript
export function createTerminal(containerId) {
  const terminal = new Terminal({ convertEol: true, cursorBlink: true });
  const fitAddon = new FitAddon.FitAddon();
  terminal.loadAddon(fitAddon);
  terminal.open(document.getElementById(containerId));
  fitAddon.fit();
  return { terminal, fitAddon };
}
```

```rust
pub fn TerminalStage(/* props */) -> impl IntoView {
    // create host element, initialize xterm on mount, push streamed output into bridge
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test -p ai-panel-ui terminal_view_state_tracks_mounted_session -- --nocapture`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add index.html styles.css public/xterm-bridge.js public/vendor/xterm src/xterm.rs src/components/terminal_stage.rs src/components/mod.rs src/app.rs src/state.rs
git commit -m "feat: add xterm terminal stage"
```

---

### Task 8: Implement The Rounded Editor-Like Workbench UI

**Files:**
- Create: `src/components/workbench_shell.rs`
- Create: `src/components/project_sidebar.rs`
- Create: `src/components/project_context_bar.rs`
- Create: `src/components/terminal_tabs_bar.rs`
- Create: `src/components/action_rail.rs`
- Create: `src/components/tooltip_layer.rs`
- Modify: `src/components/mod.rs`
- Modify: `src/app.rs`
- Modify: `styles.css`
- Test: `src/state.rs`

- [ ] **Step 1: Write a failing state test for tab switching and project deletion cleanup**

```rust
#[test]
fn deleting_project_removes_linked_terminal_tabs() {
    let mut state = WorkbenchState::default();
    state.insert_project("project-1".into(), "alpha-api".into());
    state.insert_terminal("session-1".into(), "project-1".into(), "term-1".into());

    state.delete_project("project-1");

    assert!(!state.terminals.contains_key("session-1"));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test -p ai-panel-ui deleting_project_removes_linked_terminal_tabs -- --nocapture`
Expected: FAIL because project / terminal cleanup behavior is not implemented.

- [ ] **Step 3: Implement the shell and feature-complete UI**

```rust
view! {
    <WorkbenchShell>
        <ProjectSidebar /* ... */ />
        <ProjectContextBar /* ... */ />
        <TerminalTabsBar /* ... */ />
        <TerminalStage /* ... */ />
        <ActionRail /* ... */ />
        <TooltipLayer /* ... */ />
    </WorkbenchShell>
}
```

- [ ] **Step 4: Run tests and manual verification**

Run:

```bash
cargo test -p ai-panel-ui -- --nocapture
cargo test -p ai-panel --test repository -- --nocapture
cargo test -p ai-panel --test session_manager -- --nocapture
cargo tauri dev
```

Expected:

- Unit tests pass
- Repository tests pass
- Session manager tests pass
- App launches and supports:
  - create local project
  - create SSH service
  - create SSH project
  - open local terminal
  - open SSH terminal
  - switch tabs / panel
  - delete project and close linked sessions

- [ ] **Step 5: Commit**

```bash
git add src/app.rs src/components src/state.rs styles.css
git commit -m "feat: build ai panel workbench ui"
```

---

### Task 9: Final Cleanup And Safety Checks

**Files:**
- Modify: `.gitignore`
- Modify: `README.md`

- [ ] **Step 1: Add `.superpowers/` to `.gitignore` if it is still unignored**

```gitignore
.superpowers/
```

- [ ] **Step 2: Replace template README guidance with workbench-specific run instructions**

```md
## Development

- `cargo tauri dev`
- SQLite data path notes
- Local and SSH terminal prerequisites
```

- [ ] **Step 3: Run final repository checks**

Run:

```bash
git diff --check
cargo test -p ai-panel-ui -- --nocapture
cargo test -p ai-panel --test repository -- --nocapture
cargo test -p ai-panel --test session_manager -- --nocapture
```

Expected:

- no whitespace errors
- all tests pass

- [ ] **Step 4: Commit**

```bash
git add .gitignore README.md
git commit -m "chore: finalize ai panel workbench setup"
```
