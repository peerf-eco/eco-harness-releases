# Eco.AI Assembly Meta-Harness

Eco.AI Assembly is a cross-platform ACOM component assembly harness. It owns
the EcoOS domain tools, marketplace RAG, project-generation policy, bounded
role orchestration, human plan approval, and a shared event contract. Agent
backends are replaceable: the built-in agent, Pi, Codex, and Claude Code can
be selected per role.

See the [latest release notes](https://github.com/peerf-eco/eco-harness-releases/releases/latest)
for version-specific changes.

## Install

Two customer-grade flows, both one command, no source checkout, no wine.
The installer always downloads **native** eco-package / eco-wizard builds for
your OS (never wine), plus the prebuilt marketplace RAG index, verified
against the release manifest's sha256 checksums. Missing API keys never
block install or launch — finish configuration in the in-app `/setup`
wizard (or later in the app-home `.env`).

### Linux

```bash
curl -fsSL https://github.com/peerf-eco/eco-harness-releases/releases/latest/download/install.sh | sh
```

or download and inspect first: `curl -fsSLO https://github.com/peerf-eco/eco-harness-releases/releases/latest/download/install.sh && sh install.sh`.

- Requires `curl` or `wget` (everything else — including Python 3.11 — is
  provided by `uv` automatically, per-user, no sudo).
- Shims land in `~/.local/bin/eco-harness` and `~/.local/bin/eco-harness-update`;
  if `~/.local/bin` is not on your `PATH`, add it:
  `echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc` (or your shell profile).

### macOS

```bash
curl -fsSL https://github.com/peerf-eco/eco-harness-releases/releases/latest/download/install.sh | sh
```

Same as Linux; downloaded binaries are Apple-Silicon/Intel native (arm64 /
x86_64) and the installer strips the Gatekeeper quarantine attribute
(`xattr -d com.apple.quarantine`) so first launch is not blocked. Gatekeeper
may still warn on first run of an unsigned binary — right-click → Open, or
System Settings → Privacy & Security → Allow.

### Windows

From PowerShell:

```powershell
irm https://github.com/peerf-eco/eco-harness-releases/releases/latest/download/install.ps1 | iex
```

or from cmd.exe:

```bat
powershell -Command "irm 'https://github.com/peerf-eco/eco-harness-releases/releases/latest/download/install.ps1' | iex"
```

or download and run: `powershell -ExecutionPolicy Bypass -File install.ps1`.
`uv` provides Python 3.11 if missing (3.12 on Windows ARM64 — see below); the app installs to
`%USERPROFILE%\ecoos\toolchain\eco-harness` and `eco-harness.cmd` /
`eco-harness-update.cmd` shims are added to your **user PATH** (available in
new terminals).

**Windows 11 ARM64** (e.g. an Apple-silicon Mac running a Windows 11 ARM64
VM) is fully supported, with no degraded features. The installer detects the
native architecture and prefers the `windows/arm64` build of each tool. Note
that the two tools differ there: **eco-wizard's `windows/arm64` is a native
ARM64 binary**, while **eco-package's is the x64 build republished under the
arm64 name** (GraalVM native-image and aws-crt-jni have no Windows ARM64
target yet) and runs through Windows 11's built-in x64 emulation —
the updater says so explicitly in its report rather than implying a native
install. The installer also selects **Python 3.12** here instead of
3.11 — PyYAML, a required dependency, publishes no `cp311` `win_arm64` wheel,
so 3.11 would fail the install outright. RAG vector search works too: because
upstream `sqlite-vec` has no `win_arm64` wheel, this host installs
`sqlite-vec-win-arm64`, the peerf-eco rebuild of the upstream amalgamation,
which provides the same `sqlite_vec` module. See `WORKING_DOCUMENTATION.md`
§18.10.

### After install (all native OSes)

```bash
eco-harness                 # serve the UI + API on http://localhost:8000
eco-harness-update          # self-update (wheel + binaries + RAG index)
# health report (the shim wraps serve; call the installed venv for doctor):
"$HOME/ecoos/toolchain/eco-harness/venv/bin/python" -m eco_harness doctor
```

Open http://localhost:8000 — the setup wizard at `/setup` walks you through
the OpenRouter key (or **Skip — add later in `.env`**; the app runs degraded
but starts regardless).

### Update notifications (optional)

Set `CHECK_UPDATES=true` in the app-home `.env` (or the process environment)
to have the harness check the public release manifest at startup and on every
new session, throttled by a six-hour cooldown. When a newer core, a changed
harness-managed `eco-package` / `eco-wizard` binary (sha256/zip sha256), or a
refreshed RAG index is published, the UI header shows **Install Update**.

- The check reads the same `manifest.json` as the installer and
  `eco-harness update`, so its diff can never disagree with what an install
  would change. User-owned tools (`ECO_PACKAGE` / `ECO_WIZARD`, custom paths,
  `PATH` finds) are reported but never overwritten, so they never trigger the
  notification.
- The **Harness updates** card at the top of **Settings** always shows the
  current version and has a manual **Check for Updates** button; manual checks
  work even when `CHECK_UPDATES` is unset (an explicit click is consent).
- **Install Update** runs `eco-harness update --json` as a background
  subprocess in the harness venv: wheel, managed binaries, RAG index and the
  `framework/devkit` mirror, each only when its version/sha256 actually
  differs. A wheel update needs a harness restart, which the UI prompts for;
  the running process is never restarted automatically and live sessions are
  not interrupted.
- Docker installs cannot run the native updater inside the container. The UI
  still reports availability and tells you to run `docker compose pull`.
  `CHECK_UPDATE` (singular) is accepted as an alias for older `.env` files.

Installed layout (Eco platform standard, see `env.example`):
`ECO_HOME` (default `~/ecoos`) is the ecosystem root, `ECO_TOOLCHAIN` =
`$ECO_HOME/toolchain`, and the app home `ECO_HARNESS` =
`$ECO_TOOLCHAIN/eco-harness`:

```text
~/ecoos/                          # ECO_HOME — Eco OS ecosystem root
├── toolchain/                    # ECO_TOOLCHAIN
│   ├── eco-package/              # ECO_PACKAGE  (native binary + bundle)
│   ├── eco-wizard/               # ECO_WIZARD
│   ├── eco-idl/                  # ECO_IDL      (future)
│   └── eco-harness/              # ECO_HARNESS — this app's home:
│       ├── venv/                 #   Python 3.11 venv (uv-managed)
│       ├── bin/                  #   fallback binary dir + Windows shims
│       ├── data/                 #   marketplace_index.sqlite + marketplace_cache/
│       ├── .env                  #   OPENAI_API_KEY, ECO_API_TOKEN, … (chmod 600)
│       ├── workspace.yaml        #   UI settings overrides
│       ├── output/               #   generated projects + session registry
│       └── traces/               #   per-session LLM traces
├── framework/devkit              # ECO_FRAMEWORK
├── framework/runtime             # ECO_FRAMEWORK_RT
├── worktrees/                    # Docker Worktree sessions (ECO_WORKTREE_ROOT)
└── workspace/                    # ECO_PROJECTS_DIR — your projects
```

eco-package / eco-wizard are downloaded ONLY when not already present: the
installer checks `ECO_PACKAGE` (legacy `ECO_CLI`) / `ECO_WIZARD` env vars and the
standard locations (the pre-rename `eco-cli` spellings count as present too)
first, keeps user-installed copies as-is (version-checked — outdated ones get
a replacement recommendation instead of an overwrite; `PATH` finds are never
probed/executed), and installs fresh downloads into
`$ECO_TOOLCHAIN/<tool>`, falling back to the app-home `bin/` folder when
those directories cannot be created. Harness-managed copies (carrying the
`.<tool>.version` marker) are refreshed in place by `eco-harness-update`.
When `ECO_PACKAGE` or `ECO_WIZARD` is supplied in the installer environment, the
installer intentionally persists that value into the app-home `.env`; this
keeps the explicitly selected user-owned binary winning on later launches
and avoids silently switching versions.

Uninstall: remove the app home (`~/ecoos/toolchain/eco-harness`, Windows:
`%USERPROFILE%\ecoos\toolchain\eco-harness`; pre-standard installs:
`~/.eco-harness`) and the shim files / user-PATH entry. The installer-managed
tool directories are sibling paths (`$ECO_TOOLCHAIN/eco-package` and
`$ECO_TOOLCHAIN/eco-wizard`); remove those only when they are not shared with
other Eco tools. Preserve `$ECO_HOME/framework` and `$ECO_HOME/workspace` if
other Eco components or projects use them. For a complete Eco installation
removal, delete `$ECO_HOME` only after confirming it contains no other data.

### Docker (any host with Docker)

```bash
# Linux / macOS (note `-s --` to pass flags through the pipe):
curl -fsSL https://github.com/peerf-eco/eco-harness-releases/releases/latest/download/install.sh | sh -s -- --docker

# Pick the target the image's toolchain builds for (image-backed targets only):
sh install.sh --docker --target linux/x86_64
sh install.sh --docker --target Linux · arm64      # aliases / display spellings ok
sh install.sh --docker --image ghcr.io/peerf-eco/eco-harness:0.1.7-13.2.0-x86_64-debian-x86_64
```

```powershell
# Windows — irm | iex cannot take flags; download first, then run:
curl.exe -fsSLO https://github.com/peerf-eco/eco-harness-releases/releases/latest/download/install.ps1
powershell -ExecutionPolicy Bypass -File install.ps1 -Docker
powershell -ExecutionPolicy Bypass -File install.ps1 -Docker -Target linux/x86_64
```

Installs the release `docker-compose.yml` (fetched from the same public
release; single source of truth with the checked-in `docker/docker-compose.yml`)
into the app home and writes your absolute host paths into the adjacent
`.env` — compose interpolates the volume sources from it, while the
container keeps the standard paths. Then it pulls the **per-target** harness
image from ghcr.io, starts it, and downloads the native binaries and RAG
index from the same public GitHub release (sha256-verified, exactly like the
native flow) into the mounted ecosystem root, which the container sees at the image's
`ECO_HOME` (`/home/user/ecoos`); tools live under `toolchain/<tool>` and the
harness app home at `toolchain/eco-harness`. The in-container update uses the public
release manifest by default — no S3 or extra configuration involved.

**Target selection.** Each published harness image layers the runtime on the
matching public toolchain base
(`ghcr.io/peerf-eco/toolchain:<toolchain_version>-<target_arch>-<distro>-<host_arch>`),
so the baked compiler builds for exactly one target. The tagged image name
encodes it: `<harness_version>-<toolchain_tag>` (e.g.
`0.1.7-13.2.0-x86_64-debian-x86_64`). With no flag the installer maps the host
architecture onto the matching Linux image; `--target <os>/<arch>` selects
another image-backed target (aliases like `x64`/`aarch64` and display
spellings work). A target with no published image stays native-only: the
installer prints the image-backed list and exits rather than silently running
an emulated x86 build. `--image <repo:tag>` bypasses the matrix entirely
(air-gapped / mirror installs). On an interactive terminal with no flag the
installer offers a numbered picker; `curl … | sh` (non-TTY) always uses the
host default without prompting.

If the UI requests a target the running image does not bake, the harness
returns a structured mismatch the chat renders as a dialog: switch the image
(exact command shown), continue with the available image target (clamped and
mirrored into the platform dropdown), or install natively and use
`setup_toolchain`.

```bash
# update the image (re-resolves the same target tag):
docker compose -f ~/ecoos/toolchain/eco-harness/docker-compose.yml pull && \
docker compose -f ~/ecoos/toolchain/eco-harness/docker-compose.yml up -d
# switch the install to a different image-backed target:
sh install.sh --docker --target linux/arm64
# update the binaries + RAG index (re-runs the manifest-driven download):
docker compose -f ~/ecoos/toolchain/eco-harness/docker-compose.yml exec eco-harness python -m eco_harness update
# health:
docker compose -f ~/ecoos/toolchain/eco-harness/docker-compose.yml exec eco-harness python -m eco_harness doctor
```

### Key paths / env vars in installed mode

| What | Where |
|---|---|
| Ecosystem root (`ECO_HOME`) | `~/ecoos` (`toolchain/`, `framework/`, `workspace/`) |
| App home (`ECO_HARNESS`) | `$ECO_HOME/toolchain/eco-harness` (`venv/`, `bin/`, `data/`, `.env`, `output/`, `traces/`) |
| User project (`ECO_PROJECT_DIR`) | picked in the UI folder browser (or pre-set with `--project-dir`); defaults to `ECO_PROJECTS_DIR` (`$ECO_HOME/workspace`) when it exists |
| Binary lookup order | explicit → `ECO_PACKAGE` (primary) / `ECO_CLI` (legacy alias) / `ECO_WIZARD` env → `$ECO_TOOLCHAIN/<name>` → app-home `bin/` (`.exe` probed on Windows) → `PATH` |
| Shared project toolchains | `ECO_TOOLCHAIN_CACHE` (Compose: `/data/eco-toolchains`, persistent named volume) |
| Prebuilt index | `$ECO_HARNESS/data/marketplace_index.sqlite` (refreshed by `eco-harness-update`) |
| Update checks | `CHECK_UPDATES=true` (automatic; startup + new session, 6 h cooldown); cached in `$ECO_HARNESS/update-state.json` |
| Config file | `$ECO_HARNESS/.env` (or `$ECO_HARNESS/workspace.yaml` for UI settings) |

For the packaging internals (wheel contents, manifest contract, hosted
artifacts, CI pipeline) and the go-live checklist (§18.9) see
`WORKING_DOCUMENTATION.md` §18.

## Current runtime

The production path is:

```text
Next.js UI → FastAPI /ws/chat → architect → plan approval → coder ↔ tester
                         └────── marketplace RAG, eco-package, eco-wizard
```

The WebSocket path is `/ws/chat`. New code should use the neutral
`eco_harness` modules and the shared chat event contract.

## Requirements

- Python 3.11+
- Node.js and npm for the UI
- Docker Desktop or Docker Engine for the compose deployment
- An OpenRouter-compatible API key for the internal agent and embeddings
- `eco_package` (the `eco-package` binary) for marketplace discovery and component downloads
- `eco-wizard` for generated project/component structure
- Optional external sub-agents installed on `PATH`:
  - `codex exec --sandbox workspace-write --skip-git-repo-check -` (prompt on stdin)
  - `pi -e "prompt"`
  - `claude -p` (adapter adds `--bare`, JSON output, `dontAsk`, and an explicit
    tool list; needs `ANTHROPIC_API_KEY`)

If a configured external executable is missing, the selected role fails with
an actionable error. The harness does not silently fall back to another agent.

The Architect owns marketplace discovery and project-local `eco-package pull`
materialization. The Coder consumes those paths, uses `eco-wizard`'s exact
entry/build paths, and may not use `marketplace_cache` as a build dependency.
Structured `eco-package` calls use `--output=json`; the wrapper normalizes the
unsupported `--json` spelling as a safety net.

### Executable Resolution

All external tool binaries (eco-package, eco-wizard, and the optional external
sub-agents) are resolved by one shared policy —
`eco_harness/agent/internal/tools/binaries.py::resolve_binary`:

1. explicit argument (harness.yaml `eco_*_path` settings, tool args)
2. `ECO_PACKAGE` (primary) / `ECO_CLI` (legacy alias) / `ECO_WIZARD` env (the `_PATH` spells are no longer honored)
3. `$ECO_TOOLCHAIN/<name>` — the standard platform location (binary inside
   the tool directory or the file itself)
4. `<repo>/bin/<name>` — the canonical, gitignored dev home / installed-mode
   app-home `bin/` fallback (the Windows `.exe` spelling is required)
5. System `PATH` (called as-is)

No other locations are probed. The dev container reads `<repo>/bin/` (the
same canonical home) through the monorepo bind-mount at `/app`, so neither
host nor container needs env vars; use `ECO_PACKAGE` (or the legacy `ECO_CLI`) / `ECO_WIZARD`
in `.env` only for custom locations.

Native installs never need wine: the installer downloads the target-OS build
into `$ECO_TOOLCHAIN/<name>` and falls back to the app-home `bin/` directory.
`ECO_PACKAGE_PREFIX` (legacy `ECO_CLI_PREFIX`) / `ECO_WIZARD_PREFIX`
wine wrappers remain available only as a legacy override.

**When to use the env vars:** custom executable locations or pinning a
specific version. Values must be **absolute** paths (no `~` expansion, no
repo-root anchoring) and are interpreted by the resolving process — a host
path for host runs, a container path (e.g. `/app/...`) inside the api
container. For normal setups just drop the binaries into `<repo>/bin/`
(dev), use the standard toolchain directories, or let the installer
provision them — no config is needed.

## Setup Details

### Environment Configuration

1. **Copy environment template:**
   ```bash
   cp env.example .env
   ```

2. **Edit `.env` file:** Set at minimum:
   - `OPENAI_API_KEY` - Your OpenRouter API key
   - `OPENROUTER_URL` - OpenRouter API URL (default: `https://openrouter.ai/api/v1`)
   - `ECO_API_TOKEN` - Eco marketplace token (for `fetch_marketplace.py`)
   - `DEEPSEEK_API_KEY` - optional; enables DeepSeek-backed web search/fetch
     in **Settings → Web**

### Environment Variable Matrix

Canonical reference for every environment variable the harness reads.
`.env` is loaded via python-dotenv; variables already set in the shell are
not overridden.

| Variable | Read by | Default | Purpose |
| --- | --- | --- | --- |
| `OPENAI_API_KEY` | embedder, providers | — | OpenRouter API key (RAG embeddings + LLM calls + OpenRouter web fallback) |
| `OPENROUTER_URL` | providers | `https://openrouter.ai/api/v1` | OpenRouter endpoint |
| `OPENROUTER_ALLOW_FALLBACKS` | providers | `true` | Allow another provider for the same model when a pinned provider is unavailable; set `false` for strict pinning |
| `DEEPSEEK_API_KEY` | web tools | — | DeepSeek Search/Fetch key (same key as the DeepSeek model endpoint) |
| `HARNESS_WEB_PLUGINS` | web registry | empty | Comma-separated allowlist of external `eco_harness.web_providers` entry points; built-ins always load |
| `LLM_MODEL` | config loader | `tencent/hy3` | Default model id (sets/overrides the `default` profile). The older `tencent/hy3-preview` slug no longer routes on OpenRouter — use `tencent/hy3`. |
| `ECO_API_TOKEN` | fetch_marketplace.py, eco-package | — | Eco marketplace token |
| `ECO_PACKAGE` | binary resolution | `$ECO_TOOLCHAIN/eco-package` → `<repo>/bin/eco-package` → PATH | eco-package binary or tool dir — PRIMARY spelling (`eco-cli` was renamed to `eco-package`) |
| `ECO_CLI` | binary resolution | same as `ECO_PACKAGE` | legacy alias of `ECO_PACKAGE` (still honored) |
| `ECO_WIZARD` | binary resolution | `$ECO_TOOLCHAIN/eco-wizard` → `<repo>/bin/eco-wizard` → PATH | eco-wizard binary or tool dir (legacy `ECO_WIZARD_PATH` still read, deprecated) |
| `ECO_IDL` | binary resolution (reserved) | `$ECO_TOOLCHAIN/eco-idl` → `<repo>/bin/eco-idl` → PATH | future eco-idl binary or tool dir (legacy `ECO_IDL_PATH` deprecated — warns only) |
| `ECO_PACKAGE_PREFIX` | eco_package tool | — | Wrapper command for Windows binaries under Linux (e.g. `wine64`); legacy alias `ECO_CLI_PREFIX` |
| `ECO_WIZARD_PREFIX` | eco_wizard tool | — | Same, for eco-wizard |
| `ECO_WIZARD_TIMEOUT_S` | eco_wizard tool | `180` | Scaffold generation timeout |
| `ECO_TOOLCHAIN_TIMEOUT_S` | setup_toolchain | `120` | Maximum marketplace/package-manager operation timeout |
| `ECO_TOOLCHAIN_CACHE` | setup_toolchain/build env | unset outside Compose; `/data/eco-toolchains` in Compose | Shared installed toolchains reused across projects |
| `ECO_TOOLCHAIN_CACHE_CONTAINER` | Docker Compose interpolation | `/data/eco-toolchains` | Container path for the persistent named toolchain volume |
| `BASE_IMAGE` | Docker Compose build arg | `ghcr.io/peerf-eco/toolchain:13.2.0-x86_64-debian-x86_64` | Base image for `docker compose build api`; point it at a locally built image to test without publishing |
| `ECO_CLAUDE_PATH` / `ECO_CODEX_PATH` / `ECO_GROK_PATH` / `ECO_PI_PATH` | factory / ExternalCliBackend | PATH lookup | External sub-agent binaries; flags come from `config/agents/external/<name>.yaml` |
| `ECO_MAKE_EXE` | server chat handler | `make` | Path to the make binary used by builds |
| `MARKETPLACE_CACHE_ROOT` | paths.py consumers | `<repo>/marketplace_cache` → `$ECO_HOME/data/marketplace_cache` → `/app/marketplace_cache` | Pre-pulled component cache |
| `MARKETPLACE_INDEX_PATH` | paths.py consumers | `<repo>/marketplace_index.sqlite` → `$ECO_HOME/data/marketplace_index.sqlite` → `/app/marketplace_index.sqlite` | sqlite-vec RAG index |
| `ECO_FRAMEWORK` | paths/assembler, eco_package pulls, index builder, eco-wizard | `<repo>/eco_framework` | Standard ACOM framework root. Host/installed layouts may be versioned DK trees; the Compose dev service points it at the flat read-only `marketplace_cache` layout expected by wizard templates. |
| `HARNESS_OUTPUT_ROOT` | server | `<repo>/output` (dev) / `$ECO_HOME/output` (installed) | Where per-chat workspace dirs are created |
| `HARNESS_TRACES_DIR` | server | `<repo>/traces` (dev) / `$ECO_HOME/traces` (installed) | Per-conversation LLM trace folders |
| `CHECK_UPDATES` | update check | off | Set `true` to check the public release manifest at startup and on new sessions (6 h cooldown); `CHECK_UPDATE` is a legacy alias |
| `HARNESS_ALLOWED_ROOTS` | server: bootstrap static roots | home dir + `HARNESS_OUTPUT_ROOT` | Extra static directories (os.pathsep-separated) accepted as projects. Folders selected outside these roots in the UI are persisted under `trusted_paths` in `<output_root>/.harness-registry.json`, so adding a project does not require an API restart. Directory browsing itself is unrestricted; file enumeration and session/agent access remain bounded to allowed + trusted roots. |
| `HARNESS_MAX_HOPS` | orchestrator | `8` | Max handoff hops (also `harness.yaml.max_hops`) |
| `HARNESS_PLAN_HANDOFF_MAX_BYTES` | architect `to_coder` gate | `16384` (`harness.yaml.plan_handoff_max_bytes`) | Bytes-of-markdown cap on the architect's handoff to the coder. Plan validator BLOCKS the `to_coder` call when the plan exceeds this; raise for big multi-component apps, lower when targeting small-context models. |
| `HARNESS_CONTEXT_WINDOW` | context resolution | unset (provider lookup) | Optional fixed token-window override. Leave unset to track the provider; over-claiming causes HTTP 400. See [Context window resolution](#context-window-resolution). |
| `HARNESS_CONTEXT_LOOKUP_NETWORK` / `HARNESS_CONTEXT_CACHE_TTL` | context resolution | `1` / `86400` | Enable live provider window lookup and set how long its disk cache stays fresh. |
| `AGENT_MAX_ITERATIONS` | CLI and `/ws/chat` role construction | empty | Positive values replace every role's `max_iters`; empty, invalid, or nonpositive preserves role budgets. Not a ceiling. |
| `AUTO_RESUME` | internal orchestrator | `0` | Opt-in (`1/true/on/yes`): one coder continuation after `max_iters`, only with new successful write/build/smoke work and resume support; never scaffold-only/read-only spirals or errors. |
| `HARNESS_ELIDE_CONTEXT_RATIO` | internal agent | `0.4` | Retain tool evidence unchanged until estimated system + tools + history reaches this fraction of the actual model context window. |
| `HARNESS_REASONING_LEVELS_NETWORK` / `HARNESS_REASONING_LEVELS_CACHE_TTL` | `pi_ai/reasoning_levels.py`, server `GET /api/models/reasoning-levels` | `1` / `86400` | Live catalogue query for a model's `reasoning.supported_efforts` (what the settings panel's per-model reasoning selector offers) and how long its disk cache (`$ECO_HARNESS/data/model_reasoning_levels.json`) stays fresh. The agent's request path never queries — it reads the profile declaration, the registry, or that cache. |
| `HARNESS_ELIDE_MAX_BYTES` / `HARNESS_REVIEWER_ELIDE_MAX_BYTES` | internal agent / reviewer | unset | Optional fixed-byte retention overrides instead of model-relative pressure; newest three results stay available. Malformed reviewer-specific values are ignored, falling back to the global override if set, otherwise model-relative retention. |
| `HARNESS_REVIEW_BUNDLE_MAX_BYTES` / `HARNESS_REVIEW_BUNDLE_MAX_FILE_BYTES` | reviewer `review_bundle` | `131072` / `65536` | Byte caps for the one-call reviewer bundle. Files the total cap omits (or files a per-file cap truncates) are named in the result so the reviewer reads only the missing part. |
| `HARNESS_REVIEWER_REPORT_AFTER_OUTPUT_TOKENS` | reviewer `EcoAgent` | unset → 75% of the role output cap (`6144` for the shipped 8192 cap) | Absolute token override. When set, stop investigation once a review has spent that many completion tokens (hidden reasoning included); otherwise the trigger is 75% of the effective `maxTokens`, so it moves with a workspace/profile cap change. Later non-report tool calls are answered with a report-now instruction, so the verdict is delivered before the provider output limit truncates it. |
| `HARNESS_DYNAMIC_TAIL_ITEMS` | agent context | `5` (`harness.yaml`: 12) | Newest tool results kept verbatim in context |
| `HARNESS_RETAINED_TOOL_OUTPUTS` | agent context | `budgets.yaml.retained_tool_outputs` (**10**) | Wired alias controlling `max_tool_results` context retention. Raised from 5: a 5-result window is narrower than one diagnostic loop, and evicted results were re-read repeatedly (see WORKING_DOCUMENTATION.md §4). |
| `HARNESS_SOURCE_MAX_BYTES` | context assembler | `300000` | Cap on static source material in the system prompt |
| `HARNESS_PREPULL_FRAMEWORK` | explicit legacy scaffold | `1` | `0` disables framework pre-copy when `HARNESS_SCAFFOLD=1`; normal auto runs use eco-wizard |
| `HARNESS_TOOL_DEDUP` | read-only tool memo + build memo | on | `0` disables exact-repeat memo dedup and unchanged-revision `run_build` reuse (consistent name across code/.env/env.example) |
| `HARNESS_READ_INDEX` | agent prompt observation index | on | `0` disables the prompt-only `[HARNESS READ INDEX]` / `[HARNESS BUILD INDEX]` state. The indexes survive elision/compression but never enter the static system prompt. |
| `HARNESS_WARM_SEED` | server warm-retry | `0` | `1` enables warm retry seeds |
| `ESCALATE_ON_PERMISSION_ERRORS` | internal agents / WS | `0` (off) | `1` turns path/scope denials into an interactive **Allow once / Allow for session / Deny** card instead of an immediate tool error. Grants and denials are session-scoped and keyed by resolved real paths (symlink/traversal safe), shared across coder and reviewer resumes. Deny returns a scope error, is remembered for the session, and the agent continues. Harness-private session-state paths (`.eco-harness/runs`, `.harness-state`) are a hard boundary and are never escalatable. Does not apply to external CLI backends. The effective flag is exposed at `GET /health` and in `GET /config` → `permissions.escalate_on_permission_errors`, and the Settings → Access panel shows it. |
| `HARNESS_SCAFFOLD` | server scaffold | `0` | `1` explicitly enables the legacy src/EcoMain.c + Makefile compatibility scaffold; normal auto runs must use eco-wizard |
| `FILE_SEARCH_BACKEND` | server `/api/fs/search` | `os_walk` | File-search backend for the `@`-mention picker: `os_walk` (zero-dependency recursive walk, default) or `fff` (opt-in `fff-search` wheel; automatically falls back to `os_walk` if the wheel is absent). The query `backend` param is only a hint. |
| `ECO_WORKTREE_ROOT` / `ECO_PROJECT_DIR` | worktrees / server | repo sibling `<repo>.worktrees` / unset (UI folder picker); production Compose: `/home/user/ecoos/worktrees` | Isolated git-worktree root / user project dir. Worktrees are grouped by base repo when this root is set. Installed native mode may leave it unset; Docker installs should keep it inside the mounted `ECO_HOME`. |
| `ECO_HARNESS_WORKSPACE_CONFIG` | config loader | `<repo>/.eco-harness/workspace.yaml` (dev) / `$ECO_HOME/workspace.yaml` (installed) | Workspace override file |
| `ECO_ROLE_<ROLE>_BACKEND` / `_MODEL` / `_REASONING` / `_MAX_TOKENS` | config loader | `roles.yaml` | Per-role overrides (e.g. `ECO_ROLE_CODER_BACKEND=pi`) |
| `DEFAULT_LANGUAGE` | config loader | `C` | Default implementation language |

Optional external env vars consumed indirectly by eco-package/eco-wizard:
`ECO_API_TOKEN`, `ECO_FRAMEWORK` / `ECO_FRAMEWORK_PATH` (framework DK path).

3. **Optional configuration notes:**
   - `MARKETPLACE_CACHE_ROOT` / `MARKETPLACE_INDEX_PATH` are normally NOT
     needed: on a host checkout the repo-root artifacts are detected
     automatically; in Docker the `/app` mounts are detected too; native
     installs find them under `$ECO_HOME/data/` (seeded by the installer).

### Initialization Scripts (Run on Host Machine)

**These scripts MUST run on the host machine BEFORE starting Docker containers:**

1. **Fetch marketplace components:**
   ```bash
   # Requires: ECO_API_TOKEN in .env
   # Downloads components to ./marketplace_cache/
   python scripts/fetch_marketplace.py
   ```

   By default only the **12 framework DevKit components** are downloaded
   (`Eco.Core1`, `Eco.InterfaceBus1`, `Eco.MemoryManager1`,
   `Eco.FileSystemManagement1`, `Eco.System1` and the seven `Eco.*.C89`
   libraries). Every other published product is **metadata-only**: its profile
   is still written to `marketplace_cache/_profiles/`, so `search_marketplace`
   and `read_component_profile` see the whole catalog, and `eco-package` pulls
   the component on demand when a session actually needs it. The cache feeds
   the installer distribution, so its download size is deliberately bounded —
   this is why a `SKIP` line is normal and not an error. Set `FETCH_TOOLCHAINS=1`
   to download everything instead.

   ```bash
   # Refresh the cache (replaces stale component trees and repairs missing
   # ones; components already at the advertised version are left untouched):
   python scripts/fetch_marketplace.py --force

   # Refresh only specific components:
   python scripts/fetch_marketplace.py --force-only Eco.Core1 Eco.Math.C89
   ```

   > **You need `--force` after anything is republished upstream.**
   > `eco-package` refuses to overwrite files that are already present, so a
   > cache holding an older release cannot refresh itself. Without `--force`
   > the fetch reports `stale_blocked` and **fails** rather than silently
   > leaving stale content behind. With `--force`, only trees that differ
   > from the catalog — or are missing/empty — are replaced; components
   > already at the advertised version are left untouched.

   **Reading the output.** Every run ends with a status legend and writes
   `marketplace_cache/_fetch_summary.json`:

   | Status | Meaning |
   |---|---|
   | `ok` | Downloaded or replaced. `files=` is **that component's** file count. |
   | `skip` | Left as-is. Either metadata-only, or already fetched at this version, or already present and `eco-package` declined to overwrite it. **`skip` is not proof the content is current.** |
   | `stale_blocked` | The cache holds the component but `eco-package` will not overwrite it without `--force`; the bytes on disk may be from an **older** version. **Hard failure** — re-run with `--force`. |
   | `pull_fail` | `eco-package` returned a non-retryable error. |
   | `parse_fail` | The catalog profile could not be parsed. |
   | `download_empty` | Pull reported success but the component tree is empty. |

   Check what a cache actually holds against the marketplace at any time:

   ```bash
   python - <<'PY'
   import json
   d = json.load(open("marketplace_cache/_fetch_summary.json"))
   for r in d["results"]:
       if r["status"] in ("ok", "stale_blocked", "pull_fail"):
           print(f"{r['name']:26} {r['status']:14} {r.get('version')}")
   PY
   ```

2. **Build RAG index:**
   ```bash
   # Requires: OPENAI_API_KEY in .env
   # Creates marketplace_index.sqlite from marketplace_cache/
   python scripts/build_marketplace_index.py
   ```

   This is a **no-op when `marketplace_index.sqlite` already exists** (it logs
   `Re-run with --rebuild`). After refreshing the cache you almost always want:

   ```bash
   python scripts/build_marketplace_index.py --rebuild   # wipe + re-embed
   ```

   For routine updates (and what **Settings → RAG → Update Index** does), use
   a merge: it embeds only new/changed chunks, so an unchanged corpus makes no
   embedding calls at all.

   ```bash
   python scripts/build_marketplace_index.py --merge
   ```

   If the embedding endpoint is unreachable, the Settings update automatically
   retries in lexical-only mode and reports a warning. From the CLI:

   ```bash
   python scripts/build_marketplace_index.py --merge --lexical-only
   ```

   That refreshes FTS5/BM25 search; the next embedding-capable merge backfills
   the missing vectors for those chunks.

   **Settings → RAG → Embeddings model** selects the provider (from the
   Providers tab) and the embedding model id. The base URL and credential are
   resolved from that provider, so a provider edit applies here automatically.
   Changing the model requires a full re-embed; the RAG **Update Index** button
   detects the model change and rebuilds the index with the new model.

3. **Publish the bundle (maintainers):**
   ```bash
   # fetch -> build -> package -> upload to S3/CloudFront.
   # --force-fetch refreshes the cache first; --rebuild re-embeds the index.
   # Without --rebuild the zip pairs the CURRENT cache with the PREVIOUS index,
   # and the script prints a warning saying so.
   # publish_rag.py auto-selects <repo>/.venv/bin/python for itself and its
   # fetch/build children when available, so plain `python` works too.
   python scripts/publish_rag.py --force-fetch --rebuild
   python scripts/publish_rag.py --no-upload          # package only
   ```

4. **Preflight check (optional but recommended):**
   ```bash
   # Validates setup before starting containers
   python scripts/dev_preflight.py
   ```

**Note:** These scripts interact with external APIs and download files to the host filesystem. They cannot run inside containers because:
- Need access to host filesystem for `marketplace_cache/`
- May need to download executables or large files
- Some require API tokens that shouldn't be in container images

### Executable Preparation

Place the binaries in the canonical gitignored home relative to project root:

```
bin/
├── eco-package(.exe)      # host-OS build for host-run scripts/agents
├── eco-wizard(.exe)         (eco-package.exe / aws-crt-jni.dll on Windows)
├── eco-package            # Linux ELF — used inside the dev container
├── eco-wizard             # (same files serve both on Linux/macOS hosts)
└── libaws-crt-jni.so      # Linux sidecar JNI lib, next to the ELF binaries
```

Both platform flavors may sit side by side: Windows host runs resolve
`eco-package.exe` first, the dev container resolves the extensionless Linux ELF
through the monorepo bind-mount at `/app` (`<repo>/bin/` is visible
in-container at `/app/Eco.Toolchain/Eco.AI.Assembly1/bin/`) — so both host
and container resolve without any env vars and without dedicated mounts.
JNI sidecar libraries must sit NEXT TO their executables.

### Docker Compose Deployment

**Prerequisites (on host):**
1. `.env` file configured
2. `marketplace_index.sqlite` created (via `build_marketplace_index.py`)
3. `marketplace_cache/` directory populated (via `fetch_marketplace.py`)
4. Binaries placed in `<repo>/bin/` (host build `eco-package(.exe)` /
   `eco-wizard(.exe)` plus the Linux ELFs for the container; the container
   reads them through the monorepo bind-mount — no env vars, no dedicated
   mounts)

**Start the application:**
```bash
docker compose up --build
```

**Access endpoints:**
- Web UI: http://localhost:3100
- API: http://localhost:8100
- WebSocket: ws://localhost:8100/ws/chat

**Volume Mounts in Docker Compose:**
- `../..:/app:rw` - the **entire monorepo** (the dir owning `.git`) as one
  unit, so the harness's git operations (`git worktree add`, etc.) resolve.
  Mount the repo root, not `.git` alone. `working_dir` points at the nested
  project, so the code is still reached correctly. Relative to this compose
  file, so it works on any machine with the same checkout layout.
- `${ECO_WORKTREE_HOST_DIR:-../../../Eco.AI.worktrees}:/repo-worktrees:rw` -
  host worktree root; paired with `ECO_WORKTREE_ROOT=/repo-worktrees` so
  created worktrees persist on disk. Default lands next to the monorepo root,
  same as harness host runs; override per machine via `.env`. Note: host
  `git worktree list` shows these entries as *prunable* (container-side paths
  are recorded) — see the Worktree-mode setup note below.
- When the dev stack is started from a **linked Git worktree**, the normal
  `../..:/app` mount does not include the main checkout's `.git/worktrees/`
  metadata. Use the optional
  `docker-compose.linked-worktree.yml` override with `ECO_GIT_COMMON_DIR` set
  to the main checkout's absolute Git common directory, or run Compose from
  the main checkout. See the Worktree isolation edge-case table below.
- The dev compose mounts the same host `./output` at two paths (`/app/.../
  output` through the monorepo mount and `/home/user/ecoos/workspace`).
  Register projects through `/app/...` when enabling Worktree: Git treats the
  two paths as different repositories, and the harness now rejects a mismatched
  base instead of creating the wrong worktree.
- `marketplace_index.sqlite` (read-write) plus read-only `eco_framework` and
  `config` overlays are mounted at their nested paths on top of the monorepo
  mount (`config/:ro` keeps agent write tools away from prompts/permission
  baselines; the UI settings pane writes `.eco-harness/workspace.yaml`, which
  stays writable).
- **The EcoOS platform contract comes from the toolchain image, not from
  `environment:`.** The dev compose deliberately does **not** re-declare
  `ECO_HOME` / `ECO_TOOLCHAIN` / `ECO_FRAMEWORK` / `ECO_PROJECTS_DIR`; the image
  exports them and the dev artifacts are mapped onto those canonical paths:

  | Container path (from the image) | Host source | Why |
  |---|---|---|
  | `$ECO_FRAMEWORK` = `/home/user/ecoos/framework/devkit` | `./marketplace_cache` | the pre-pulled devkit snapshot (`Eco.Core1`, `Eco.System1`, …) |
  | `$ECO_PROJECTS_DIR` = `/home/user/ecoos/workspace` | `./output` | where the UI creates projects |

  Overriding these in `environment:` is what previously desynchronised the
  image contract from the real filesystem and broke every generated build
  (see WORKING_DOCUMENTATION.md §19 "Toolchain path contract"). Mount them
  instead of re-pointing the variables.
- `eco-package` / `eco-wizard` (and their JNI sidecar libraries) have **no dedicated
  mounts**: they are consumed from `<repo>/bin/` — the canonical gitignored
  home — through the monorepo mount above. The image's `ECO_PACKAGE` /
  `ECO_WIZARD` point at `$ECO_TOOLCHAIN/{eco-package,eco-wizard}`, which a
  toolchain image does not ship; a set-but-missing env path is treated as
  "fall through to the standard locations", so the resolver picks `<repo>/bin/`
  with no configuration.
- `marketplace_cache/` is mounted **read-write** as the devkit, solely so
  `fix-c89-headers.sh` can run against it at startup. That script is a
  **temporary stopgap** for two published component headers that used `//`
  comments while the generated `MakefileExe` compiles with `-std=c89`; it is
  idempotent and re-applied on every start. It is now a **no-op** — those
  headers are fixed upstream — and should be deleted together with its
  `docker/entrypoint.sh` and `docker-compose.yml` invocations.
- `eco-toolchain-cache` is mounted at `/data/eco-toolchains` and is used by
  `setup_toolchain` for cross-project installation reuse. Project-local
  `.toolchain/env.json` files still record the build environment for each
  project; the installed packages and markers live in the shared volume.
- `api.environment` also sets `GIT_CONFIG_*=safe.directory` for `/app`,
  `$ECO_PROJECTS_DIR` and `/repo-worktrees` so root-in-container can operate on
  the host-owned repo (alternatively run as your host uid — see the commented
  `user:` line in the compose).
- **Build against a locally built toolchain image.** `docker compose build api`
  uses whatever `ghcr.io/peerf-eco/toolchain:<tag>` is present locally, so after
  `docker-image-factory`'s `./publish.sh --arch x86_64 --local` (which tags the
  same name it does not push) no extra flag is needed. Pin it explicitly with
  `docker compose build --build-arg BASE_IMAGE=<image>`, or set `BASE_IMAGE` in
  the adjacent `.env`.

