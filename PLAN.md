# zopencode — OpenCode v2 plugin that delegates coding tasks to the ZCode CLI

**Status:** plan (v1, 2026-11-02)
**One-liner:** an OpenCode v2 plugin exposing a `zcode` tool: OpenCode's agent hands a task to the ZCode CLI as a sub-agent; the CLI runs on the user's existing ZCode login with **GLM-5.3-Flash** as the default and only model.

---

## Hard constraints (non-negotiable, from the project owner)

1. This repository is the plugin project's home. **All development files live here.**
2. Never written or modified, ever:
   - installed OpenCode and its config (`~/.config/opencode`, any user `opencode.json`);
   - `~/.zcode` (read-only inspections only);
   - the installed ZCode desktop app (its bundled CLI is *executed in place*, never touched);
   - the ZCode source checkout (`C:\Users\USER\ZCode`) — the owner uses it live.
3. OpenCode's source is **not needed**: we develop against the published `@opencode-ai/plugin` npm package; its TypeScript type definitions are the API contract.
4. Testing uses a throwaway project *inside this repo* (`test-project/`); OpenCode is launched with its working directory there, so plugin loading never involves the owner's real harness config.
5. The only expected side effect when the plugin runs is the zcode CLI writing its own session data under `~/.zcode/cli` — normal CLI operation, and inherent to "auth must work".

## Goal & scope

- **In scope:** plugin package (`src/` → `dist/index.js`), `zcode` tool (delegate task, auto-resume sessions), test project, README, npm-publishable packaging.
- **Out of scope (owner runs these later, on their terms):** `opencode plugin add …` into real projects, npm publishing, any change to ZCode itself.

## Verified facts this design rests on (from ZCode source research)

| Fact | Evidence |
| --- | --- |
| Auth needs zero setup: CLI decrypts `~/.zcode/v2/credentials.json` (already present) with a key derived from `platform:homedir:username` when `ZCODE_CREDENTIAL_SECRET` is unset — any same-user subprocess is authenticated | `apps/zcode-cli/packages/adapters/src/auth/credential-cipher.ts:87-100` |
| Headless mode has no `--model` flag; the default model comes from `defaultModelSelection: {providerId, modelId}` in `~/.zcode/v2/provider_config.json` | `apps/zcode-cli/docs/usage/03-config.md` |
| The owner's plan exposes provider `account:zai-start-plan` with models `GLM-5.3-Flash` \| `GLM-5.2` \| `GLM-5-Turbo`; current desktop session already runs `GLM-5.3-Flash` | `config/provider/zcode-builtin.json:1095-1099` |
| Headless contract: `zcode -p "<prompt>" --output-format json --cwd <dir> [--resume <id>] [--mode <mode>]` → one JSON object with `sessionId`, `response`, usage, projection; exit 0/1 | `apps/zcode-cli/packages/cli/src/prompt-command.ts:386-446`, `docs/usage/13-automation.md` |
| The JSON contract is documented in-repo but **not frozen between builds** → the plugin must tolerate unknown fields | `docs/usage/02-cli-reference.md:39-53` |
| Bundled CLI exists on this machine at `%LOCALAPPDATA%\Programs\ZCode\resources\glm\zcode.cjs`; the desktop itself runs it via Electron-as-Node (`ELECTRON_RUN_AS_NODE=1`) | confirmed on disk; `packages/services/src/zcode-agent/zcodeAgentProcessManager.ts:427-451` |
| ZCode has **no MCP server mode** — wrapping the headless CLI (this design) is the only practical integration; MCP wrapping would need an extra shim for no OpenCode-specific benefit | `apps/zcode-cli/packages/adapters/src/mcp` (client-only) |
| OpenCode v2 permission rules are ordered `{action, resource, effect}` (last match wins, default allow); string-named actions mean the plugin's tool can be gated by name, e.g. `{"action": "zcode", "resource": "*", "effect": "ask"}` | opencode.ai/v2/docs/permissions |
| Inside a delegated call the CLI is a separate process governed by **ZCode's** permission service (headless default `yolo`); OpenCode's rules stay the outer gate via the `ask` rule above | ZCode `PermissionService` (`packages/core/src/permission/service.ts:87`) |

