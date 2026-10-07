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

## What Paper does not do

- No network calls of any kind: no telemetry, no analytics, no model API
  calls, no crash reports, no command-output uploads, no update checks.
- No background processes or daemons.
- No credential handling: Paper never reads, stores, or forwards API keys,
  tokens, or secrets beyond what your own failing command prints to its own
  stderr (which is written to the local report verbatim, exactly as captured).

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
3. **Small binary.** The shipped bundle is ~19 KB. You can read all of it.

## Data flow

Everything Paper does happens on your machine:

- `paper run <cmd>` executes your command locally and forwards its output.
  On failure, Paper writes `.paper/incident_report.md` and
  `.paper/latest.json` (structured fields plus the full markdown report) in
  the current repository. On success, nothing is written.
- `paper shelf` moves the active report out of the repository to the local
  shelf (`%LOCALAPPDATA%\Paper\shelf\` on Windows,
  `$XDG_DATA_HOME/Paper/shelf/` or `~/.local/share/Paper/shelf/` elsewhere).
  Reports never leave the disk they were written to.
- Paper adds/updates a managed workflow block in the repository's
  agent-instruction file (e.g. `AGENTS.md`) so coding agents know where to
  find the report. This is local file I/O only.

No step in this flow opens a socket.

## Evidence integrity

Paper does not diagnose or fix your failure, and — by design — the coding
agent never writes the Incident Report; it only reads it. The report is
produced by the wrapper at the OS boundary, outside the agent's control.
This matters: October 2026 research on agent security (Adversa's agent
vulnerability roundup) documents that coding agents can delete or tamper
with their own execution traces without monitors noticing — logs the agent
can write are not evidence. Paper's log is one the agent cannot reach.

## Reporting a vulnerability

Found a bug or a security issue? Email **varmanvishnu96@gmail.com** or post
in the **#bugs** channel on the Paper Discord. Please include the Paper
version (`paper --version`) and a minimal reproduction.

---

*Verification log: bundle network-primitive grep and npm metadata review
performed against @varman96/paper v1.0.12 on 2026-10-07. Re-audit on each
release with the command above.*