**Important:** The RAG index (`marketplace_index.sqlite`) is mounted read-write because the UI can update it through import functionality. The component cache (`marketplace_cache/`) is read-only as it contains pre-downloaded components.

## Configuration

Repository configuration is under `config/`:

### Path Resolution System

One shared resolver (`eco_harness/agent/internal/tools/binaries.py::resolve_binary`)
handles every external binary, host and container alike:

1. Explicit config (e.g. `harness.yaml` `eco_package_path` / `eco_wizard_path`)
2. `ECO_PACKAGE` (primary) / `ECO_CLI` (legacy alias) / `ECO_WIZARD` env (the `_PATH` spells are no longer honored)
3. `$ECO_TOOLCHAIN/<name>` (standard platform location, e.g.
   `~/ecoos/toolchain/eco-package` — binary inside the dir or the file itself)
4. `<repo>/bin/<name>` (canonical, gitignored dev home / installed-mode
   app-home `bin/` fallback; `.exe` on Windows)
5. System `PATH`

**Environment Variable Examples:**
```bash
# Canonical home (recommended):
export ECO_PACKAGE=$PWD/bin/eco-package
export ECO_WIZARD=$PWD/bin/eco-wizard

# Docker: no env vars needed — the container reads <repo>/bin/ (Linux ELF
# builds) through the monorepo bind-mount (see docker-compose.yml).

# Windows executable via wine:
export ECO_PACKAGE=/path/to/eco-package.exe
export ECO_PACKAGE_PREFIX=wine64

# Custom path (development)
export ECO_PACKAGE=/usr/local/bin/eco-package
export ECO_WIZARD=/home/user/tools/eco-wizard
```

