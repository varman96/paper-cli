# Security

Paper is a local-first CLI. This document describes its threat model, what it
does and does not do with your data, and how to verify those claims yourself.

## Threat model

Paper sits at the failure-capture point: it spawns your command, sees its
stdout/stderr, exit code, and environment-shaped context (command, working
directory, timestamp), and writes a structured Incident Report for a coding
agent to read. That position is privileged — stderr and stack traces are
exactly where secrets leak — so Paper is built on one principle:

**Paper has no network code. There is nothing to exfiltrate with.**

A tool that sees every failed command AND phones home is a supply-chain
nightmare. Paper cannot phone home because the capability does not exist in
the shipped binary.

The transfer risk lives downstream: a coding agent that reads the local
report may paste its contents into an external model API. Paper cannot
control what the agent does — but it can make sure what the agent ingests
is scrubbed, and prove afterward exactly what was captured. That is what
the controls below are for.

## What Paper does not do

- No network calls of any kind: no telemetry, no analytics, no model API
  calls, no crash reports, no command-output uploads, no update checks.

- No background processes or daemons.

- No credential handling: Paper never reads, stores, or forwards API keys,
  tokens, or secrets beyond what your own failing command prints to its own
  stderr. By default that output is written to the local report verbatim,
  exactly as captured. `paper run --redact` enables an opt-in local regex
  pass that replaces known secret patterns (AWS keys, GitHub/GitLab
  tokens, bearer/api_key assignments, private key blocks, credentialed
  connection strings, webhook URLs) with `[REDACTED:<label>]` before
  anything is written. Deterministic, offline, off unless you ask — and
  pattern-based, not exhaustive. Incident reports are written with default
  file permissions; on shared machines, prefer `--redact`.

## Verifiable claims

Security review for Paper is a ten-minute exercise:

1. **Bundle audit.** The npm package ships a single bundled file,
  `dist/paper.js`. It contains no network primitives. Verified against
  v1.0.12 (2026-10-07): `grep` of the shipped bundle for `fetch`,
  `XMLHttpRequest`, `WebSocket`, `http(s).request`, `net.connect`,
  `dns.lookup`, `tls.connect`, `socket.io`, and any `node:http/https/net/
  dns/tls/dgram/http2` import returned zero matches. Re-run it yourself:
  `npm pack @varman96/paper && tar xzf *.tgz && grep -oiE '\b(fetch|XMLHttpRequest|WebSocket)\b' package/dist/paper.js`.

2. **Dependency surface.** `package.json` lists zero runtime dependencies
  (dev-only tooling: esbuild, wrangler, biome). There is no transitive
  dependency tree to audit or poison.

3. **Small binary.** The shipped bundle is ~27 KB. You can read all of it.

4. **Receipt verification.** Every capture is sealed with an HMAC-SHA256
  receipt under a machine-local key. `paper verify` recomputes the
  per-field hashes and the HMAC from `.paper/latest.json` and either
  prints `RECEIPT OK` or names exactly what changed
  (`stderr was modified after capture`). Try it: edit the stored
  `stderr` yourself and run `paper verify`. The per-field hashes also cover
  `signal`, `signal_code`, and `probable_cause`, so signal tampering is
  named by `paper verify` like any other field.

## Data flow

Everything Paper does happens on your machine:

- `paper run <command>` executes your command locally and forwards its output.
  On failure, Paper writes `.paper/incident_report.md` and
  `.paper/latest.json` (structured fields plus the full markdown report) in
  the current repository. On success, nothing is written.

- Every capture appends one JSON line to `.paper/history.jsonl`
  (append-only): timestamp, command, exit code, receipt hash, and flags.
  Each entry carries an HMAC of the previous entry's hash — a hash chain.
  `paper verify --chain` checks the whole sequence and reports the first
  break index. Appending is best-effort and never fails a run. Shelving a
  report does not remove its history entry — the audit trail survives
  `paper shelf`.

