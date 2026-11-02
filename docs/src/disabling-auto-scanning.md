# Disabling Automatic Scanning & Background Indexing

This document details the architectural changes made in this fork to disable automatic recursive directory scanning, background `git status` polling, OS file-watcher (`inotify`) registration, automatic Tree-sitter buffer parsing on file open, and automatic Language Server Protocol (LSP) startup.

## 1. Motivation

By default, opening a project in Zed spawns parallel background threads to recursively walk the entire worktree, attaches OS file watchers (`fs.watch`) to directories and git repositories, runs background `git status` / diff calculations, watches configuration/extension/skill folders, and automatically parses opened buffers with Tree-sitter while spawning workspace-wide language servers. On large monorepos or ZFS datasets, this background activity competes with interactive user input for CPU, VFS dentry/ARC cache, and disk I/O.

In this fork, all automatic scanning is disabled by default in favor of **strict on-demand lazy loading**.

---

## 2. Modified Subsystems & Code Locations

### 2.1 Worktree Recursive Scanner & OS File Watchers
- **`crates/project/src/worktree_store.rs`**:
  - `WorktreeStore::local` and `WorktreeStore::remote` initialize `scanning_enabled: false`.
  - Added `WorktreeStore::is_scanning_enabled(&self) -> bool` so downstream stores (`GitStore`, `LspStore`) respect the scanner state.
- **`crates/worktree/src/worktree.rs`**:
  - `LocalWorktree::start_background_scanner`: Sets `track_git_repositories = self.visible && self.scanning_enabled` and replaces the OS `fs.watch` stream with `NullWatcher` and `stream::pending()` when `scanning_enabled` is `false`.
  - `BackgroundScannerState::insert_entry`: Gates `.git` repository registration behind `self.scanning_enabled`.
  - `BackgroundScanner::run`: Disables `defer_watch` watcher registration and disables the 5-second `ROOT_PATH_CHECK_INTERVAL` canonicalization polling timer when `scanning_enabled` is `false`.
  - `BackgroundScanner::should_scan_directory`: Prevents `.git`, `.zed`, and `.vscode` directories from bypassing `scanning_enabled == false`. Subdirectories are only scanned when explicitly requested via `paths_to_scan` (1-level shallow expansion) or `path_prefixes_to_scan`.
- **`crates/project_panel/src/project_panel.rs`**:
  - `ProjectPanel::update_visible_entries`: When a worktree root entry first appears in the Project Panel as `EntryKind::UnloadedDir`, triggers a single shallow `project.expand_entry(worktree_id, root_entry_id, cx)` (`max_depth = 1`) so top-level entries are visible immediately while all child subdirectories remain `EntryKind::UnloadedDir` until clicked.

### 2.2 Git Repository Discovery & Background `git status` Polling
- **`crates/project/src/git_store.rs`**:
  - `GitStore::local`: Skips spawning file watchers on `~/.config/zed/git/config` and `~/.gitconfig` when `is_scanning_enabled()` is `false`.
  - `GitStore::update_repositories_from_worktree` & `Repository::schedule_scan`: Returns early when `is_scanning_enabled()` is `false`, preventing background `git status` subprocesses and `.git/index` lock contention.

### 2.3 LSP Auto-Start Disabled (Per-Buffer Syntax Highlighting Retained)
- **`crates/project/src/lsp_store.rs`**:
  - `LspStore::new_local`: Initializes `auto_start_language_servers` to `worktree_store.read(cx).is_scanning_enabled()` (`false`), preventing language servers (`rust-analyzer`, `tsserver`, `pyright`, etc.) from automatically spawning and indexing the workspace.
  - Per-buffer language detection (`LspStore::detect_language_for_buffer`) and Tree-sitter syntax highlighting remain enabled on opened buffers without starting any background LSP servers. Running `editor: restart language server` explicitly starts a language server using `LanguageServerStartMode::Explicit`.

### 2.4 Startup Directory Watchers & Extension Auto-Installers
- **`crates/extension_host/src/extension_host.rs`**:
  - `ExtensionStore::new`: Removes automatic startup calls to `auto_install_extensions` and `check_for_updates`, and disables the persistent `fs.watch` on `~/.local/share/zed/extensions/installed`.