**Default `.env` configuration:**
```
# Host and container: leave unset when the binaries are in <repo>/bin/ or on
# PATH (the container reads <repo>/bin/ through the monorepo mount).
# Override only for custom locations:
# ECO_PACKAGE=/usr/local/bin/eco-package
# ECO_WIZARD=/home/user/tools/eco-wizard
```

| File | Purpose |
| --- | --- |
| `harness.yaml` | defaults, cache limits, hop limits, executable paths |
| `models.yaml` | named model profiles |
| `roles.yaml` | backend, model profile, reasoning, tools, budgets, per-role permission baseline |
| `languages.yaml` | supported languages and language skill profiles |
| `prompts/` | stable role and framework prompt fragments |
| `skills/` | ACOM, generator, and language skills |
| `tools.yaml` | timeouts and output limits |
| `marketplace.yaml` | cache/index and framework component settings |
| `ui.yaml` | UI defaults and selector options |
| `security.yaml` | security policy defaults |

Model selection (important): the working chat model is `tencent/hy3` — the older
`tencent/hy3-preview` slug was removed from OpenRouter's standard routing (it now
only exists on Tencent's own TokenHub endpoint). The provider **pin** is resolved
per model, with this precedence (workspace profiles merge over
`config/models.yaml` per key; a workspace `models: {name: null}` entry removes a
profile):

