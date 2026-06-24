---
name: harness-agent
description: "Use when solving an exercise through the local Harness middleware. The harness owns connector credentials and submissions; agents use harness context, exercise, run start/current/ping, submit, refresh, solution, best, and history without reading run_state.json or connector secrets."
---

# Harness Agent

Use this skill when solving an exercise through the local Harness middleware.
The harness owns connector credentials and submissions; the agent only talks to
`harnessd`.

This skill is not the Harness server. It assumes the user has already installed
and started the server from `https://github.com/josusanmartin/harness` and has
provided a scoped `HARNESS_RUN_TOKEN`. If the `harness` CLI is missing or
`HARNESS_URL` / `HARNESS_RUN_TOKEN` are not available, stop and report that the
Harness server setup or exercise API key is missing. Do not clone or run this
skill repository as the server.

## Harness Context

Do not ask the user for connector API keys, cookies, credential paths, or
`run_state.json`. The normal workflow starts in the Harness web UI: the user
creates an exercise API key for exactly one user, credential profile, connector,
and exercise, then provides the agent with:

```bash
export HARNESS_URL=http://127.0.0.1:8718
export HARNESS_RUN_TOKEN=hrun_...
```

The `harness` CLI also supports auto-discovery from a harness workspace for
legacy runs, but a web-issued `HARNESS_RUN_TOKEN` is the preferred path:

```bash
harness context
```

The token is never printed. For the preferred web-issued workflow,
`harness context` must show a server scope with `kind=run_token`, `user_name`,
`connector`, `exercise`, and `credential_profile`. If a user expected a web-issued
exercise API key but that scoped context is missing, stop and report that the
harness exercise API key is required. Legacy workspace contexts may show
`kind=arm_token`; use them only for existing harness workspaces that already
have an active arm context.

Connector credentials are stored on the middleware side.
The Harness UI can store many named credential profiles per connector, but the
exercise API key binds exactly one of them. Do not try to list, infer, or switch
credential profiles from the agent side. If the selected profile, connector, or
exercise is wrong, ask the user to create a new exercise API key in the web UI.
Multiple independent runs may use the same credential profile. The run token is
still an isolated view: `best`, `history`, `refresh`, and `solution`
only apply to the token's own run after `harness run start`. Do not ask the
harness for previous code, sibling-run submissions, or another run's connector
solution id; the middleware should reject those requests.

## Token Accounting

Every submission must include a real run-relative `--total-tokens` value. Do
not submit `0`, an estimate, or a number copied from local history. If no exact
counter is available within one quick check, stop before submitting and ask the
user to restart the run through a supervised Codex/provider runner that exposes
usage.

Use the bundled helper to make token accounting deterministic:

```bash
HARNESS_TOKEN_HELPER="${CODEX_HOME:-$HOME/.codex}/skills/harness-agent/scripts/token_usage.py"
```

Immediately after `harness run start`, capture a baseline from the first exact
source available:

- **Codex with `get_goal` tool**: call `get_goal` once. If it returns a current
  cumulative token count for the active goal, store it:

  ```bash
  python3 "$HARNESS_TOKEN_HELPER" start --total-tokens <get_goal_total_tokens> --source codex_goal
  ```

- **Codex noninteractive/supervised runner**: if the launcher started Codex
  with JSONL capture and exported the live event log path, use that file as the
  usage source:

  ```bash
  python3 "$HARNESS_TOKEN_HELPER" start --codex-jsonl "$CODEX_EXEC_JSONL" --source codex_exec_jsonl --confidence parsed
  ```

  Codex JSONL emits `turn.completed.usage`; the helper sums
  `input_tokens + output_tokens` for completed turns. Do not launch a nested
  `codex exec` from inside the solving agent.

- **Codex TUI without `get_goal`**: if `/status` visibly shows an exact current
  session token count, use that number. Do not use `/usage`, because account
  usage is not scoped to this harness run.
- **Claude Code**: use the exact visible `/cost` token total if it is shown. If
  it only shows money or cannot be read exactly, stop before submitting.
- **Gemini CLI**: use the exact visible `/stats` token total if it is shown. If
  it cannot be read exactly, stop before submitting.
- **Provider/API runner**: use provider `usage` fields summed cumulatively for
  this run. Source: `provider_usage` or `runner_measured`.

Before every submit, take the same exact snapshot source again and let the
helper produce the harness flags:

```bash
TOKEN_FLAGS="$(python3 "$HARNESS_TOKEN_HELPER" flags --total-tokens <current_total_tokens> --source codex_goal)"
```

or, for a Codex JSONL runner:

```bash
TOKEN_FLAGS="$(python3 "$HARNESS_TOKEN_HELPER" flags --codex-jsonl "$CODEX_EXEC_JSONL" --source codex_exec_jsonl --confidence parsed)"
```

The helper subtracts the run-start baseline and emits run-relative
`--total-tokens`, `--usage-source`, `--usage-confidence`, and
`--tokens-total-source`. For Codex sources it emits trusted
`--usage-source codex_usage` and keeps the specific origin, such as
`codex_goal` or `codex_exec_jsonl`, in `--tokens-total-source`. Do not
hand-write those flags unless the helper is unavailable.

Never search `~/.codex`, `~/.claude`, browser profiles, shell snapshots, or
old transcripts to infer tokens. Those sources are not run-scoped and have
already produced wrong dashboard data.

## Workflow

Follow these steps in order. Do not skip `harness run ping`; the harness uses
server timestamps from run start, ping, and submissions to derive elapsed time
and remove restart gaps.

1. Confirm the scoped harness context:

```bash
harness context
```

2. Read the configured exercise:

```bash
harness exercise
```

If the user explicitly asks for a different problem than the run default, pass
that problem with `--exercise`, but normal runs should not need this.

3. Choose and start a run before submitting:

```bash
harness run start --id run001
```

Use a stable, human-readable run id such as `run001`,
`harness-run-20260622-001`, or a short strategy name plus timestamp. Optional
metadata can be supplied when it is useful:

```bash
harness run start --id run001 \
  --label "skill-research run001" \
  --strategy "progress logging skill with perf access" \
  --hypothesis "perf-guided changes should reduce score faster"
```

If the harness says a previous run already exists for this user/profile/exercise,
continue it with the exact `harness run start --id <previous-run>` command from
the error. Only use `--confirm-new-run` when the user explicitly wants a new
independent run rather than a continuation.

After starting, confirm the active run:

```bash
harness run current
```

4. Immediately ping the run before doing optimization work:

```bash
harness run ping --event start --note "starting work"
```

If this is a resumed session, use:

```bash
harness run ping --event resume --note "resuming work"
```

Ping once at the start of every new or resumed agent session. The ping is not a
claim about effort; it gives the middleware a server timestamp so reports can
drop dead time between sessions.

5. Initialize token accounting before doing optimization work. If no exact
counter is available, report that token accounting is blocked and do not submit
candidates for this run.

6. Work in the current agent workspace. Keep candidate files small and scoped to
the actual solution.

7. Capture a token snapshot before every submission using the token accounting
   helper above. Keep provenance honest. Do not delay a simple first candidate
   by reverse-engineering token usage; either use an exact supported counter or
   stop before submitting.

8. Submit through the harness only, and include `--total-tokens` in the same
   call. The CLI automatically includes the token-bound run id from
   `harness run current`; the middleware rejects submissions without both an
   active run and a token snapshot:

```bash
harness submit path/to/solution \
  --label short-name \
  --notes "what changed" \
  --idempotency-key short-name-v1 \
  $TOKEN_FLAGS
```

Do not pass `--tokens-delta` manually; the harness computes deltas from the
run-relative cumulative total. Do not pass any timing fields; the harness
derives elapsed time from server timestamps and `harness run ping`. Do not
fabricate usage fields. If no token counter is available after the quick
supported checks, do not submit.

9. Inspect feedback and refresh queued submissions through the harness:

```bash
harness best
harness history
harness refresh
```

Use `harness refresh <candidate_id>` when you need to refresh a specific
submitted candidate. If the connector exposes source retrieval through the harness,
use `harness solution <solution_id>` rather than calling the connector directly.

For PR-backed connectors, this is still the full workflow. The middleware decides
whether the candidate becomes a local harness run, an API submission, or a
GitHub pull request.

## Rules

- Do not call the external connector CLI or API directly.
- Do not read or write credential files or `run_state.json`.
- Treat `harnessd` responses as the source of truth for scores and statuses.
- Do not choose users, credential profiles, or exercises yourself; those are
  bound into the web-issued exercise API key.
- Do not compare against or inspect other credential profiles or runs unless
  the harness response for the current token exposes that data.
- Do choose exactly one run id before the first submission, then keep using it
  for that run.
- If the strategy, model, tool access, prompt, or experimental condition
  materially changes, ask whether this is a continuation or start a new run only
  with explicit confirmation from the harness/user.
- The token-bound run metadata must be honest and specific: strategy,
  hypothesis, and notes are used for deterministic strategy comparison reports.
- Keep `--notes` factual: hypothesis, result, important failure, or connector id.
- Use `--label` to make deterministic progress logs readable.
- Use `--idempotency-key` for retries of the same candidate.
- If a submit error is unclear, read `docs/middleware-protocol.md` before retrying.