- The receipt key lives at `<data dir>/receipt.key` (32 random bytes,
  mode `0600`, created on first capture): `%LOCALAPPDATA%\receipt.key`
  on Windows, `~/.local/share/Paper/receipt.key` (or `$XDG_DATA_HOME`)
  elsewhere. It is machine-local: receipts do not verify on other
  machines, and losing the key orphans old receipts (`paper verify`
  reports `KEY MISSING`, distinctly from tampering). On Windows, Node
  cannot set POSIX-style permissions, so the `0600` mode is best-effort
  there.

- `paper run --fence` (opt-in) creates a git worktree at
  `.paper/scratch` on failure so agents diagnose off the main tree;
  `paper shelf` removes it. Fencing never blocks capture.

- `paper shelf` moves the active report out of the repository to the local
  shelf (`%LOCALAPPDATA%\Paper\shelf\` on Windows,
  `$XDG_DATA_HOME/Paper/shelf/` or `~/.local/share/Paper/shelf/` elsewhere).
  Reports never leave the disk they were written to.

- Paper adds/updates a managed workflow block in the repository's
  agent-instruction file (e.g. `AGENTS.md`) so coding agents know where to
  find the report. This is local file I/O only.

No step in this flow opens a socket.

## Filesystem behavior

- Paper refuses to write through symbolic links. If `.paper`, the report
  path, or an instruction file resolves to a symlink, Paper aborts rather
  than following it.
- `.paper/latest.json` is written atomically (temp file + rename), so a
  crash mid-write cannot leave a half-written artifact that `paper verify`
  would then bless.
- The managed instruction block only ever touches its own fenced section
  (`<!-- paper:start -->` … `<!-- paper:end -->`) in your agent-instruction
  files. Everything outside the fence is left byte-identical.

## Evidence integrity

Paper does not diagnose or fix your failure, and — by design — the coding
agent never writes the Incident Report; it only reads it. The report is
produced by the wrapper at the OS boundary, outside the agent's control.

This matters: October 2026 research on agent security (Adversa's agent
vulnerability roundup) documents that coding agents can delete or tamper
with their own execution traces without monitors noticing — logs the agent
can write are not evidence. Paper's log is produced entirely out-of-band,
so the agent cannot tamper with it through prompt manipulation or context
pollution.

That was the architectural claim. It is now also cryptographic. A plain
hash would be forgeable by anyone who edits the file — recompute and move
on — so receipts are HMAC-SHA256 under a machine-local key the agent
doesn't have. Forging a valid receipt requires the key, which lives
outside the repo workspace. The per-field hashes mean verification doesn't
just say "tampered," it says what: `stderr was modified after capture`.

The receipt seals the redacted output, not the raw capture: when
`--redact` is on, the HMAC covers exactly what was stored — which is what
the agent ingested. An auditor can therefore prove the agent only ever saw
the scrubbed version. That is the whole blame-limiting story in one
property.

Stated plainly, the limits: a sufficiently motivated process with shell
access could go key-hunting outside the workspace — the bar is "deliberate
jailbreak," not "accidental edit." Key loss orphans old receipts. The
history chain cannot detect removal of the final entries. Fencing is a
directive, not enforcement — Paper tells the agent to work in the
worktree; it cannot jail it there.

## Reporting a vulnerability

Found a bug or a security issue? Email **varmanvishnu96@gmail.com** or post
in the **#bugs** channel on the Paper Discord. Please include the Paper
version (`paper --version`) and a minimal reproduction.

---

*Verification log:*
- *v1.0.12 (2026-10-07): bundle network-primitive grep and npm metadata
review — zero matches for `fetch`, `XMLHttpRequest`, `WebSocket`,
`http(s).request`, `net.connect`, `dns.lookup`, `tls.connect`,
`socket.io`, and no `node:http/https/net/dns/tls/dgram/http2` imports.*
- *v1.0.13 (2026-10-11): bundle re-audited — zero matches for the same
network primitives; all imports are `node:` builtins (`child_process`,
`crypto`, `fs`, `os`, `path`, `url`); zero runtime dependencies in
`package.json`. Bundle ~25 KB.*
- *v1.0.16 (2026-10-13): bundle re-audited — zero matches for the same
network primitives; signal fields (`signal`, `signal_code`,
`probable_cause`) added to receipt coverage. Bundle ~27 KB.*

*Re-audit on each release with the command above.*