```text
model profile's provider_pin (workspace models > config/models.yaml)  >  OPENROUTER_PROVIDER_PIN (.env)
```

So each role's model carries its own `provider_pin` (see `config/models.yaml`,
e.g. `reasoning_heavy.provider_pin: tencent`), keeping the pin matched to that
model's provider and avoiding mismatched-pin 404s when different roles use
different models. A profile that omits `provider_pin` (the `default` profile)
falls back to the global `OPENROUTER_PROVIDER_PIN` env — that is the
`LLM_MODEL` + `OPENROUTER_PROVIDER_PIN` "default combination" for roles with no
specific model set. `allow_fallbacks` defaults to **True**, so a
missing/unavailable pinned endpoint routes to the other providers that also
serve the model (Tencent Cloud, DeepInfra, NovitaAI, …) instead of failing with
HTTP 404 "No endpoints found". Set `OPENROUTER_ALLOW_FALLBACKS=false` only if you
want strict single-provider pinning (availability traded for cache warmth).
**Every model's pin must resolve to a real OpenRouter provider for that model** —
pinning a model no provider serves (e.g. `deepseek/...` under a `tencent` pin)
makes OpenRouter 404. Each role's `per_query_tokens` budget is sent
as `max_tokens`; a context-window-aware clamp in
`eco_harness/agent/pi_ai/providers/openai_completions.py` caps it to fit the model context once
the system prompt is included, preventing HTTP 400 overflow. `harness.yaml:
source_roots` / `max_source_bytes` (300000) now drive the curated `Eco.Core1`
stitch (see context injection below).

Live vs baked config: the **entire monorepo** is bind-mounted into the api
container (see `docker-compose.yml`), so edits to `./eco_harness`,
`./config`, and `./scripts` apply on uvicorn reload / next request (dev
compose). `working_dir` targets the nested project dir.

Precedence is:

```text
environment variables > .eco-harness/workspace.yaml > config/*.yaml > code defaults
```

UI role settings are stored in `.eco-harness/workspace.yaml` (written by the
Settings panel via `PUT /config/workspace`; sections a client does not send —
e.g. `harness` — are preserved, and the `models`/`providers` sections merge
per key so a save never wipes user-defined profiles). The Web tab writes only
the top-level `web:` section through `PUT /api/web-tools/config`, preserving
every other section. Keep API keys and secrets in `.env` or a secret manager,
never in YAML.

Each role can select an internal or external backend:

```yaml
roles:
  architect:
    backend: internal
    model: reasoning_heavy
    reasoning: high
  coder:
    backend: codex
    model: coding_balanced
    reasoning: medium
  tester:
    backend: pi
    model: cheap_fast
    reasoning: low
```

Model profiles may use a named profile or a provider model ID. Per-role
budgets include token limits, iteration limits, wall-clock limits, and
optional cost ceilings.

## Settings panel

The gear icon opens the settings sidebar with six tabs. Most tabs save into
`.eco-harness/workspace.yaml` (the workspace override layer) and are picked up
by new sessions — running pipelines keep the config they started with. One
**Save settings** button persists the shared tabs; **Discard** reloads the
server state. **Web** is self-contained and has its own **Save web settings**
button (see below).

### Roles — per-role backend, model, reasoning

Each role (architect, coder, tester, reviewer) is a card with three selects:

- **backend** — `internal` or an external CLI (`pi`, `codex`, `claude`, `grok`)
- **model** — any profile from the model registry, or **Custom…** for an
  ad-hoc `provider/model-id` string (no registry entry needed)
