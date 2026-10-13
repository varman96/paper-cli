# Security

Paper is a local-first CLI. This document describes its threat model, data handling, and how to verify these claims independently.

## Threat model

Paper operates at the failure-capture point: it spawns a command, observes its stdout, stderr, exit code, and surrounding context (command, working directory, timestamp), and writes a structured Incident Report for a coding agent to read. Standard error and stack traces are where secrets leak, so this position is treated as privileged.

The governing property: **Paper contains no network code. There is no exfiltration path in the shipped binary.**

The remaining transfer risk is downstream. A coding agent that reads the local report may submit its contents to an external model API. Paper cannot control the agent's behavior. It can ensure the agent ingests scrubbed output where requested, and prove afterward exactly what was captured. The controls below serve that purpose.

## What Paper does not do

- No network calls of any kind: no telemetry, no analytics, no model API calls, no crash reports, no output uploads, no update checks.
- No background processes or daemons.
- No credential handling beyond capture: Paper never reads, stores, or forwards API keys, tokens, or secrets except as they appear in the failing command's own stderr. By default that output is written to the local report verbatim. `paper run --redact` applies an opt-in, offline, deterministic regex pass that replaces known secret patterns (AWS keys, GitHub/GitLab tokens, bearer and api_key assignments, private key blocks, credentialed connection strings, webhook URLs) with `[REDACTED:<label>]` before anything is written. The pattern list is fixed and not exhaustive. Reports are written with default file permissions; on shared machines, prefer `--redact`.

## Verifiable claims

Each claim below can be checked in about ten minutes against the published package.

1. **Bundle audit.** The npm package ships a single bundled file, `dist/paper.js`, containing no network primitives. Verified against v1.0.12 (2026-10-07): grep of the shipped bundle for `fetch`, `XMLHttpRequest`, `WebSocket`, `http(s).request`, `net.connect`, `dns.lookup`, `tls.connect`, `socket.io`, and any `node:http/https/net/dns/tls/dgram/http2` import returned zero matches. Reproduce: `npm pack @varman96/paper && tar xzf *.tgz && grep -oiE '\b(fetch|XMLHttpRequest|WebSocket)\b' package/dist/paper.js`.
2. **Dependency surface.** `package.json` declares zero runtime dependencies (dev-only tooling: esbuild, wrangler, biome). There is no transitive dependency tree.
3. **Small binary.** The shipped bundle is approximately 27 KB and fully readable.
4. **Receipt verification.** Every capture is sealed with an HMAC-SHA256 receipt under a machine-local key. `paper verify` recomputes the per-field hashes and the HMAC from `.paper/latest.json`, printing `RECEIPT OK` or identifying the altered field (e.g., `stderr was modified after capture`). The per-field hashes cover `signal`, `signal_code`, and `probable_cause` alongside the other captured fields. Reproduce: modify the stored `stderr` and run `paper verify`.

## Data flow

All processing occurs on the local machine:

- `paper run <command>` executes the command locally and forwards its output. On failure, Paper writes `.paper/incident_report.md` and `.paper/latest.json` (structured fields plus the full markdown report) in the current repository. On success, nothing is written.
- Each capture appends one JSON line to `.paper/history.jsonl`: timestamp, command, exit code, receipt hash, and flags. Each entry includes an HMAC of the previous entry's hash, forming a hash chain. `paper verify --chain` checks the sequence and reports the index of the first break. Appending is best-effort and never fails a run. Shelving a report does not remove its history entry.
- The receipt key is stored at `<data dir>/receipt.key`: 32 random bytes, mode `0600`, created on first capture (`%LOCALAPPDATA%\receipt.key` on Windows; `~/.local/share/Paper/receipt.key` or `$XDG_DATA_HOME` elsewhere). The key is machine-local: receipts do not verify on other machines, and key loss orphans old receipts (`paper verify` reports `KEY MISSING`, distinct from a tampering finding). On Windows, Node cannot set POSIX-style permissions, so `0600` is best-effort there.
- `paper run --fence` (opt-in) creates a git worktree at `.paper/scratch` on failure, directing diagnosis off the main tree; `paper shelf` removes it. Fencing does not block capture.
- `paper shelf` moves the active report out of the repository to the local shelf (`%LOCALAPPDATA%\Paper\shelf\` on Windows; `$XDG_DATA_HOME/Paper/shelf/` or `~/.local/share/Paper/shelf/` elsewhere). Reports do not leave the disk they were written to.
- Paper adds or updates a managed workflow block in the repository's agent-instruction file (e.g., `AGENTS.md`) so coding agents can locate the report. This is local file I/O only.

No step in this flow opens a socket.

## Filesystem behavior

- Paper refuses to write through symbolic links. If `.paper`, the report path, or an instruction file resolves to a symlink, Paper aborts instead of following it.
- `.paper/latest.json` is written atomically (temporary file plus rename). A crash mid-write cannot produce a partial artifact that `paper verify` would accept.
- The managed instruction block is confined to its fenced section (`<!-- paper:start -->` … `<!-- paper:end -->`) in agent-instruction files. Content outside the fence is left byte-identical.

## Evidence integrity

Paper does not diagnose or fix failures. The coding agent reads the Incident Report; it never writes it. The report is produced by the wrapper at the OS boundary, outside the agent's control.

This separation matters because agent-written logs are not reliable evidence: October 2026 security research (Adversa's agent vulnerability roundup) documents coding agents deleting or altering their own execution traces without monitor detection. Paper's log is produced out-of-band, so prompt manipulation or context pollution cannot alter it.

The integrity guarantee is cryptographic. A plain hash would be forgeable by anyone able to edit the file and recompute it, so receipts use HMAC-SHA256 under a machine-local key held outside the repository workspace. Forgery requires the key. Per-field hashes identify which field changed.

Where `--redact` is enabled, the HMAC covers the stored (redacted) output — the same content the agent ingested. An auditor can therefore establish that the agent saw only the scrubbed version.

Limitations: a process with shell access could deliberately locate the key outside the workspace; the protection assumes deliberate exfiltration effort rather than accidental edits. Key loss orphans old receipts. The history chain cannot detect removal of trailing entries. Fencing directs the agent to the worktree but cannot enforce it.

## Scope and responsibility

**1. Capture scope.** For each captured run, Paper records: the exact command and arguments; the exit code or, where the process was terminated by a signal, the signal name, the conventional 128+N signal code, and the fixed one-line probable-cause hint; the child process's stdout and stderr; the working directory; a timestamp; and git state, consisting of HEAD plus a fingerprint of uncommitted changes. Paper records nothing outside this set.

**2. Exclusions.** Paper does not inspect system state beyond what the operating system reports for the child process. It does not read kernel logs, process filesystems, core dumps, or environment variables beyond those required for command resolution. It initiates no network connections and performs no telemetry. Paper does not direct the agent's investigation and does not execute remediation actions.

**3. Allocation of responsibility.** The coding agent is solely responsible for its actions following ingestion of the Incident Report, including any files accessed, commands executed, or remediations applied. Paper's managed instruction block directs the agent to obtain explicit user approval before remediation; enforcement of that directive is the responsibility of the agent harness. Paper makes no representation as to the agent's compliance with that directive.

**4. Verifiability.** The HMAC receipt seals the captured record — or, where `--redact` is enabled, the redacted record the agent was permitted to see. An auditor may thereby establish precisely what the agent was given, and what it was not.

## Reporting a vulnerability

Report bugs or security issues to **varmanvishnu96@gmail.com** or the **#bugs** channel on the Paper Discord. Include the Paper version (`paper --version`) and a minimal reproduction.

---

*Verification log:*
- *v1.0.12 (2026-10-07): bundle network-primitive grep and npm metadata review — zero matches for `fetch`, `XMLHttpRequest`, `WebSocket`, `http(s).request`, `net.connect`, `dns.lookup`, `tls.connect`, `socket.io`; no `node:http/https/net/dns/tls/dgram/http2` imports.*
- *v1.0.13 (2026-10-11): bundle re-audited — zero matches for the same network primitives; all imports are `node:` builtins (`child_process`, `crypto`, `fs`, `os`, `path`, `url`); zero runtime dependencies in `package.json`. Bundle ~25 KB.*
- *v1.0.16 (2026-10-13): bundle re-audited — zero matches for the same network primitives; signal fields (`signal`, `signal_code`, `probable_cause`) added to receipt coverage. Bundle ~27 KB.*

*Re-audit on each release with the command above.*

