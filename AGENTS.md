# Instructions for an assistant helping a user run bnr-build-state

This file is for an AI coding assistant (Claude Code, Codex, or similar) helping a person install and
run `bnr`, the Branch & Row build state client for dbt. Read it fully before running anything.

## What the tool does

`bnr` wraps a dbt command. Before dbt runs, it asks the Branch & Row state service which models can
be skipped because their inputs have not changed. It then runs the same dbt command, with skipped
models excluded when the service's mode allows it, and reports what was built and reused. The run
exits with dbt's exit code.

`bnr` does not replace dbt, and it does not include dbt. The user's dbt and warehouse adapter must
already be installed, or installed alongside `bnr`.

## Check first

Before any build, confirm:

1. `bnr --help` prints `usage: bnr dbt ...`. If not, the tool is not installed. See Install.
2. `BNR_URL`, `BNR_TOKEN` and `BNR_ACCOUNT_ID` are set in the environment. Check with
   `test -n "$BNR_URL" && test -n "$BNR_TOKEN" && test -n "$BNR_ACCOUNT_ID" && echo ok`. Never print
   `BNR_TOKEN`.
3. The user has confirmed which dbt project, profile and target the build will write to.

If a required variable is missing, `bnr` does not fail. It prints
`BNR_URL, BNR_TOKEN, BNR_ACCOUNT_ID not set, running dbt unchanged` to stderr and runs plain dbt.
A run like that is a normal dbt run with no state checks, so say so to the user.

## Install

Requires uv (or pipx) and a Python 3.11+ interpreter. Use the version this repository's release page
lists. If the user already has `bnr`, check the version with `uv tool list` before reinstalling.

```
uv tool install "bnr-build-state[snowflake] @ https://github.com/usbrandon/bnr-build-state-releases/releases/download/bnr-build-state-v0.3.11/bnr_build_state-0.3.11-py3-none-any.whl" --with dbt-snowflake
```

- Use the extra for the warehouse: `snowflake`, `postgres` or `duckdb`.
- `--with dbt-snowflake` (or the matching adapter) installs dbt alongside `bnr`. Match the dbt version
  the user's project uses. Do not guess it; ask, or read `dbt --version` in the project.
- If `dbt` is already on PATH in the user's project, `--with` may be unnecessary. Check that
  `bnr dbt` finds it: `BNR_DBT_BIN` overrides the dbt binary.
- The uv tool bin directory may not be on PATH. uv prints the export line; use it.

## Configuration

Set these in the user's shell or secret store, not in a project file and not in git:

| Variable | Required | Meaning |
|---|---|---|
| `BNR_URL` | yes | Base URL of the Branch & Row API, from the user's account administrator. |
| `BNR_TOKEN` | yes | The user's run token. Treat as a secret. |
| `BNR_ACCOUNT_ID` | yes | Numeric account id. |
| `BNR_STATE_MODE_OVERRIDE` | no | `off` runs dbt unchanged (kill switch). `shadow` asks for a Shadow plan for this run. |
| `BNR_DBT_BIN` | no | dbt executable to run. Default `dbt`. |
| `BNR_RUN_ID`, `BNR_CAUSE` | no | Labels recorded with the run. |
| `BNR_QUERY_TIMEOUT_SECONDS` | no | Timeout for Snowflake metadata queries. Default 600. |

dbt reads its own environment variables and flags as usual. `--profile` (or `DBT_PROFILE`) picks the
profile whose connection supplies the state checks.

## Commands

```
bnr dbt build --selector <name>
bnr dbt run -s <model_selector> --target <target>
bnr state explain <plan_id> [-s <node>]
```

- `bnr dbt ...` accepts the same arguments as `dbt`. Pass them through unchanged.
- `bnr state explain <plan_id>` prints the stored decision for each node in a plan. A run prints the
  line `bnr state explain <plan_id>` when it has a plan; use that id.
- Exit code is dbt's exit code.

## Modes

The service decides a mode per key. The mode is set by the account
administrator, not by the user's local config:

- **off:** no plan. dbt runs unchanged.
- **shadow:** the plan is recorded and every planned skip is rebuilt to check the skip was correct.
  Nothing is skipped in the result.
- **enforce:** planned skips are excluded from the build, after verification.

`BNR_STATE_MODE_OVERRIDE=shadow` asks for Shadow for one run even when the key is in Enforce. A run
with an explicit `--selector` is always Shadow, because dbt does not combine `--exclude` with
`--selector`.

Do not change a key's mode, and do not set `BNR_STATE_MODE_OVERRIDE` to force Enforce. Those are
administrator decisions.

## Reading the output

- `[bnr] mode shadow: built N, reused N, errors N, undetermined N, disagreements N` is the summary.
  `built` and `reused` are counts from the client. `disagreements` is the service's answer: planned
  skips that the rebuild contradicted. A non-zero disagreements count means a skip was wrong. Report
  it to the user; do not hide it.
- `state service unavailable, building everything: <error>` means the plan could not be made, or the
  compile step before it failed. dbt then ran the whole command with no skips. This is safe, but the
  run was not optimized. The text after the colon names the cause (for example a connection or HTTP
  error). Do not guess the cause; show it to the user.
- `the selection is empty: nothing to plan, running dbt unchanged` means nothing matched the selector.
- `... not set, running dbt unchanged` means the configuration is missing (see above).
- `--empty or --sample builds partial data: not planned or reported` means the run was not checked.

## Troubleshooting

- **`bnr: command not found`:** the uv tool bin directory is not on PATH. Run the export line uv
  printed, or `uv tool update-shell`, then open a new shell.
- **`dbt: command not found`:** dbt is not installed in the environment. Install the adapter with
  `--with`, or set `BNR_DBT_BIN` to the dbt executable.
- **`state service unavailable` with an HTTP 401 or 403 in the error:** the token is wrong, expired, or
  belongs to a different account. Ask the user to get a new token from their administrator. Do not try
  other tokens.
- **Compile errors before any plan:** `bnr` runs `dbt compile` first. A failure shows as
  `state service unavailable, building everything: compile exited N`. Run `dbt compile` on its own with
  the same arguments to see the error.
- **Profile errors (`ProfileError`):** `bnr` reads `profiles.yml` the way dbt does. Environment
  variables in the profile must be set; `env_var()` without a default fails when unset.
- **dbt Fusion:** Fusion has no `--empty`, `--sample`, `--favor-state` or `--fail-fast`. Setting them
  (or their `DBT_*` variables) makes `bnr` more cautious: no plan for some partial builds, unknown
  parents, and no test step after a failed build. Expect more builds, not fewer.
- **Debugging a suspected wrong result:** run once with `BNR_STATE_MODE_OVERRIDE=off` to see plain
  dbt, and compare. Use `bnr state explain <plan_id>` to see why a node was skipped or built.

## Safety

- A `dbt build` or `dbt run` writes to the user's warehouse. Confirm the target and profile with the
  user before running it. Prefer `dbt compile` or `--help` to check arguments first.
- Never print, log, or commit `BNR_TOKEN`. Do not paste it into a file in the project. If the user
  pastes it into the chat, tell them to rotate it, and do not repeat it.
- Do not edit `profiles.yml` or the dbt project to get `bnr` to run. Report the problem instead.
- If a run reports disagreements or errors, stop and report them before running again.

## About this project

`bnr-build-state` is MIT licensed. The source is not published here; this repository holds release
artifacts only. dbt is a trademark of dbt Labs, LLC. This project is not affiliated with or endorsed
by dbt Labs.