- **reasoning** — the levels the **selected model** actually accepts, fetched
  from the provider's catalogue: `minimal` (thinking off) plus the model's own
  efforts, e.g. `high, max` for DeepSeek V4.1 — not one fixed ladder shared by
  every model. Overrides the profile default. A level the model does not offer
  stays visible with its translation spelled out (`xhigh → max`) so an existing
  `workspace.yaml` value never silently changes meaning. See
  [Reasoning effort resolution](#reasoning-effort-resolution).

### Models — the model registry

Lists every configured model profile with provider/reasoning/PIN badges and
which roles use it. From here you can add, edit, or remove profiles:

- **Add model** takes a profile name, the model ID (`provider/model-id`), a
  provider, the default reasoning level, an optional OpenRouter **provider
  PIN** (a routing hint — which upstream provider serves the model, e.g.
  `tencent`; *not* a secret), and optional `max_tokens` / `temperature`.
- Profiles still owned by `config/models.yaml` can be edited (the edit is
  persisted as a workspace override) or deleted (persisted as a removal
  marker). A profile referenced by a role cannot be deleted.
- The `default` profile always exists — deleting it resets it to the
  `LLM_MODEL` env value.
- The reasoning selector under each profile offers the levels that model
  accepts (fetched, see [Reasoning effort resolution](#reasoning-effort-resolution)).
  For an endpoint whose catalogue publishes none — `api.deepseek.com` — fill in
  the profile's **Supported efforts** field (or `reasoning_efforts` in
  `.eco-harness/workspace.yaml`): `high, max`. A new profile is seeded with a
  supported level once the catalogue/declaration answers; a supported level
  you pick is kept until you change it (or change the profile's model). Tick
  **Model requires reasoning** when the endpoint rejects reasoning-off.

### Access — LLM permissions

Permissions are bound to **roles**, not models: models are interchangeable
capability providers, roles are the harness's trust boundary. There are two
layers, resolved per role (later wins, key-by-key):

```text
code defaults < workspace defaults < repo roles.yaml per-role baseline < workspace per-role override
```

- **Harness defaults** — ten tool groups plus a command allowlist applied
  to every role:

  | Group | Gates |
  | --- | --- |
  | File read | `grep` `glob` `read` (line ranges) `bytes_seek` `read_file` `list_dir` `read_component_profile` |
  | File write | `write_file` `patch_file` `edit_file` |
  | Build | `run_build` (make) |
  | Run binaries | `run_artifact` |
  | RAG search | `search_marketplace` |
  | Skills | `read_skill` |
  | Network | `eco_package` `eco_wizard` |
  | Toolchain | `setup_toolchain` (requires both network and toolchain permission) |
  | Web search | `web_search` (DeepSeek/OpenRouter search) |
  | Web fetch | `fetch` (read one URL locally, via DeepSeek, or via OpenRouter) |

  The **command allowlist** narrows execution tokens: make targets (or
  `make` for a default build), artifact basenames for `run_artifact`, and
  eco-package subcommands and `setup_toolchain`. `*` (default) allows
  everything; an **empty list** denies all gated commands.

- **Per-role overrides** — a matrix of the same groups per role. Toggle a
  cell to deviate from the defaults; a `reset` link clears a role's
  deviations. Handoff/stop tools (`to_*`, `fail`) are pipeline plumbing and
  are never gated.

Enforcement is real: denied tools are removed from the role's toolset before
the system prompt is built (the model never sees them), and gated execution
tools are wrapped with the allowlist check (`eco_harness/permissions.py`).
It is a policy layer on top of the tools' own sandboxes (project_dir
containment, eco-package subcommand whitelist), not a replacement for them —
and it applies to **internal** backends only: external CLI backends
(pi/codex/claude/grok, marked `ext` in the matrix) run in their own process
with their own tool policy.

When `ESCALATE_ON_PERMISSION_ERRORS=1`, a path/scope denial becomes an
**Allow once / Allow for session / Deny** card in chat instead of an
immediate tool error. Grants and denials are session-scoped, survive
coder/reviewer resumes, and are visible in the Access panel. Harness-private
session state (`.eco-harness/runs`, `.harness-state`) can never be granted.

### RAG — index import, export, status

Shows live index stats (chunks, size, last import), imports files/folders,
and **exports** `marketplace_index.sqlite` for backup or another workspace.
See [RAG import and export](#rag-import-and-export) for formats and endpoints.

### Web — search and URL fetch for the Architect

Off by default. Enabling the master switch adds `web_search` and `fetch` to
the Architect (Plan phase and pipeline plan phase) when the role uses an
internal backend. The tab controls the mode (`snippets` / `hybrid` /
`managed`), ordered provider fallbacks, local extraction limits, SSRF/domain
policy, and the DeepSeek key. Default chain: DeepSeek Search → OpenRouter →
active LLM providers; fetch: local extraction → DeepSeek Fetch → OpenRouter
Web Fetch. Exa/Tavily/Firecrawl/Parallel/TinyFish are MCP-only and are not
direct provider choices. Changes apply to the next message; running sessions
keep the toolset they started with. See the
[web tools guide](https://github.com/peerf-eco/Eco.AI/blob/main/Eco.Toolchain/Eco.AI.Assembly1/docs/WEB_TOOLS.md)
for privacy, trust-model, and troubleshooting details.

### Trace metrics and Run health

Open the **Trace browser** from a project or session menu and switch to the
**Metrics** tab to inspect the selected session: calls and failures, per-stage
and per-model timing, token usage, list cost, retry and handoff counts, and
cache/prefix-reuse health. The tab loads aggregates lazily, so opening the
browser does not parse every call trace.

Completed pipelines also show a **Run health** card in chat with the run-level
outcome, retry pressure, and cache signals. Per-run and per-session rollups are
written to `<project_dir>/.eco-harness/runs/ses-<8hex>/` as `metrics.json` and
`metrics-run-<start>-<last>.json`; they contain aggregates only, never prompts,
transcripts, or tool output.

## Language and platform selection

The UI exposes `C`, `CPP`, `Python`, and `Java` beside the platform selector.
Language prompt and skill profiles are selected from `config/languages.yaml`
and `config/skills/languages/`. Python and Java layouts are intentionally
delegated to the upcoming `eco-wizard` release rather than invented by the
model.

## RAG import and export

The Settings panel (RAG tab) shows live index stats — chunk count, size, and
the last import summary — and accepts individual files, browser-selected
folders, source documents, Markdown/text documentation, and compatible SQLite
dumps. Imports update `marketplace_index.sqlite` through
`scripts/import_rag.py`; **Export** downloads the current index (a valid
SQLite file that can be re-imported as a dump or dropped into another
workspace as-is).

The API endpoints are:

- `GET /rag/status` — chunk count, size, last-import summary
- `POST /rag/import`
- `GET /rag/export`

For CLI use:

```cmd
python scripts/build_marketplace_index.py --rebuild
# Index straight from the ACOM development-kit tree ($ECO_FRAMEWORK) instead
# of a prior fetch into marketplace_cache/:
python scripts/build_marketplace_index.py --rebuild --source framework
python scripts/import_rag.py path/to/docs path/to/dump.sqlite
python scripts/export_rag.py --index marketplace_index.sqlite --out marketplace_index.team.sqlite
```

The current design keeps one shared marketplace index for team exchange.
Separate personal/project indexes and remote MCP-backed retrieval remain
future-compatible extension points.

## Generator and marketplace rules

Generated boilerplate must be produced by the local `eco-wizard` tool. The
internal coder receives the tool but not a template-generation instruction
that bypasses it. If a component is absent locally, the architect uses
`eco-package` to discover and pull it from the marketplace.

Normal auto-pipeline order is `eco_wizard → setup_toolchain → write/build →
test`. The wizard manifest, generated build files, and toolchain requirements
are authoritative. The Coder may add source/header/configuration files inside
existing wizard directories and edit an existing generated Makefile to include
them, but may not edit `.eco-wizard.json`, create new project directories, or
replace the generated build tree. A missing or unavailable toolchain is
escalated rather than solved by changing the manifest or inventing a host
version.

`eco_harness/backend/scaffold/` (shipped as wheel data) remains only as an
explicit compatibility fallback. It is never selected automatically. Set
`HARNESS_SCAFFOLD=1` only for an intentional legacy run; normal auto runs
must call `eco-wizard` and use its manifest-driven layout.

### marketplace_cache layout

`marketplace_cache/` (mounted read-only into the container at
`/app/marketplace_cache`) has two distinct parts the agents use differently:

- `<Component>/SharedFiles/*.h` — the **interface/source headers** (e.g.
  `Eco.Core1/SharedFiles/IEcoSystem1.h`). This is the readable corpus explored
  with `grep` / `glob` / `read`. Headers are UTF-8 (sometimes with a BOM) and
  may contain Cyrillic comments; the `read`/`read_file` tools decode them
  lossy-but-safe.
- `_profiles/<Name>.json` — one **metadata profile per component** (30 in the
  current snapshot), written by `scripts/fetch_marketplace.py`. Each holds the
  raw `find -n` profile: CID, available versions, DEVKIT file manifest, and
  dependencies. It answers "given this component name, what do I `pull`?" and
  is consumed by the `read_component_profile` tool. It is **excluded from the
  RAG index** (the directory begins with `_`), so `search_marketplace` never
  embeds it.

The buildable implementation (`.c` / `.lib` / `.so`) is **not** in the cache —
it is fetched on demand via `eco-package pull` into `project_dir`.

## Customization

The harness has a two-layer customization model: git-tracked repo defaults
under `config/` and an operator-owned workspace layer under `.eco-harness/`
(no repo changes needed, survives pulls). Everything is resolved at agent
construction; lower layers win only when the workspace layer is absent or
empty.

### Role prompts: `config/prompts/` vs `.eco-harness/prompts/`

Resolution (first non-empty wins):

```text
1. .eco-harness/prompts/<role>.md    workspace override — replaces entirely
2. config/prompts/<role>.md          editable source of truth
3. built-in constant                 fallback in eco_harness/agent/internal/agents/<role>.py
```

Create `.eco-harness/prompts/coder.md` and the coder runs with YOUR text
instead of the repo prompt. An empty or whitespace-only file is treated as
absent, so a stub can never blank out real instructions.

### Skills: same name overrides, new names must be declared

Skill bodies are searched with the **workspace first** (`.eco-harness/skills/`
→ `config/skills/`), name order `v<N>.md → SKILL.md → <skill>.md`:

- **Same name** → your workspace body replaces the repo skill
  (`.eco-harness/skills/acom_framework/v1.md` wins over
  `config/skills/acom_framework/v1.md`).
- **Different name** → it becomes available as an additional skill, but it
  does NOT self-activate. A skill only loads when listed in the merged
  `skill_versions` map (`roles.yaml` / `languages.yaml`, overridable per
  workspace).

To activate skills for a role, override its map in
`.eco-harness/workspace.yaml`:

```yaml
roles:
  coder:
    skill_versions:            # NOTE: wholesale replacement of the repo map
      acom_framework: "1"      # keep — body from .eco-harness/skills/ if present
      eco_wizard: "1"          # keep
      team_style: "1"          # add — .eco-harness/skills/team_style/v1.md
```

Only `budgets` blocks merge key-by-key; every other role/language key
(including `skill_versions`) replaces the repo value wholesale when set in
the workspace file.

### On-demand skills (Anthropic-style)

Large or rarely-needed skills can be made dynamic instead of always baked
into the system prompt. Create a `SKILL.md` with YAML frontmatter:

```markdown
---
name: my_skill
description: One line saying WHEN the agent should fetch this skill.
---
(free-form markdown body)
```

Behavior once referenced from `skill_versions`:

- The prompt receives only the one-line description in an
  `=== ON-DEMAND SKILLS ===` manifest (plus the source path).
- Internal agents fetch the full body on demand via the auto-wired
  `read_skill` tool; external CLI agents read the listed path themselves.
- A plain `v<N>.md` file (or frontmatter-less `SKILL.md`) keeps the classic
  always-injected behavior — both forms coexist.

First live example: `config/skills/component_author/SKILL.md` (~60 KB of ACOM
C templates) — the coder sees only its description until hand-authoring
component code without eco-wizard makes fetching worthwhile.

### Rules files

Project-wide rules go in the root `AGENTS.md`; all layers found are
concatenated (repo root first):

```text
config/agents/<role>/AGENTS.md
.eco-harness/agents/<role>/AGENTS.md
```

### Domain vs language rules split

`config/prompts/acom_domain.md` holds language-agnostic ACOM knowledge
(identifier taxonomy, framework stack, dev-kit boundary, EcoMain flow, trust
model) and is shared by every role and language. All language-specific coding
conventions live once in `config/skills/languages/<lang>.md`. When adding
rules, put them in exactly one place — the two cross-reference each other
instead of duplicating.

The legacy `agent/skills/` root was retired in PRD_2 Phase 2
(`agent/skills/c.md` → `config/skills/component_author/`; the file later
became the on-demand `SKILL.md`). The root-level `agent`/`backend`
directories are now back-compat import shims — real code lives under
`eco_harness/`.

  Language skills belong in `config/skills/languages/<language>.md`. Stable
  prompt changes belong in `config/prompts/`; workspace-specific instructions
  belong in `.eco-harness/`.

## Initial context injection

Every role's initial context (system prompt) is assembled once per agent
construction by `eco_harness/agent/context/assembler.py::build_static_system_prompt`
(driven by `eco_harness/roles.py`) from multiple sources, in a fixed order:

```text
# SYSTEM HEADER                          config/prompts/acom_system_header.md
=== STATIC ACOM DOMAIN KNOWLEDGE ===     config/prompts/acom_domain.md
=== STATIC TOOL CONTRACT ===             config/prompts/tool_contract.md
=== ROLE INSTRUCTIONS ===                composed from your selections:
                                           === MODE ===      prompts/modes/<mode>.md
                                           <role prompt>     precedence chain below
                                           <language prompt> prompts/languages/<lang>.md
                                           AGENTS.md layers + selected skills
                                           ROLE CONFIGURATION (backend/model/reasoning)
=== IMMUTABLE SOURCE CODEBASE ===        curated Eco.Core1/SharedFiles stitch
```

**How your selections drive it**

- **Mode** (`config/modes.yaml`): `auto`/`migrate` inject the pipeline-mode
  prompt and engage the full architect→approval→coder↔tester loop; `plan`,
  `code`, `test`, `review` inject their one-shot prompt and load exactly one
  role.
- **Active role** (`config/roles.yaml`): picks backend, model profile,
  reasoning level, budgets, and which skill profiles are injected.
- **Language** (`config/languages.yaml`): picks `prompts/languages/<lang>.md`,
  the language skill (`config/skills/languages/<lang>.md`), and the eco-wizard
  template family.

**Role-prompt precedence** (first non-empty wins):

```text
1. .eco-harness/prompts/<role>.md     workspace override, no repo changes needed
2. config/prompts/<role>.md           editable source of truth for each role
3. built-in constant                  fallback in eco_harness/agent/internal/agents/<role>.py
```

Empty or placeholder files are skipped, so they can never blank out real
instructions.

**Cache/token efficiency**: blocks 1–3 are byte-identical for every role,
mode, language, and backend; the Eco.Core1 stitch is C-only (`.h` headers,
`.hpp` wrappers excluded — ~29% smaller) and constant across turns and tasks;
only the ROLE INSTRUCTIONS block varies per selection
and stays stable while an agent iterates. This ordering maximizes provider
KV-cache reuse (measured: 99.6% cached tokens, −81% cost per call on a pinned
provider via `provider_pin`). Nothing dynamic — RAG results, tool outputs,
user requests — enters the system prompt; those live in message history where
older tool results are elided automatically. The harness-maintained
`[HARNESS READ INDEX]` and `[HARNESS BUILD INDEX]` messages are appended at the
prompt tail, after history and before the iteration status; they are never
written into stored history or the static cache prefix.

See `WORKING_DOCUMENTATION.md` §4 for the full developer-level contract
(skill resolution rules, artifact path resolution, maintainer rules).

Supporting changes that keep the architect fast and on-policy:

- **C language skill** (`config/skills/languages/C.md`) injects the full ACOM/C89/MISRA
  contract (EcoOS types, `IEcoMemoryAllocator1`-only allocation, UGUID byte format,
  mandatory Dev-Kit boundary, minimum stack, file/function-header discipline) into
  every C agent, so those conventions are no longer rediscovered from headers.
- **`read_component_profile` Contract Card** — returns the CID, IIDs, the
  `GetIEcoComponentFactoryPtr_<CID>` factory symbol, vtable method names, and the
  `SharedFiles/` layout in one structured call, avoiding large raw header reads.
- **`eco_package`** now auto-resolves the binary via the shared resolver
  (`ECO_*_PATH` → `<repo>/bin/` → `PATH`) and,
  when none is found, returns an actionable error noting the read-only
  `marketplace_cache` already holds the needed headers.

## Working modes

The UI mode selector and CLI (defined in `config/modes.yaml`) choose which
role(s) run and whether the automatic plan→implement→verify pipeline is engaged:

- `auto` (default) — the deterministic loop: **architect plans → human plan
  approval (HITM) → coder ↔ tester build & runtime-test**. A lightweight intent
  gate classifies each message first: plain questions are answered directly
  (no pipeline), while a concrete coding task triggers the loop.
- `plan` — architect only: research + PRD / closed plan. **No pipeline and no
  coder hand-off** — the plan is surfaced as the final answer so you can approve
  it later in `auto` / `code`.
- `code` — coder only: direct implementation. **No architect pass and no
  automatic test pipeline**; you drive each phase manually from the mode menu.
- `migrate` — the same `auto` pipeline, but with a migration-focused system
  prompt: analyze the existing codebase, divide it into reusable modules, map
  each to ACOM component contracts, then incrementally refactor.
- `test` — read-only testing agent.
- `review` — read-only ACOM style/correctness reviewer.

In `auto` / `migrate` the architect emits `plan_review_required` and the server
waits for your `plan_decision` before the coder runs (`eco_harness/backend/server.py`).
`plan` / `code` / `test` / `review` load exactly one role with its profile and
never auto-trigger the cross-role pipeline, so you can switch roles freely from
the mode menu. Each mode selects its own system prompt and capability set from
`config/modes.yaml`.

## Attaching files & @-mention

The chat input supports **session-scoped file attachments** so the agent can
read your files directly. Attachments ride on **every** message in the current
session (until you start a New Session) — they are not cleared after a single
send, so the planner *and* the coder both see them.

- **@-mention a file** — type `@` anywhere in the message to open a file
  autocomplete. It searches from the active project root (or your home when no
  project is selected) and inserts a `@relative/path` token. The file is added
  to the attachment list and the agent reads it by its real path.
- **`+` button** — opens a file picker rooted at the active project. Select one
  or many files across folders (selections accumulate as you navigate); they are
  attached by absolute path.
- **Paste / drag-and-drop** — paste or drop images and text. Pasted content is
  attached inline (images ≤ 5 MB; larger files should use the `+` picker).
- **Chips** — attached files appear as removable chips above the input. Remove
  any one with its `×`, or clear everything with **New Session** (which keeps
  your platform / language / mode / worktree / project settings).

> **How delivery works:** small text files are inlined into the prompt; large
> text files and images are copied into `project_dir/.eco-attachments/` and
> referenced by path so the agent can `read()` them on demand; files already
> inside the project are referenced by their real relative path (no copy).
> The total attachment payload per message is capped at ~12 MB to protect the
> WebSocket connection.

## Worktree isolation

Enable **Worktree** beside the chat input, or pass `--worktree` to a CLI
one-shot. The harness creates a **detached Git worktree** at the selected
project's repository `HEAD` and uses the corresponding project subdirectory as
`project_dir` for every agent read, write, build, and test in that session.
The primary checkout is never used as the write target, and worktree changes
are **not merged back automatically**.

### What gets copied into the worktree

There are two distinct cases:

| Selected project | Worktree contents |
|---|---|
| **Ignored project path** — matches `.gitignore` (for example a harness `output/<project>` directory), even when someone force-added a file under it | The **entire current project working tree** is snapshot-copied into the same relative path in the worktree. Hidden files and directories such as `.env`, `.eco-harness/`, `node_modules/`, `.venv/`, build outputs, caches, generated sources, and dependency trees are included. Nested `.git` directories are excluded. |
| **Not ignored, with tracked files** | The normal checkout of the repository at `HEAD`. Uncommitted edits, untracked files, and ignored files from the primary checkout are deliberately **not** copied. |
| **Not ignored, with no tracked files** | The current project working tree is snapshot-copied, as above. This covers new untracked projects that are not excluded by `.gitignore`. |

The snapshot copy is what makes the default development `output/proj-*`
projects usable: `git worktree add` alone only materializes tracked Git objects,
so without this step the agent would see an empty project directory. A forced-
added file under an ignored project path does not change that rule; the whole
project is still snapshotted so the agent does not lose the other working
files.

The snapshot is intentionally complete. It can therefore be large (for example
a JavaScript project's `node_modules/`) and it duplicates ignored secrets such
as `.env` into the worktree. Keep large generated trees out of the project or
use non-Worktree mode when that cost or duplication is unacceptable. For a
non-ignored tracked project, uncommitted changes and ignored dependency trees
are **not** copied; that worktree is a clean checkout of `HEAD`. The UI marks a
snapshot worktree with a **snapshot** badge and explains the contents in the
reference-strip tooltip.

### Repository selection and failure behaviour

- The worktree base is **always the repository containing the selected
  project**. There is no fallback to `ECO_PROJECT_DIR`, the default project, or
  another registered project.
- If the selected folder is not inside a Git repository, Worktree mode is
  rejected with a clear error. The UI disables the toggle when the project Git
  status reports `in_repo: false`.
- If the selected folder resolves to a different repository than the resolved
  base (for example through two Docker mount aliases for the same host
  directory), the request is rejected instead of creating a worktree for the
  wrong project.
- If worktree creation or project seeding fails, the partially created
  worktree is removed and the original error is reported. The primary
  checkout is not modified.

### Naming, location, and lifetime

When no explicit name is given, the directory is auto-named
`<repo>-<session8>-<commit7>` (for example
`Eco.AI.Assembly1-3f2b8c1a-d41cd09`). The name and path appear on the reference
strip below the progress bar as soon as the worktree is created and stay
visible until **New session**.

- `ECO_WORKTREE_ROOT` sets the destination root. UI sessions group worktrees
  under that root by base repository name, so projects with different repos do
  not share a working area.
- Without `ECO_WORKTREE_ROOT`, the default is a sibling of the repository:
  `<repo-parent>/<repo>.worktrees/<name>`.
- Worktrees persist until you remove them. Starting a new session with
  Worktree enabled creates a new worktree; an existing destination name is
  rejected rather than reused.
- The CLI uses the same creation routine but, for one-shot runs, points
  `project_dir` at the worktree root instead of a selected project subpath.
- A custom name is available through `--worktree-name` on the CLI and through
  `worktree_name` on the websocket request.

```cmd
python -m eco_harness run "Review this component" --mode review --worktree
python -m eco_harness /plan "Design a checksum component" --worktree
```

The CLI runs a single role one-shot (`plan|code|test|review`); the full
`auto`/`migrate` pipeline needs the human plan-approval gate and lives in the
UI / websocket API.

### Installed-mode and production notes

- **Native install:** worktree creation works from the installed wheel without
  a source checkout. Pick a project that is a Git repository (or a directory
  inside one). `ECO_WORKTREE_ROOT` is optional; set it to an absolute,
  writable path if the default sibling location is not appropriate for your
  backup or disk layout.
- **Harness-generated projects in installed mode:** the app-home
  `$ECO_HARNESS/output/` tree is not a Git repository by default. Worktree mode
  is therefore unavailable for those generated projects unless the project is
  placed inside a repository. Select a repository-backed workspace project, or
  initialise and commit a repository for that project first.
- **Production Docker install:** the release compose sets
  `ECO_WORKTREE_ROOT=/home/user/ecoos/worktrees`, inside the mounted
  `ECO_HOME`, so worktrees survive container recreation and updates. If you
  override it to a path outside that mount, add a matching host volume and a
  Git `safe.directory` entry for both the project and worktree roots.
- **Docker worktree visibility:** worktree metadata records the *container*
  path. On the host, `git worktree list` may show those entries as
  **prunable**. The files are still on disk under the host worktree directory.
  Do not run `git worktree prune` on the host; automatic `git gc` can also
  prune entries older than `gc.worktreePruneExpire` (default three months).
  Set `git config gc.worktreePruneExpire never` if host-side metadata must be
  retained. For fully valid host entries, mount the host worktree directory at
  the identical absolute path inside the container and point
  `ECO_WORKTREE_ROOT` at that same path.

### Edge-case handling guide

| Symptom / situation | Cause | What to do |
|---|---|---|
| Toggle is disabled or request says “not inside a Git repository” | The selected project is not in a repository, or Docker exposes it through a path alias that Git cannot relate to the base repo. | Select the project through the repository mount (in dev Compose normally `/app/...`), or initialise/commit it as a repository. Do not rely on the old default-project fallback; it no longer exists. |
| Worktree is created but the project looks empty | This was the old `output/` bug. Current builds snapshot-copy ignored projects, including ignored projects with a force-added tracked file. | Update to a harness version containing this fix. Verify the reference strip path and check that `output/<project>` exists inside it. A snapshot worktree also carries the **snapshot** badge. |
| A non-ignored tracked project lacks `.env`, `.venv`, `node_modules`, or other ignored files | Worktree is a clean checkout of `HEAD`; ignored files are not part of Git. | Commit the required files, or provide a repository-specific bootstrap/setup step outside Worktree mode. An ignored project path is different: its whole current tree, including those files, is snapshotted. |
| Local edits to a tracked file are missing | Worktree starts at the last commit, not at the dirty working tree. This applies to non-ignored tracked projects. | Commit or otherwise publish the changes before enabling Worktree. An ignored project path is snapshotted from the current working tree, and the strip shows the **snapshot** badge. |
| Docker Compose is launched from a linked Git worktree and `git worktree add` fails | The checkout's `.git` is a file pointing at the main checkout's `.git/worktrees/...`, which is outside the normal `/app` mount. | Run the dev stack from the main checkout, or use `docker-compose.linked-worktree.yml` with `ECO_GIT_COMMON_DIR` set to the main checkout's absolute Git common directory. Host-side uvicorn is an alternative, but it loses the container toolchain contract. |
| Host `git worktree list` reports a container-created worktree as prunable | Metadata contains the container path, not the host path. | Inspect the files under the host worktree root; do not prune. Use the identical-path mount recipe above if you need host-valid metadata. |
| Worktree directory exists after a failed run | A previous attempt created the destination before failing. | The current code removes partial worktrees on seeding failure. For old leftovers, remove them with `git worktree remove --force <path>` from the base repository, or delete the directory and run `git worktree prune` in that repository. |
| Two sessions on the same project need isolation | Each session gets a separate worktree. | Enable Worktree for each new session; names include the session id and commit prefix. Do not reuse a worktree directory name. |
| Worktree root is not writable in Docker | Container runs as root or a different uid than the host owner. | Pre-create the host worktree root as your user and/or run the container with your uid/gid; fix ownership before retrying. |

For a dev stack started from a linked worktree, the common-directory mount is:

```bash
MAIN_CHECKOUT=/path/to/main/Eco.AI
export ECO_GIT_COMMON_DIR="$(git -C "$MAIN_CHECKOUT" rev-parse \
  --path-format=absolute --git-common-dir)"
docker compose -f docker-compose.yml \
               -f docker-compose.linked-worktree.yml up --build
```

> **Dev Compose path-alias warning:** the dev compose mounts the same host
> `output/` directory both through `../..:/app` and at
> `/home/user/ecoos/workspace`. Git does not know those are the same directory.
> Prefer registering projects through the `/app/...` path. If a project is
> selected through the workspace alias, Worktree mode may correctly refuse it
> as “not in a Git repository” instead of creating a worktree for an unrelated
> base.

## Workspace and per-agent limits

### How the working folder is chosen

The workspace (`project_dir`) is chosen per chat session:

- **Explicit project:** register a folder through the UI (project panel /
  folder browser). In installed mode this sets `ECO_PROJECT_DIR` (pre-seed
  it at install time with `--project-dir`); the user's project lives under it.
  Isolated session worktrees are separate and follow `ECO_WORKTREE_ROOT` (or
  the repository-sibling default) as described above.
- **Default (no project registered):** `_output_root()` —
  `HARNESS_OUTPUT_ROOT` (default `<repo>/output` on a dev checkout,
  `$ECO_HOME/output` installed) + `proj-<thread8>`. A fresh sub-directory
  is created for each WebSocket session/thread, so concurrent chats never
  share a tree.
- **Any folder on the API host:** the project picker browses the server's
  filesystem without an allowlist. Selecting a folder outside the built-in
  roots (`$HOME`, `HARNESS_OUTPUT_ROOT`, `HARNESS_ALLOWED_ROOTS`) shows a
  grant warning; confirming persists the path in `trusted_paths` in
  `<output_root>/.harness-registry.json`. Every session of that project can
  then use it as `project_dir` without restarting the API. Trust is sticky —
  removing the project from the panel does not revoke it. Filesystem roots
  and OS system trees (POSIX `/`, Windows drive roots and `C:\Windows`, etc.)
  are refused as project folders.
  In the dev Compose stack the monorepo is mounted at `/app`, so host folders
  under the `Eco.AI` tree are visible and selectable as `/app/...`; host
  folders outside the mounts need one bind mount (a container cannot see
  unmounted host paths). File listing (`/api/fs/browse?files=1`), attachments
  and file search stay bounded to allowed + trusted roots.
- **Worktree (isolated):** enable **Worktree** in the UI (or `--worktree` on
  the CLI). The harness creates a detached Git worktree outside the primary
  checkout — under `ECO_WORKTREE_ROOT` — and uses the matching project
  subdirectory as `project_dir`. An entirely untracked/ignored project (such
  as the dev `output/proj-*` tree) is snapshot-copied into the worktree;
  tracked projects start at `HEAD`. The primary checkout is never modified.
  The selected folder must be inside a Git repository; there is no fallback to
  an unrelated default project.

All file tools are **sandboxed to `project_dir`** for writes: the coder's
`write_file` / `build` / `runtime` can only touch `project_dir`, while
`read` / `glob` / `grep` / `read_file` / `list_dir` may also read the
read-only `marketplace_cache`. UI role/model/language selections and the
permission policy are persisted in `.eco-harness/workspace.yaml` (see
[Settings panel](#settings-panel)).

### Per-agent space limits (budgets)

Each role's limits come from `config/roles.yaml` →
`roles.<role>.budgets` and are applied identically to every sub-agent
(architect, coder, tester, reviewer):

| Budget | Effect |
| --- | --- |
| `per_query_tokens` | Sent as the model `max_tokens`; a context-window-aware clamp in `eco_harness/agent/pi_ai/providers/openai_completions.py` caps it so the system prompt fits (prevents HTTP 400). |
| `max_output_tokens` | Optional positive internal-agent role ceiling on completion tokens, including hidden reasoning and tool arguments. Applied even when a selected profile sets a larger `max_tokens`; the reviewer default is 8192 with `reasoning: high`, and it is forced to report at 75% of that cap (`HARNESS_REVIEWER_REPORT_AFTER_OUTPUT_TOKENS` overrides) so the verdict is delivered before the provider truncates it. Explicit model/effort selections remain authoritative. External CLIs retain their own token controls. |
| `per_query_usd` / `per_day_usd` | Cost ceilings for a single query / per day. |
| `max_iters` | Hard cap on the agent's tool-call loop iterations. |
| `max_wall_s` | Wall-clock timeout for the agent. |

Global guards: `HARNESS_MAX_HOPS` (default `8`) bounds orchestrator
handoffs. A positive `AGENT_MAX_ITERATIONS` replaces ALL role budgets on
CLI and websocket paths. The shipped environment leaves it empty so
`config/roles.yaml` supplies architect=40, coder=60, reviewer=30, tester=20.
An explicit shell/container override still wins; restart/recreate the service
after changing its environment. Each `EcoAgent.run` has a fresh iteration
counter, including a resumed run, with one grace turn to consume final results.
File-tool results are also
size-capped (`read` line ranges with a 32 KB whole-file short-circuit and a
200 KB cap / 2 KB per line, `bytes_seek` 32 KB default / 200 KB max,
`read_file` 256 KB, `grep` 100 matches, `glob` 500 entries) so a single call
can't blow the context budget.

Code-review seeds append bounded plan contracts/static criteria, observed public
header declarations, the build record, and source byte sizes without expanding
read permissions. Resumed fix rounds refresh build/sizes without repeating static
contracts. Commands and link flags remain quoted from the coder card because
`last-build.json` does not store them. Library handoff checks inspect archive
declarations only in artifact/link sections, not prose; source-path gates share
annotation-aware parsing and name the exact failed paths.

### Context window resolution

The model's context window is **not** a fixed harness constant. It is resolved
once per agent (`eco_harness/agent/pi_ai/context_window.py`) and every budget
decision reads that one value: history elision (40% of the window), full-history
compression (60%), the `max_tokens` clamp in the OpenAI-completions provider,
and the session context-load gauge.

Resolution order — **first match wins**:

| # | Source | Notes |
| --- | --- | --- |
| 1 | `HARNESS_CONTEXT_WINDOW` env var | Explicit operator override. Beats everything, including live provider data. |
| 2 | Local registry (`pi_ai/models.py`) | Instant, no network. Only consulted for a registered id. |
| 3 | **Live provider query** | `GET https://openrouter.ai/api/v1/models`, `context_length` per id. Cached on disk at `$ECO_HARNESS/data/model_context_windows.json` with a TTL (`HARNESS_CONTEXT_CACHE_TTL`, default 24 h). The stale cache is still used when the network is down. |
| 4 | `FALLBACK_CONTEXT_WINDOW` (262 144) | Only when the override is unset **and** neither the registry nor a live/stale provider answer is available. |

**`HARNESS_CONTEXT_WINDOW` should normally be unset.** Leave it unset and the
window tracks whatever the provider reports, so a model that gains or loses
context between releases is handled without a code change. Set it only when you
want a deliberate, fixed budget — for example to make runs reproducible, or to
cap a model whose reported window is far larger than the hardware behind it.

If you do set it, note that the failure modes are **not symmetric**:

- **Under-claiming is safe.** Setting `250000` on a 1M model just compresses
  earlier than necessary. You lose some history, not correctness.
- **Over-claiming is not.** Setting a value *larger* than the real window means
  elision and compression fire too late, and the `max_tokens` clamp stops
  biting in time — the request goes out oversized and the provider answers
  HTTP 400 "maximum context length exceeded".

  The harness therefore **warns at startup** when `HARNESS_CONTEXT_WINDOW`
  exceeds what the provider reports. It warns and still honours your value; it
  does not silently clamp it.

Related knobs:

| Variable | Default | Effect |
| --- | --- | --- |
| `HARNESS_CONTEXT_LOOKUP_NETWORK` | `1` | Set `0` to forbid the live provider query. The disk cache is still consulted. |
| `HARNESS_CONTEXT_CACHE_TTL` | `86400` | Seconds a cached provider answer stays fresh. |

Startup logs the resolved value **and its source**, because a wrong window is
otherwise invisible:

```
[INFO] Effective model context window: 1048576 tokens (source: provider, provider[deepseek/deepseek-v4.1-flash-1017])
```

### Reasoning effort resolution

The reasoning selector is **per model**, not one hard-coded ladder: DeepSeek
V4.1 speaks `high` and `max`, hy3 tops out at `high`, and the OpenRouter
gateway accepts `minimal … max`. The same list drives the settings panel and
the value written into the request body — `pi_ai/reasoning_levels.py` resolves
it, `pi_ai/providers/simple_options.py` clamps onto it — so the two can no
longer disagree.

Resolution order — **first match wins**:

| # | Source | Notes |
| --- | --- | --- |
| 1 | `reasoning_efforts` on the model profile (Models tab / `workspace.yaml`) | Operator declaration, e.g. `reasoning_efforts: [high, max]`. The right answer for an endpoint whose catalogue publishes no `reasoning` block (`api.deepseek.com`) or for a local alias no catalogue knows. |
| 2 | **Live provider query** | `GET <provider>/models`, `reasoning.supported_efforts` per id (OpenRouter publishes it; the docs say "use this when building client UIs"). Cached on disk at `$ECO_HARNESS/data/model_reasoning_levels.json` with a TTL (`HARNESS_REASONING_LEVELS_CACHE_TTL`, default 24 h); the stale cache is still used when the network is down. |
| 3 | Local registry (`pi_ai/models.py`) | The value the entry was verified with — the offline answer, and what an agent's request path reads (it never blocks on a fetch). |
| 4 | Dialect default | OpenRouter-style endpoints: `low, medium, high, xhigh, max` (the gateway maps what the model lacks). Plain OpenAI-compatible endpoints: `low, medium, high` until they say otherwise. |

Whatever the source, a configured level is **clamped onto** that list before it
is sent (`simple_options.clamp_effort`): ties clamp *up* (`xhigh` on a
`high`/`max` model becomes `max` — you asked for more reasoning, not less), and
a level the harness does not recognise clamps *down* to the lowest supported
one. The model factory logs the resolved vocabulary, and the role prompt's
`ROLE CONFIGURATION` block reports the effort actually sent (e.g.
`reasoning=max` for a configured `xhigh`):

```
[INFO] Reasoning efforts: high, max (source: declared, workspace profile (deepseek-flash))
```

If the provider says the model **mandates** reasoning (`reasoning.mandatory`),
`minimal` is no longer offered for new selections; a configured `minimal`
stays visible as `minimal → <lowest>`. The profile's **Model requires
reasoning** checkbox (or `reasoning_mandatory: true` in `workspace.yaml`)
overrides the catalogue's answer; a configured `minimal` is then sent as the
model's lowest supported effort instead of `enabled: false`.

Related knobs:

| Variable | Default | Effect |
| --- | --- | --- |
| `HARNESS_REASONING_LEVELS_NETWORK` | `1` | Set `0` to forbid the live catalogue query. The disk cache is still consulted. |
| `HARNESS_REASONING_LEVELS_CACHE_TTL` | `86400` | Seconds a cached provider answer stays fresh. |

`GET /api/models/reasoning-levels?profile=<name>` (or `?id=<model-id>&provider=<p>`)
serves this resolution to the settings panel. When the Models tab is editing an
unsaved profile it adds `id`/`provider` to `profile`, so the response describes
the edited model while the profile's declaration is still applied client-side.

### Context compression strategy

The harness uses **selected / structural compression with a pinned head**, not
an LLM-written hand-off. Two layers, applied oldest-first:

**Layer 1 — result elision (at 40% of the window).** Individual large tool
results are collapsed in place to a stub naming what they were:

```
[older read path='SharedFiles/IEcoSimplexRBTree1.h' output elided for context
 budget; the full result is retained in the session trace. This is a
 CONTEXT-BUDGET eviction, not a truncated or failed tool call. Recover the
 named file with a focused read/read_file or grep using a basename-only glob
 and the original directory. An identical read is allowed again after this
 eviction.]
```

Only results ≥ `HARNESS_ELIDE_MIN_BYTES` (2 KB) are eligible, the most recent
`HARNESS_ELIDE_KEEP_RECENT` (3) are never touched, and the mutation is
**monotonic** — once elided, always elided, so the context prefix only ever
shrinks and never oscillates. Full output is still on disk in the session
trace.

**Layer 2 — full-history compression (at 60% of the window).** The old head
collapses to a single synthetic user message. What survives:

| Kept | How |
| --- | --- |
| The seed / initial task block | pinned **verbatim**, never digested |
| The latest user task block | pinned verbatim (a re-entry may send several adjacent user messages) |
| The last 4 messages | verbatim, extended back to a turn boundary so an assistant `tool_call` is never separated from its `toolResult` (which the provider APIs reject) |
| Everything between the pins | replaced by the digest |

The digest itself is **deterministic string extraction, not a model call** — no
LLM is invoked to write it. Per message it keeps up to 3 text blocks truncated
to ~500 chars and lists the tool *names* it called (up to 5), then caps the
whole digest at ~12 000 characters. Full tool output text and full IID bytes are
deliberately discarded, because they can be re-derived.

So the strategy is **re-seed on demand** rather than hand-off: the digest ends
with

> *"If you need full text of a specific tool result, re-call the tool with the
> same arguments and the result will be re-supplied."*

and the dedup memo is cleared at the same time so a re-call cannot be answered
from a memo pointing into content that is no longer in context.

**Anti-obsolescence indexes (prompt-only).** `_build_context` regenerates two
compact harness-maintained messages from in-memory state every turn:
`[HARNESS READ INDEX]` records each file already read (path, line/byte ranges,
read iteration, mtime/size freshness, and whether the result is still in
context or was elided), and `[HARNESS BUILD INDEX]` records the last
`run_build` result per input set with its `rc` and structured compiler
diagnostics. They survive eviction, compression and resumed histories because
they are not part of the transcript. The read index marks a file `unchanged`
only while its mtime/size still match; an edited file becomes `CHANGED since
read`. `HARNESS_READ_INDEX=0` disables them.

Exact-repeat dedup remains separate and revision-aware: a path-aware write
invalidates only the touched file's memo entries (`read_scaffold` is dropped
conservatively because its bundle key is implicit), and an identical
`run_build` at an unchanged observed source revision is answered from the
previous result instead of launching `make` again.

**Known consequence.** Because content is dropped, an agent that needs the
full text of an elided file still has to re-read it (the trace keeps the full
text). The index prevents the *redundant* re-ask when the file is unchanged
and the earlier result is still in context, and the build index prevents a
failed build from being re-run just to recover diagnostics that were elided.
A correct window is still the cheapest fix, which is why resolution is
provider-backed rather than fixed.

### Harness Improvement Plan And Validation

The October 2026 run report points primarily to latency and autonomy defects,
not unaffordable inference: the run passed at $0.082, but took 14m44s and a
manual continuation. The changes below target the causal failures without
assuming that a single provider-specific observation generalizes.

| Measure | Decision for this harness | Why / validation |
| --- | --- | --- |
| Restore role budgets | Implemented in deployment and both environment templates; retain documented explicit global override | Removes the accidental coder 60 -> 10 replacement rather than masking it with retries. Check effective prompt/run limits. |
| Automatic continuation | Implemented, OFF by default (`AUTO_RESUME=0`) | One shared-orchestrator coder continuation preserves history and workflow; fresh successful work is required. No second server retry engine, no model/settings change, no unbounded loop. |
| Phase-aware iteration nudge | Implemented at half the coder budget, or after three post-scaffold discovery calls without a write | Uses wizard/write/build/smoke workflow state in the existing request suffix, without altering the static cache prefix. The early trigger avoids waiting 30 turns on a 60-turn coder budget. This is guidance, not a hard guarantee that the model obeys. |
| Search contract | Implemented | `grep.glob` is basename-only, not a recursive path pattern; path separators are rejected. `path` is a directory, and `glob` is the separate path-pattern tool. A miss explains these semantics. |
| Authoritative signature answers | Implemented narrowly | Exact header identifier queries can reuse actual prototypes and header paths from the handoff; source, unknown, broad and deliberate `verify_handoff=true` searches still execute. Mere quoted mentions do not count as declarations. |
| Context retention | Implemented | Estimate the full request, including static tools, thinking and tool arguments, against `Model.contextWindow`. Default result eviction starts at 40%; emergency compression at 60% preserves the complete task seed and latest incoming user-message block (review/resume instructions) verbatim and in order. Older payloads remain eligible for lossy summaries. An estimate is not an exact provider tokenizer. |
| Recovery after eviction | Implemented | Prompt-only `[HARNESS READ INDEX]` keeps compact receipts (path, ranges, mtime/size freshness, elided state) across eviction, compression and resume; `[HARNESS BUILD INDEX]` keeps structured build diagnostics. Path-aware writes invalidate only the touched file's exact-repeat memo (`read_scaffold` conservatively), and an unchanged-revision `run_build` is answered from its stored result. Explicit byte overrides remain supported. |
| Reviewer completion | Implemented | 8192-token completion ceiling includes reasoning (`reasoning: high` by default); the reviewer bundles its file scope in one `review_bundle` call and is forced to report at 75% of the cap instead of waiting for a `length` stop. A normal stop without a report tool still gets one report-only retry within the remaining iteration budget, then fails honestly. Reviewer prose is never forwarded as a report. OpenAI-compatible `length` completions preserve their stop reason and fail closed, including pending tools. Selected model/effort remains user-controlled; ceiling changes participate in checkpoint drift detection. |
| EOF / malformed numeric input | Strengthened task/review contracts | Architect acceptance criteria must cover observable EOF and invalid-input behavior; reviewer checks `fgets` results and `strtod` end pointers. This does not claim static-analysis coverage. |
| Literal include-block equality | Deferred | Existing plan Id-header validation, wizard include validation and compiler checks cover distinct contracts. Blanket equality would reject legitimate platform-conditional or additional includes. A future source gate needs per-file scope/provenance, not a global regex. |
| Cache warm-up | Deferred, require billed A/B | An initial cache miss and a later hit do not prove an exact third-request rule. A warm-up itself pays for the static prefix and may need delay/another request; savings are unproven. |
| Rewrite old tool-call arguments | Deferred | It changes old prefix bytes and removes source evidence; a pointer to the current file is not necessarily the historical revision. Counting these arguments fixes pressure detection without changing their meaning. |
| Pin every role to HY3 | Not forced | Settings/workspace model overrides are intentional configuration. Routing to a cache-capable provider is an explicit operator choice; provider pinning alone does not guarantee cache hits. |
| Broad discarded-vtable-result gate / `gcc -fanalyzer` | Deferred | Void/Release calls are often legitimately discarded; indirect vtable calls and platform toolchains make generic regex/analyzer coverage unreliable. Trial diagnostics with the real generated compile commands before any hard gate. |

Automatic continuation emits a visible coder event and gets a fresh iteration
budget on the same agent, preserving its gates and accounting. The limit is one
per orchestrator invocation, independent of handoff hops; a subsequent failure
uses the existing pause/checkpoint/manual-resume path. Turning it on deliberately
increases the maximum work/spend allowed for a progressing coder.

Old result eviction is a context-pressure tradeoff: mutating a previously sent
message can invalidate that part of the provider prefix cache, even when done
only once. Under the threshold, history is left unchanged. The latest results
remain available; emergency compression can still discard historical details,
so recover them with the original tool/path rather than guessing. The
prompt-only observation indexes make that recovery decision explicit: reuse an
unchanged read, re-read only when the file changed or the text is needed, and
fix a failed build from its retained diagnostics instead of re-running it.

**Local regression validation:** use scripted models, temporary scaffolds, and
mocked providers. Cover budget precedence, output ceilings and drift, invalid
search patterns, exact prototype answers and bypasses, pagination/recovery,
real model windows, source arguments, preserved seed/tool-result grouping,
one-shot continuation, inherited/read-only observations, cancellation, and
websocket event forwarding. No production coding-session traces are needed.

**Effectiveness validation before claiming savings:** replay a small fixed task
set with the same model/provider/settings and explicit reviewer selection. Include
the reported numeric console task plus a multi-file/component task and a
build-fix task; use several runs per condition, comparing medians and tails.
Record time to first source, discovery calls before writing, manual pauses,
completion/quality rate, reviewer duration/output, total fresh/cache/output
tokens, all billed calls, and wall time. Require unchanged build/happy-path,
EOF, malformed-input and cleanup outcomes, no increase in false clean reviews,
and fewer discovery calls/pauses before enabling wider deployment. The 40%
retention threshold and 8192 reviewer ceiling are starting policies, not proven
optima; adjust only when replay measurements justify it.

For a later cache warm-up experiment, keep it default-off and provider-specific
override > global setting, not OR. Include warm-up usage in overall totals and
tag it separately; compare fully billed runs with/without warm-up and inspect
the first real call's cache-read/latency. Do not exclude warm-up cost, infer
cache capability from a provider pin, or assume different roles share a prefix.

## CLI and future MCP use

The headless entrypoint is:

```cmd
python -m eco_harness run "Design a calculator component" --mode plan --language C
python -m eco_harness run "Implement the approved plan" --mode code --language C
python -m eco_harness serve            # UI + API on http://localhost:8000
```

One-shot modes only (`plan|code|test|review`); use the UI / websocket for the
full `auto` pipeline with plan approval.

`eco_harness.adapters.AgentBackend`, `eco_harness.tools.ToolRouter`,
`eco_harness.interfaces.mcp`, and `eco_harness.extensions` are stable
boundaries for future CLI products, MCP servers, AST endpoints, and UI
panels. The current FastAPI server is an optional interface over those
boundaries, not the domain core.

## Releases

Releases are cut from `main` by pushing a `v*` tag. The workflow builds the
wheel and per-target container images, then publishes the public release in
[`peerf-eco/eco-harness-releases`](https://github.com/peerf-eco/eco-harness-releases)
— the only install channel used by the one-liners above. Each release ships the
wheel, installers, `manifest.json`, `docker-compose.yml`, per-platform
eco-package/eco-wizard zips, the RAG bundle, and a `README.md` copy.

Release notes are written in `docs/releases/<tag>.md` in the source repo and
become the release body. See the
[latest release notes](https://github.com/peerf-eco/eco-harness-releases/releases/latest)
for what changed. Images are published as
`ghcr.io/peerf-eco/eco-harness:<version>` plus a per-target
`<version>-<toolchain_tag>` tag.

To verify a build without publishing, run the release workflow manually with
`dry_run=true` (default); `dry_run=false` publishes only a `dev-<sha>` image.
A real release requires a `v*` tag on `main`.

## Troubleshooting

### Install, launch, and usage

1. **`eco-harness` is not found after install**
   - Open a new terminal so the updated PATH is loaded.
   - Linux/macOS: add `~/.local/bin` to `PATH` if the installer reported it
     missing.
   - Windows: the shim is in your user PATH; open a new PowerShell/cmd window.
   - For a full health report, run the installed venv's Python:
     `"$HOME/ecoos/toolchain/eco-harness/venv/bin/python" -m eco_harness doctor`
     (Windows: `%USERPROFILE%\ecoos\toolchain\eco-harness\venv\Scripts\python.exe -m eco_harness doctor`).

2. **The UI starts but model calls fail or setup is incomplete**
   - Open `http://localhost:8000/setup` and add an OpenRouter key, or set
     `OPENAI_API_KEY` in `$ECO_HARNESS/.env` and restart.
   - Updates, RAG search, and local tools still work in degraded mode; model
     calls need a provider key.

3. **Port 8000 is already in use**
   - The installed shim already runs `serve`, so use `eco-harness --port 8080`
     (source checkout: `python -m eco_harness serve --port 8080`), then open
     `http://localhost:8080`.

4. **No update notification / Install Update is missing**
   - Set `CHECK_UPDATES=true` in `$ECO_HARNESS/.env`, or use the
     **Harness updates** card at the top of **Settings** → **Check for
     Updates** (manual checks work without the flag).
   - Docker installs cannot self-update; run
     `docker compose -f ~/ecoos/toolchain/eco-harness/docker-compose.yml pull`
     then `... up -d`.

5. **Web search or fetch is missing**
   - Enable it in **Settings → Web**, save, and send a new message.
   - The Architect must use an internal backend. Web tools are Architect-only
     by default, and `web_search` / `fetch` must be enabled in
     **Settings → Access**.
   - DeepSeek search/fetch needs `DEEPSEEK_API_KEY`; OpenRouter uses the normal
     `OPENAI_API_KEY` / `OPENROUTER_URL`.

6. **Worktree toggle is disabled, or the worktree is empty**
   - The selected project must be inside a Git repository.
   - Ignored project paths are snapshot-copied; tracked projects start from
     `HEAD`, so commit local edits first.
   - Harness-generated projects under `$ECO_HARNESS/output` are not Git
     repositories by default. See [Worktree isolation](#worktree-isolation).

7. **An agent reports a permission/scope error**
   - With `ESCALATE_ON_PERMISSION_ERRORS=1`, use the **Allow once / Allow for
     session / Deny** card in chat.
   - Otherwise register the folder as a trusted project or adjust the role's
     matrix in **Settings → Access**.

8. **HTTP 404 "No endpoints found"**
   - The model profile's `provider_pin` must name a provider that serves that
     model. Remove the pin or set `OPENROUTER_ALLOW_FALLBACKS=true` to let
     OpenRouter route to another provider.

9. **HTTP 400 "maximum context length exceeded"**
   - Leave `HARNESS_CONTEXT_WINDOW` unset so the provider-reported window is
     used; if you set it, never claim more than the real model window. See
     [Context window resolution](#context-window-resolution).

### Development and Docker

10. **"eco-package not found" error**
    - Place the binary at `<repo>/bin/eco-package` (canonical home;
      `eco-package.exe` on Windows).
    - In Docker, verify the Linux ELF builds sit in `<repo>/bin/` (reached via
      the monorepo bind-mount); `ECO_PACKAGE` / `ECO_WIZARD` are only for custom
      locations.
    - Run `python scripts/dev_preflight.py --fix` for the full lookup order.

11. **GPG signature errors during Docker build**
    - Update the Dockerfile base image.
    - Clear the Docker build cache: `docker builder prune -a`.
    - Rebuild: `docker compose build --no-cache`.

12. **RAG index not found**
    - Run `python scripts/build_marketplace_index.py` on the host.
    - Ensure `marketplace_index.sqlite` exists in the project root (host and
      container runs auto-detect it there; set `MARKETPLACE_INDEX_PATH` only
      for custom locations).
    - Check file permissions.

13. **Marketplace components missing**
    - Run `python scripts/fetch_marketplace.py` on the host.
    - Verify `ECO_API_TOKEN` in `.env`.
    - Check network connectivity. Re-run with `--force` after an upstream
      republish; `stale_blocked` is a hard failure, not a normal skip.

14. **Windows executables not working in Linux container**
    - Ensure `wine64` is installed in the Docker image.
    - Set `ECO_PACKAGE_PREFIX=wine64` in `.env` for Windows executables.
    - Prefer native Linux executables; native installs never use wine.

### Debugging

- Check container logs: `docker compose logs api`
- Enter container shell: `docker compose exec api bash`
- Test eco-package in container: `docker compose exec api bin/eco-package --version`
- Verify volume mounts: `docker compose exec api ls -la bin/`
