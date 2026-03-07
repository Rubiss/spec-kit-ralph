# Research: Port Ralph Loop to Spec-Kit Extension

## R1: Extension Structure & File Layout

**Decision**: Extension-native flat structure at repo root

**Rationale**: The [Extension Development Guide](https://github.com/github/spec-kit/blob/main/extensions/EXTENSION-DEVELOPMENT-GUIDE.md) specifies extensions as self-contained directories with `extension.yml` at root. Since this is a dedicated extension repo (not embedded in a larger project), the repo root IS the extension root. This aligns with the distribution pattern: `git clone` + `specify extension add --dev ./spec-kit-ralph`.

**Alternatives Considered**:
- Nested `src/` structure: Rejected — no Python/compiled code exists; this is scripts + markdown commands
- Monorepo with extension as subdirectory: Rejected — single-purpose repo, unnecessary nesting

**Source Reference**: Extension Development Guide § Distribution → Option 1: GitHub Repository

---

## R2: Script Porting Strategy

**Decision**: Port `ralph-loop.ps1` and `ralph-loop.sh` with minimal modifications

**Rationale**: The existing scripts (368 LOC PowerShell, 350 LOC Bash) in `C:\Users\Rubis\Projects\spec-kit` are mature and tested. They are already mostly self-contained — the `common.ps1` import is guarded with `if (Test-Path)` and only used for optional path helpers. Key changes needed:

1. **Remove `common.ps1`/`common.sh` dependency**: These are spec-kit core scripts. The extension scripts must be fully self-contained. The only usage is optional — inline what's needed or skip.
2. **Config file loading**: Add loading of `ralph-config.yml` from `.specify/extensions/ralph/` for defaults (model, max iterations, agent CLI path). Script parameters still override config values.
3. **Agent CLI path**: Read from config's `agent_cli` field instead of hardcoding `copilot`. Default to `copilot` if not configured.
4. **Agent name**: The scripts reference `--agent speckit.ralph` — keep this unchanged since the agent profile name is stable.

**Alternatives Considered**:
- Rewrite scripts from scratch: Rejected — existing scripts are proven and handle all edge cases (Ctrl+C, consecutive failures, completion detection)
- Add abstraction layer for agent CLIs: Rejected — spec clarification Q3 confirms Copilot-only at v1.0; premature abstraction

**Source Files**:
- `C:\Users\Rubis\Projects\spec-kit\scripts\powershell\ralph-loop.ps1` (368 lines)
- `C:\Users\Rubis\Projects\spec-kit\scripts\bash\ralph-loop.sh` (350 lines)

---

## R3: Agent File Placement Strategy

**Decision**: Extension bundles `speckit.ralph.agent.md`; the `run.md` command handles placement into `.github/agents/` as a prerequisite step

**Rationale**: The spec-kit extension system's `CommandRegistrar` currently only supports Claude (`.claude/commands/` registration via `register_commands_for_claude()`). Copilot reads agent profiles from `.github/agents/*.agent.md`. Since the extension system has no `register_commands_for_copilot()` method, the extension must handle agent file placement itself.

The `run.md` command (thin launcher) checks for `.github/agents/speckit.ralph.agent.md` in the target project. If missing, it copies from the extension's `agents/` directory. This is:
- Self-contained (no core changes needed)
- Idempotent (safe to run repeatedly)
- Transparent (user sees the setup step)

**Alternatives Considered**:
- Contribute `register_commands_for_copilot()` to spec-kit core: Rejected — out of scope for this extension; would require core PR
- Require manual copy: Rejected — violates SC-001 (install + run with two commands, no additional manual config)
- Post-install hook: Rejected — extension system has no post-install hook event

**Key Detail**: The agent profile is stored at `agents/speckit.ralph.agent.md` in the extension root (NOT in `.github/agents/` which is the repo's own dev agent). During `run.md` prerequisite validation, it copies to the target project's `.github/agents/`.

---

## R4: Command Pipeline Architecture

**Decision**: Two-command architecture with clear separation of concerns

### `speckit.ralph.run` (thin launcher)
- **Role**: User-facing entry point invoked via `/speckit.ralph.run` in an agent session
- **Behavior**: 
  1. Validate prerequisites (copilot CLI, tasks.md, git repo, feature branch)
  2. Ensure agent profile is placed in `.github/agents/`
  3. Detect platform (PowerShell or Bash)
  4. Locate orchestrator script in extension directory
  5. Launch script with configured parameters
- **Does NOT**: Contain loop logic, manage iterations, track progress

### `speckit.ralph.iterate` (single iteration)  
- **Role**: Agent-facing command invoked BY the orchestrator script for each iteration
- **Behavior**: Read tasks.md → identify first incomplete work unit → implement it → update tasks.md checkboxes → append to progress.md → commit if work unit complete
- **Invoked by**: `copilot --agent speckit.ralph -p "Iteration N"` from the orchestrator script

### Pipeline Flow
```
User ──→ /speckit.ralph.run ──→ validate ──→ ralph-loop.ps1/sh
                                                    │
                                              ┌─────┴─────┐
                                              │  Loop N×   │
                                              │            │
                                              │  copilot   │
                                              │  --agent   │
                                              │  speckit.  │
                                              │  ralph     │
                                              │     │      │
                                              │  iterate   │
                                              │  command   │
                                              │     │      │
                                              │  tasks.md  │
                                              │ progress.md│
                                              └────────────┘
```

**Rationale**: This matches the spec clarification Q4 (thin launcher) and Q2 (dual invocation paths). Users can also bypass the agent entirely and run scripts directly from terminal.

**Alternatives Considered**:
- Single command with embedded loop: Rejected — violates Q4 decision; can't run directly from terminal
- Three commands (run, iterate, status): Rejected — status can be read from progress.md; third command adds complexity without value

---

## R5: Config System Design

**Decision**: YAML config template with environment variable overrides

**Config File**: `ralph-config.template.yml` → installed as `.specify/extensions/ralph/ralph-config.yml`

**Schema**:
```yaml
# Ralph Extension Configuration
# DO NOT store authentication tokens here!
# Use environment variables: GH_TOKEN or GITHUB_TOKEN

# AI model for agent iterations
model: "claude-sonnet-4.6"

# Maximum loop iterations before stopping
max_iterations: 10

# Path to agent CLI binary (default: searches PATH for 'copilot')
agent_cli: "copilot"
```

**Loading Precedence** (per Extension Dev Guide § Config Loading):
1. Extension defaults (`extension.yml` → `defaults`)
2. Project config (`.specify/extensions/ralph/ralph-config.yml`)
3. Local overrides (`.specify/extensions/ralph/ralph-config.local.yml` — gitignored)
4. Environment variables (`SPECKIT_RALPH_MODEL`, `SPECKIT_RALPH_MAX_ITERATIONS`, `SPECKIT_RALPH_AGENT_CLI`)
5. Script parameters (highest priority — CLI flags always win)

**Rationale**: Follows the extension config pattern from the dev guide. The warning against storing tokens satisfies FR-015. Environment variables for auth satisfy spec clarification Q1.

---

## R6: Porting the Iterate Command

**Decision**: Adapt `templates/commands/ralph.md` from spec-kit core into `commands/iterate.md`

**Key Changes**:
1. **Script reference**: The frontmatter `scripts` field uses relative paths from extension root. After registration, these resolve to core spec-kit scripts:
   ```yaml
   scripts:
     sh: ../../scripts/bash/check-prerequisites.sh --json --require-tasks --include-tasks
     ps: ../../scripts/powershell/check-prerequisites.ps1 -Json -RequireTasks -IncludeTasks
   ```
   These paths become `.specify/scripts/bash/check-prerequisites.sh` and `.specify/scripts/powershell/check-prerequisites.ps1` after registration (core spec-kit scripts).

2. **Agent profile reference**: The iterate command references `speckit.ralph` agent. Since the `run.md` command ensures the agent profile is placed, iterate can assume it exists.

3. **Content**: The iterate command body from `templates/commands/ralph.md` (121 lines) is mature and comprehensive. Port with minimal changes:
   - Update header description to reference extension context
   - Keep all scope constraints, outline steps, progress format, stop conditions, quality gates, and error handling

**Source File**: `C:\Users\Rubis\Projects\spec-kit\templates\commands\ralph.md`

---

## R7: Porting the Agent Profile

**Decision**: Port `speckit.ralph.agent.md` with minimal modifications

**Source**: `C:\Users\Rubis\Projects\spec-kit\.github\agents\speckit.ralph.agent.md` (184 lines)

**Changes Needed**:
1. **Script reference in step 1**: The agent profile references `.specify/scripts/powershell/check-prerequisites.ps1`. This is a core spec-kit script that exists in any initialized project. Keep unchanged.
2. **Work unit scope**: The agent profile defines work unit as "smallest of: one phase, one user story, one logical grouping." Keep unchanged.
3. **Extension context**: Add a note that this profile is provided by the ralph extension, but avoid breaking the prompt structure.

**Rationale**: The agent profile is the most critical piece — it defines exactly how the AI behaves during each iteration. Minimal changes reduce risk of behavioral regressions.

---

## R8: Testing Strategy

**Decision**: Manual integration testing with verification checklist

**Test Plan**:
1. **Manifest validation**: `specify extension add --dev` validates schema automatically
2. **Command registration**: `specify extension list` shows commands
3. **Dry run**: Install on a project with completed tasks.md, run one iteration
4. **Full loop**: Run 3-5 iterations on a small task set
5. **Cross-platform**: Test PowerShell on Windows, Bash on macOS/Linux
6. **Interruption**: Test Ctrl+C during active iteration
7. **Config**: Test with custom model, max iterations, agent CLI path
8. **Resume**: Interrupt and restart to verify pickup from checkpoint

**Rationale**: This is a script+command extension with no compilable code. Unit testing is limited value; integration testing is the primary validation mechanism. The spec-kit `ExtensionManifest` class provides schema validation.

**Future**: If the extension grows, add automated tests using the Python ExtensionManifest API (see Extension API Reference § Testing).
