# Project Context

## Purpose

fx is a tiny, open, embeddable coding-agent harness and CLI. It is optimized for
research, integration into larger systems, fast startup, a small native binary,
and a Unix-shell-like interactive experience rather than a terminal IDE. The
product is model-agnostic, supports local and cloud inference, and exposes
native CLI, Agent Client Protocol (ACP), WebAssembly, and Node-API embedding
surfaces.

The project is experimental and Apache-2.0 licensed. The canonical repository
is `vercel-labs/fx`.

## Tech Stack

- Zig 0.16.0 or newer for the product, build graph, unit tests, benchmarks, and
  native, WebAssembly, and Node-API artifacts
- Zig standard library and libc, with no third-party Zig package dependencies
- Bun and TypeScript for deterministic end-to-end tests and live model evals
- JavaScript ES modules for the experimental `libfx` SDK and host adapters;
  Node.js 20 or newer is required by the SDK package
- ACP JSON-RPC 2.0 for editor and client integration
- Model Context Protocol (MCP) for external tools, resources, prompts,
  completion, subscriptions, and elicitation
- Vercel AI Gateway and Vercel authentication for model-backed operation
- tmux for real-TTY end-to-end coverage, especially resize and rendering tests
- GitHub Actions for native multi-platform CI, release qualification,
  benchmarks, binary-size reporting, and releases

Primary commands:

```bash
zig fmt src/
zig build
zig build test
zig build run
```

The freshly built development binary is always `./zig-out/bin/fx`. Never use a
bare `fx` command as verification evidence.

## Project Conventions

### Code Style

- Run `zig fmt src/`; Zig formatting is canonical.
- Prefer `snake_case` for Zig identifiers and `PascalCase` for types.
- Keep the `pub` surface minimal and expose declarations only across real module
  boundaries.
- Pass allocators explicitly, free owned allocations with `defer` or
  `errdefer`, and prefer request-scoped arenas where bulk cleanup is suitable.
- Document ownership when returning allocated memory.
- Return errors for runtime failures. Reserve `@panic` for programmer errors
  and prefer bounded error sets when practical.
- Use Zig 0.16 `std.Io` APIs and the shared `src/core/shared/io.zig` helpers for
  files, environment variables, time, real paths, process spawning, and sleep.
- Use `std.json.Stringify.value` with an allocating writer for JSON. Use
  `writeJsonStr` from `src/acp/jsonrpc.zig` when manually escaping JSON strings.
- Use kebab-case for CLI flags. Keep help text and argument contracts
  centralized rather than scattering strings across implementations.
- Do not use emoji in code, output, or documentation. Do not use double hyphens
  as prose punctuation in documentation.

### Architecture Patterns

fx separates typed product contracts and state from transports and
presentation:

- `src/main.zig` is the composition root. It wires dependencies but must not
  contain leaf feature logic.
- `src/core/` owns contracts, runtimes, application flow, configuration,
  sessions, permissions, MCP, skills, subagents, tooling dispatch, terminal
  engines, and other product state.
- `src/tools/` owns built-in tool implementations. Generic contracts and
  dispatch belong in `src/core/tooling/`; default tool specifications remain
  centralized in `src/core/tooling/tool_specs.zig` or `src/builtins/tools.zig`.
- `src/ui/` owns terminal rendering, input, the event loop, and transcript
  presentation. It must not own hidden product state.
- `src/gateway/` owns provider transport and streaming. It must not absorb
  product-state logic.
- `src/acp/` owns the ACP JSON-RPC 2.0 server.
- `src/builtins/` assembles built-in commands, tools, skills, modes, hooks, and
  gateway adapters.
- `sdk/` owns JavaScript host adapters for the WebAssembly and Node-API
  embedding surfaces.

Define a typed contract before adding behavior when ownership is unclear. A new
feature must identify its owning module, persistence needs, text and JSON output
contracts, documentation and tests, and deterministic E2E corpus
classification.

New commands add their specification in
`src/core/slash_commands/command_specs.zig`, dispatch through
`src/core/cli/cli_surface.zig`, and render text and JSON from the same typed
snapshot via `src/core/output/output_contracts.zig` when structured output is
needed.

Security is permission-first. Sensitive tools must integrate with
`src/core/permissions/permissions.zig`; permission checks must not be bypassed
or duplicated in an alternate execution path. Configuration and saved-session
rules remain authoritative, and action-bound approval requests require exact
live revalidation.

The interactive UI renders inline by default. Only the five owner classes in
`AlternateScreenOwner` may exclusively use the alternate screen: permission
review, full transcript, catalog menus, subagent manager, and hosted child
terminal takeover. Each owner must restore the main grid and all terminal modes
when it exits or hands off ownership.

### Testing Strategy

- Put Zig unit tests in the source file they cover and use `std.testing`.
- During development, run the narrowest focused test for the changed path,
  format the source, build the binary, and exercise the behavior end to end
  with `./zig-out/bin/fx`.