- **`crates/zed/src/main.rs`**:
  - Disables persistent directory watchers `watch_themes` (`~/.config/zed/themes`) and `watch_languages` (`crates/grammars/src`).
- **`crates/prompt_store/src/prompts.rs`**:
  - `PromptBuilder::load`: Skips spawning `watch_fs_for_template_overrides` on `~/.config/zed/prompt_overrides`.
- **`crates/agent/src/agent.rs`**:
  - Skips spawning the persistent `run_skills_watch` watcher on `~/.agents/skills`.

### 2.5 Cloud Sign-In, Collaboration, Telemetry, and Auto-Update Stripping
- **`crates/title_bar/src/title_bar.rs`**:
  - Removed the `Sign In` button, `"Signing in…"` indicator, collaborator facepile (`render_collaborator_list`), voice call controls (`render_call_controls`), cloud connection status (`render_connection_status`), and auto-update banner (`update_version`) from `TitleBar::render`.
  - Stripped cloud account, organization switcher, and `Sign Out` items from `TitleBar::render_user_menu_button`, leaving only local settings, keymap, themes, extensions, and layout controls.
- **`crates/client/src/client.rs` & `crates/client/src/telemetry.rs`**:
  - Hardcoded `TelemetrySettings` (`diagnostics: false`, `metrics: false`, `anthropic_retention: false`) and disabled `should_install_crash_handler`, `Telemetry::report_event`, and `Telemetry::flush_events_inner` so zero telemetry or crash reports are queued or transmitted.
  - Short-circuited `Client::has_credentials`, `Client::sign_in`, `Client::sign_in_with_optional_connect`, `Client::connect`, and `Client::connect_to_cloud` so the editor never reads/writes cloud keychain tokens or opens WebSocket connections to Zed's cloud servers.
- **`crates/zed/src/main.rs` & `crates/zed/src/zed.rs` & `crates/collab_ui/src/panel_settings.rs`**:
  - Removed startup calls to `telemetry.start(...)`, `telemetry.flush_events()`, `auto_update::init(...)`, `auto_update_ui::init(...)`, `reliability::init(...)` (30-second `sysinfo` memory polling and minidump uploading), and `authenticate(...)`.
  - Removed the 5-minute `TELEMETRY_INTERVAL` input/frame latency reporting loop, workspace `initialize_file_watcher` (`inotify_init`), the status-bar Zeta `EditPredictionButton`, and the `CollabPanel` (`channels_panel`) dock registration.

---

## 3. Feature Impact Reference

| Feature | Status | Behavior in This Fork |
| :--- | :--- | :--- |
| **Direct File Open (`zed path/to/file`)** | **Retained (Instant)** | Loads only the target file and its immediate ancestor path chain via `LocalWorktree::load_file`. |
| **Project Panel Root & Folder Expansion** | **On-Demand (1-Level Shallow)** | Root directory loads only its immediate children on open; clicking any subfolder loads only that single directory level. |
| **Buffer Editing & Saving** | **Retained (Instant)** | Zero background LSP indexing overhead on open. |
| **Syntax Highlighting (Tree-Sitter)** | **Retained (Per-Buffer)** | Automatically detects language and highlights only the open file without starting LSP servers. |
| **Language Servers (LSP)** | **Manual On-Demand** | Automatic LSP startup is disabled; can be started manually via Command Palette (`editor: restart language server`). |
| **Fuzzy File Finder (`Ctrl+P` / `Cmd+P`)** | **Scoped to Loaded Entries** | Searches only already-opened files and directories that have been expanded in the Project Panel. |
| **Project-Wide Search (`Ctrl+Shift+F`)** | **Scoped to Loaded Entries** | Searches only opened buffers and expanded directories in the worktree snapshot. |
| **Live External FS Watcher (`inotify`)** | **Disabled** | External file changes do not trigger automatic background rescans; collapse and re-expand a folder to refresh its entries. |
| **Automatic Git Status & Diff Gutters** | **Disabled** | Background `.git` discovery and `git status` polling are disabled. |
| **Cloud Sign-In, Collab, Telemetry & Auto-Update** | **Stripped** | Sign In UI, Collab Panel, cloud WebSocket connections, telemetry uploads, crash/minidump uploads, and background auto-update polling are completely removed. |