## Architecture

```
OpenCode agent ──tool call──▶ zcode tool (this plugin, in-process with OpenCode/Bun)
                                  │ Bun.spawn (per delegation)
                                  ▼
                        zcode CLI  -p "<prompt>" --output-format json
                        --cwd <worktree> --mode <mode> [--resume <id>]
                                  │  (auth: ~/.zcode credentials, inherited)
                                  ▼
                        GLM-5.3-Flash via account:zai-start-plan
                                  │  JSON {sessionId, response}
                                  ▼
                        tool returns response text + "zcode session: <id>"
```

- **Session continuity:** module-level `Map<opencodeSessionID, zcodeSessionId>`; plain follow-up calls auto-resume the same ZCode session. Args `resume` (explicit id) and `fresh` (skip map) override.
- **CLI resolution chain:** `ZCODE_CLI_CMD` env → `zcode` on PATH → bundled `zcode.cjs` via the installed Electron binary with `ELECTRON_RUN_AS_NODE=1`.
- **Auth failure** surfaces as an actionable hint: "log in via the ZCode desktop app or run `zcode login`" — no silent retries.
- **Permission layering (design intent):** OpenCode's `ask` rule on the `zcode` action is the gate the owner clicks; once approved, the sub-agent runs autonomously under ZCode's own mode (default `yolo`). Stricter headless modes auto-deny approval-needing actions, which stalls tasks — `mode` stays an optional arg, `yolo` the default.

## Implementation steps

1. **Scaffold** — `package.json` (ESM, `@opencode-ai/plugin` as peer dep; dev: TypeScript, build script, `@types/bun`), `tsconfig.json`, `src/index.ts`.
2. **Implement the plugin** — register the `zcode` tool per the architecture above; exact v2 hook shape confirmed against the SDK's installed `.d.ts` at this step (no OpenCode clone).
3. **Test harness in-repo** — `test-project/opencode.json` with `"plugins": [".."]` (relative local path) and the `{"action":"zcode","resource":"*","effect":"ask"}` rule; OpenCode launched with cwd = `test-project/`.
4. **Typecheck + build** until clean.
5. **Read-only model check** — confirm `defaultModelSelection` in `~/.zcode/v2/provider_config.json` already pins `account:zai-start-plan` / `GLM-5.3-Flash` (print only that key, never the file). If missing: hand the owner the exact JSON snippet to apply themselves.
6. **Smoke test** — delegate a trivial task through the plugin from `test-project/`, then a follow-up to prove `--resume`. Side effect: zcode's own session data under `~/.zcode/cli`.
7. **README + packaging** — install instructions (`opencode plugin add <pkg-or-path>`), version-notes tied to tested ZCode builds.

## Verification checklist

- [ ] `tsc`/build clean against `@opencode-ai/plugin` types.
- [ ] OpenCode (cwd = `test-project/`) discovers the plugin and the `zcode` tool.
- [ ] The `ask` permission rule fires before the first delegation.
- [ ] Delegation returns the CLI's `response`; output shows `zcode session: <id>`.
- [ ] Follow-up call resumes the same session (`--resume` path exercised).
- [ ] `defaultModelSelection` confirmed (read-only) or snippet handed to the owner.

## Risks & notes

- **Contract drift:** the CLI's JSON output may change between builds; the plugin parses tolerantly and README documents the tested ZCode version.
- **No streaming / no abort:** the tool returns the final response when the CLI finishes; OpenCode tool context exposes no abort signal.
- **Bundled-CLI fallback** depends on the desktop install layout; `ZCODE_CLI_CMD` and PATH cover relocation.
- **Model pinning:** single model by design; changing it means editing `defaultModelSelection` (owner-owned action), not the plugin.