- `zig build test` runs the Zig suite. A passing suite is necessary but does not
  replace launching and driving the built binary.
- `tests/e2e/` contains deterministic Bun tests for CLI, ACP, permissions, MCP,
  and TTY behavior. TUI tests use tmux.
- `tests/evals/` contains credentialed live-model evals and requires
  `AI_GATEWAY_API_KEY`; these nondeterministic evals are separate from the ship
  gate.
- Prefer the shared terminal engine for fast rendering tests, tmux for real TTY
  and signal behavior, and `FX_RECORD` tapes plus `fx replay` for deterministic
  reproduction of reported render bugs.
- Every root `tests/e2e/*.test.ts` file has exactly one role in
  `scripts/pgso/corpus.json`: training, verification-only, or intentional
  exclusion with a concrete reason.
- Before declaring a PR ready, Full CI and the final ship gate must pass for the
  exact commit on Linux x86_64, Linux aarch64, macOS x86_64, and macOS aarch64.
  Each platform includes a ReleaseSafe native check and four isolated E2E
  shards.

### Git Workflow

- Make small, reviewable changes on a non-`main` feature branch.
- After focused local checks, create a clean checkpoint commit, push the branch,
  and open a draft pull request so Full CI can run.
- Do not mark a draft ready or request review until Full CI and the final ship
  gate pass for the exact current commit.
- Use a clean imperative PR title without a bracketed type prefix.
- Apply exactly one primary-intent label: `type: bug`, `type: feature`,
  `type: improvement`, `type: docs`, `type: maintenance`, `type: release`, or
  `type: security`.
- Do not commit generated runtime or build state from `.fx/`, `.zig-cache/`, or
  `zig-out/`.
- Release automation owns version tags. Never create a version tag manually.
- Update all relevant user-facing documentation when behavior changes,
  including command help, `README.md`, and `CONTRIBUTING.md` when applicable.

## Domain Context

- fx is a CLI-first coding agent, not a terminal IDE. Its default interaction
  should remain compact, inline, and compatible with terminal scrollback.
- The native binary supports an interactive TUI, one-shot `fx ask`, session
  management, ACP, built-in tools, skills, MCP servers, subagents, permissions,
  and deterministic terminal replay.
- Profile configuration and runtime state live under `~/.fx/`. A project
  `.fx.json` contains only repository-safe defaults.
- Configuration precedence is environment variables, profile workspace
  overrides, profile global settings, project `.fx.json`, then built-in
  defaults.
- Sessions live under `~/.fx/sessions/<session-id>/`, are portable across
  workspaces, and record their current workspace root. Subagent children are
  ordinary sessions with their own history and control state.
- `permission_mode` may be `ask`, `auto`, or `yolo`. Configured denies are
  evaluated before saved-session rules. Session approvals are non-persistent,
  and remembered rules are exact and session-scoped.
- Native MCP configuration is trusted profile state in `~/.fx/mcp.json`.
  Runnable MCP commands, URLs, environment values, and secrets do not belong in
  project `.fx.json`.
- Reffy owns exploratory artifacts and workflow metadata under `.reffy/`.
  ReffySpec owns canonical capability specs and proposed changes under
  `.reffy/reffyspec/`; current truth belongs in `specs/`, while deltas belong in
  `changes/`.

## Important Constraints

- Maintain compatibility with Zig 0.16.0 or newer and do not add dependencies
  outside the Zig standard library without prior discussion.
- Preserve the permission boundary for every sensitive operation and keep
  runnable configuration and credentials out of repository-owned config.
- Keep product state in core runtimes rather than only in the live shell or UI.
- Avoid parallel implementations of the same feature without a clear ownership
  reason.
- Preserve a 7.800 MiB production binary ceiling for the qualified macOS arm64
  release artifact. Investigate notable binary growth.
- Preserve the Linux CI startup budget of 2 ms per benchmarked command. Local
  non-Linux timings are informational.
- Treat deterministic E2E classification and PGSO corpus ownership as part of
  feature design, not test cleanup.
- Do not document planned behavior as if it already exists.
- Do not manually change the placeholder version in `build.zig.zon`; release
  versioning is sourced from `src/main.zig` and the release workflows.
- Canonical repository links must use `https://github.com/vercel-labs/fx`.

## External Dependencies

- Vercel AI Gateway for model catalogs and model inference
- Vercel OAuth, `AI_GATEWAY_API_KEY`, or `VERCEL_OIDC_TOKEN` for authenticated
  model-backed flows
- macOS Keychain for native secret and MCP OAuth credential storage on macOS;
  other platforms use restricted profile files
- User-configured MCP servers over stdio or Streamable HTTP, with versioned
  legacy protocol adapters where required
- ACP clients such as editors communicating with the native JSON-RPC server
- Bun and tmux for deterministic end-to-end testing
- Node.js 20 or newer for the experimental JavaScript SDK package and tests
- GitHub Actions for the required CI, performance, size, release, and publishing
  workflows
- No third-party Zig packages are declared in `build.zig.zon`
