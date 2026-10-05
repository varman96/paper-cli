# Paper

Paper captures command failures in a structured Incident Report for coding-agent investigation.

[GitHub](https://github.com/varman96/paper-cli) · [npm](https://www.npmjs.com/package/@varman96/paper)

Primary workflow: Run your command normally. If it fails, run Paper.

## Install

```bash
npm install -g @varman96/paper
```

## Use

Run your command normally:

```bash
$ npm test

FAIL src/router.test.js
TypeError: primaryAction is not a function
```

If it fails, run Paper:

```bash
$ paper run npm test
```

Paper creates `.paper/incident_report.md`:

```markdown
# Incident Report

## Evidence

### Commands

npm test

### Exit code

1

### Working directory

C:\work\my-app

### Timestamp

2026-09-30T14:32:08.000Z

### stderr / stack trace

Error: Expected 2 tests to pass, received 1
    at runTests (test/runner.js:42:11)
```

Give that report to your coding agent:

```text
Diagnose and fix the failure in .paper/incident_report.md
```

If the command works, you're done.

## Commands

- `paper run <command> [args...]` — run a target command through Paper and capture a report if it fails.

- `paper shelf` — move the active Incident Report out of the workspace and preserve it in Paper's local shelf.

- `paper --help` — print Paper's usage, examples, report location, shelf command, and metadata flags.

- `paper --version` — print the installed package version.

## Errors

> **Warning:** Do not use `paper run` to run Paper itself or commands that invoke Paper recursively. Paper is designed to capture failures from the command being investigated, not its own execution. Running Paper through Paper can produce recursive or misleading incident context.

Paper has no numeric or separately registered error-code system. Its own failures use uppercase cause labels in the CLI output:

| Label | When it occurs | Meaning and action |
| --- | --- | --- |
| `CLI ENTRYPOINT MISMATCH` | Startup | Paper cannot resolve the executable path to the active CLI entrypoint. Reinstall or relink Paper, then run the command again. |
| `NO COMMAND` | Argument parsing, with no arguments or with `paper run` alone | No target command was supplied. Use `paper run <command>`. |
| `INVALID COMMAND` | Argument parsing, when the first argument is not `run` | Paper requires the canonical `paper run <command>` form. |
| `AGENT RULES WRITE FAILED` | Before `paper run` starts the target | Paper could not create or update the repository's agent-instruction file. Restore write access and run Paper again. |
| `BINARY MISSING` | Starting the `paper run` target | The target was not found in the active `PATH`. Install it or use a command that exists in the shell. |
| `PERMISSION DENIED` | Starting the `paper run` target | Paper could not execute the target from the shell. Restore execute permission or use an executable command. |
| `TARGET START FAILED` | Starting the `paper run` target | The target could not be started for another spawn error. Run it directly in the shell to verify it starts, then run Paper again. |
| `REPORT WRITE FAILED` | Writing `.paper/incident_report.md` after a target failure | Paper could not persist the Incident Report. Restore write access to the report location and run Paper again. |
| `NO ACTIVE INCIDENT` | `paper shelf` | `.paper/incident_report.md` was not found, so no files were changed. Run `paper run <command>` first. |
| `INCIDENT SHELF FAILED` | `paper shelf` while creating the shelf directory or moving the report | Paper could not move the report. The active report is preserved; restore access to the shelf location and run `paper shelf` again. |
| `PAPER OPERATION FAILED` | An unexpected local operation throws an unclassified error | Paper stopped before the requested operation completed. Resolve the reported local error and run Paper again. |

`paper run <command>` also prints `COMMAND FAILED` when the user's command exits nonzero or is terminated by a signal. This is not a Paper error: Paper captured the user's command failure, wrote the Incident Report, and exits successfully if that write succeeds. The command's own stdout/stderr, including errors from its shell or runtime, remain command-generated evidence rather than Paper error labels.

### Found a bug?

If you encounter a bug or an error that isn't documented here, please let us know.

Email **varmanvishnu96@gmail.com**, or join the [Paper Discord](https://discord.gg/9KqwMNTft) and post it in the **#bugs** channel.

## How it works

Paper is used after a command has failed. `paper run <command>` executes the target command through the local CLI and forwards its stdout and stderr. Before starting it, Paper adds or updates its managed workflow block in the detected agent-instruction file, falling back to `AGENTS.md`.

If the target succeeds, `.paper/incident_report.md` is unchanged. If it exits nonzero or is terminated by a signal, Paper traps that failure, captures the evidence locally, writes a structured Incident Report to `.paper/incident_report.md`, and prints `COMMAND FAILED`. The report contains the command, exit code, working directory, timestamp, and stderr or stack trace. Once the report is persisted, Paper exits successfully; Paper-owned failures, such as inability to update instructions or write the report, exit nonzero.

The Paper instruction tells coding agents where to find the Incident Report, so an agent can investigate the failure without the user manually copying terminal output. Paper does not diagnose or fix the failure.

Paper creates persistent local context and makes it discoverable to the agent. Its operation is local-only and air-gapped: there are no model API calls, telemetry, or command-output uploads.

### Shelving an incident

Once the incident is resolved, remove it from the active workspace with:

```bash
paper shelf
```

Paper moves `.paper/incident_report.md` out of the repository with a rename and prints the stored path. On Windows, shelved reports are stored under `%LOCALAPPDATA%\Paper\shelf\<repository-hash>\`; on other platforms, under `$XDG_DATA_HOME/Paper/shelf/<repository-hash>/` or `~/.local/share/Paper/shelf/<repository-hash>/`. The report is preserved as a timestamped `.md` file under a repository-specific 16-character SHA-256 directory, but is no longer the active report. The active report is removed from `.paper`, its contents are preserved, and the instruction file is not changed. With no active report, or if the move fails, Paper exits nonzero; a failed move preserves the active report.

### Recovering a shelved incident

Paper currently has no list or recovery command. `paper shelf` prints the exact path of the newly shelved report, which is also how to locate it later. To recover one, manually copy that `.md` file back to the repository as `.paper/incident_report.md`; Paper does not perform this restore or automatically replace an existing active report. Shelving and manual recovery do not update `AGENTS.md`; the existing Paper instruction remains unchanged and will point agents to `.paper/incident_report.md` when that file is present again.

## License

PolyForm Shield License 1.0.0
