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

### 2.3 Language Detection, Tree-Sitter Parsing, and LSP Auto-Start
- **`crates/project/src/lsp_store.rs`**:
  - `LspStore::new_local`: Initializes `auto_start_language_servers` to `worktree_store.read(cx).is_scanning_enabled()` (`false`), preventing language servers (`rust-analyzer`, `tsserver`, `pyright`, etc.) from automatically spawning and indexing the workspace.
  - `LspStore::on_buffer_added` & `LspStore::detect_language_for_buffer`: Skips automatic modeline parsing and automatic language/grammar detection when `is_scanning_enabled()` is `false`. Opened files render immediately as plain text without triggering background Tree-sitter parses (`Buffer::reparse`).
  - Manual override remains available: selecting a language via the Language Selector UI (`language_selector::Toggle` -> `Project::set_language_for_buffer`) loads the grammar for that specific buffer on demand, and running `editor: restart language server` explicitly starts a language server using `LanguageServerStartMode::Explicit`.

### 2.4 Startup Directory Watchers & Extension Auto-Installers
- **`crates/extension_host/src/extension_host.rs`**:
  - `ExtensionStore::new`: Removes automatic startup calls to `auto_install_extensions` and `check_for_updates`, and disables the persistent `fs.watch` on `~/.local/share/zed/extensions/installed`.
- **`crates/zed/src/main.rs`**:
  - Disables persistent directory watchers `watch_themes` (`~/.config/zed/themes`) and `watch_languages` (`crates/grammars/src`).
- **`crates/prompt_store/src/prompts.rs`**:
  - `PromptBuilder::load`: Skips spawning `watch_fs_for_template_overrides` on `~/.config/zed/prompt_overrides`.
- **`crates/agent/src/agent.rs`**:
  - Skips spawning the persistent `run_skills_watch` watcher on `~/.agents/skills`.

---

## 3. Feature Impact Reference

| Feature | Status | Behavior in This Fork |
| :--- | :--- | :--- |
| **Direct File Open (`zed path/to/file`)** | **Retained (Instant)** | Loads only the target file and its immediate ancestor path chain via `LocalWorktree::load_file`. |
| **Project Panel Root & Folder Expansion** | **On-Demand (1-Level Shallow)** | Root directory loads only its immediate children on open; clicking any subfolder loads only that single directory level. |
| **Buffer Editing & Saving** | **Retained (Instant)** | Zero background Tree-sitter or LSP overhead on open; files open immediately as plain text. |
| **Syntax Highlighting (Tree-Sitter)** | **Manual On-Demand** | Disabled on automatic file open; enabled per-buffer only when manually selecting a language in the Language Selector (`Ctrl+Shift+L` / Command Palette: `language selector: toggle`). |
| **Language Servers (LSP)** | **Manual On-Demand** | Automatic LSP startup is disabled; can be started manually via Command Palette (`editor: restart language server`). |
| **Fuzzy File Finder (`Ctrl+P` / `Cmd+P`)** | **Scoped to Loaded Entries** | Searches only already-opened files and directories that have been expanded in the Project Panel. |
| **Project-Wide Search (`Ctrl+Shift+F`)** | **Scoped to Loaded Entries** | Searches only opened buffers and expanded directories in the worktree snapshot. |
| **Live External FS Watcher (`inotify`)** | **Disabled** | External file changes do not trigger automatic background rescans; collapse and re-expand a folder to refresh its entries. |
| **Automatic Git Status & Diff Gutters** | **Disabled** | Background `.git` discovery and `git status` polling are disabled. |
