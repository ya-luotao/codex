# Experimental Flags and Feature Gates

This proposes a single, coherent feature‑flag system to replace scattered booleans like `experimental_use_rmcp_client`, `experimental_use_unified_exec_tool`, and `include_plan_tool`.

Goals
- Centralize definition, parsing, and lifecycle of flags
- Remove boolean plumbing across modules
- Add/remove experiments with minimal churn
- Consistent TOML/CLI surface; profile‑aware; good defaults

Problems Today
- Booleans are embedded in `Config` (core/src/config.rs:200) and `ConfigToml` (core/src/config.rs:796).
- Call‑sites manually thread flags (e.g., `ToolsConfigParams` in core/src/codex.rs:446, RMCP guard in cli/src/mcp_cmd.rs:285, tool gating in core/src/tools/spec.rs).
- Adding/removing a feature touches many files; toggles live both at top‑level and under `[tools]`.

Design Overview
- Introduce `Feature` enum + `Features` set (owned by `Config`).
- A registry stores metadata: string key, stage, default.
- Code asks `config.features.enabled(Feature::X)`; it never reads individual booleans.

Sketch
```rust
// core/src/features.rs
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum Feature {
    UnifiedExec,
    StreamableShell,
    RmcpClient,
    PlanTool,
    ApplyPatchFreeform,
    ViewImageTool,
    WebSearchRequest,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Stage { Experimental, Beta, Stable, Deprecated, Removed }

pub struct FeatureMeta { pub key: &'static str, pub stage: Stage, pub default_enabled: bool }

#[derive(Debug, Clone, Default, PartialEq, Eq)]
pub struct Features(enumset::EnumSet<Feature>);
impl Features {
    pub fn enabled(&self, f: Feature) -> bool { self.0.contains(f) }
}
```

Config Surface
- Add `pub features: Features` to `Config`.
- New TOML table `[features]`; profiles support `[profiles.<name>.features]`.

Example `~/.codex/config.toml`
```toml
[features]
unified_exec = true
rmcp_client = true
web_search_request = false
plan_tool = true
view_image_tool = true
```

CLI Surface
- Generic toggles, no bespoke flags:
  - `--enable <feature>` (repeatable)
  - `--disable <feature>` (repeatable)
- Introspection: `codex features list` prints known features, stage, and effective state.

Merge Precedence
1) Built‑in defaults (optionally model‑family aware)
2) Profile `[profiles.<name>.features]`
3) Base `[features]`
4) CLI `--enable/--disable`

Call‑Site Examples
- Tools (core/src/tools/spec.rs)
```rust
let use_unified_exec = cfg.features.enabled(Feature::UnifiedExec);
let use_streamable_shell = cfg.features.enabled(Feature::StreamableShell);
let include_plan = cfg.features.enabled(Feature::PlanTool);
let include_apply_patch = cfg.features.enabled(Feature::ApplyPatchFreeform);
let include_view_image = cfg.features.enabled(Feature::ViewImageTool);
let include_web_search = cfg.features.enabled(Feature::WebSearchRequest);
```

- MCP selection (core/src/mcp_connection_manager.rs)
```rust
let use_rmcp = cfg.features.enabled(Feature::RmcpClient);
let (mgr, errs) = McpConnectionManager::new(cfg.mcp_servers.clone(), use_rmcp, cfg.mcp_oauth_credentials_store_mode).await?;
```

- CLI guard (cli/src/mcp_cmd.rs:285)
```rust
if !cfg.features.enabled(Feature::RmcpClient) {
    anyhow::bail!("OAuth login is only supported when features.rmcp_client=true");
}
```

Tool Gating API
- Replace multiple params with `&Features`:
```rust
pub struct ToolsConfigParams<'a> { pub model_family: &'a ModelFamily, pub features: &'a Features }
```

Lifecycle
- Stage drives behavior:
  - Experimental/Beta: opt‑in; may be hidden from help
  - Stable: default based on registry; fully documented
  - Deprecated: parse and warn; still recognized
  - Removed: parse and warn; ignored

Migration (Incremental)
1) Add `core/src/features.rs` and `Config.features`.
2) Parse `[features]`; map old fields into `Features` with deprecation warnings.
3) Switch call‑sites to `features.enabled(...)` and pass `&Features` where needed.
4) Add CLI `--enable/--disable` and `codex features list`; keep old flags for one release mapping to new toggles.
5) Remove old booleans and shims in the following release after snapshots/tests are updated.

Old → New Mapping
- `experimental_use_unified_exec_tool` → `features.unified_exec`
- `experimental_use_exec_command_tool` → `features.streamable_shell`
- `experimental_use_rmcp_client` → `features.rmcp_client`
- `experimental_use_freeform_apply_patch` → `features.apply_patch_freeform`
- `include_apply_patch_tool` → `features.apply_patch_freeform`
- `include_plan_tool` → `features.plan_tool`
- `include_view_image_tool` → `features.view_image_tool`
- `tools.web_search` → `features.web_search_request`

Why This Helps
- One place to add/retire experiments
- Fewer config fields/params; less plumbing
- Consistent TOML/CLI; easy profile overrides
- Cleaner tests: toggle features via a single set
- Back‑compat path without repo‑wide churn

Notes
- This design fits current structure: `Config` remains the single source of truth; `codex.rs` and tool builders consume `Features`; RMCP selection becomes a query. Optional telemetry can log gating decisions centrally in `Features::enabled` when verbose logging is active.
