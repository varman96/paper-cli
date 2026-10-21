# Security

Paper is a local-first CLI. This document describes its threat model, data handling, and how to verify these claims independently.

## Threat model

Paper operates at the failure-capture point: it spawns a command, observes its stdout, stderr, exit code, and surrounding context (command, working directory, timestamp), and writes a structured Incident Report for a coding agent to read. Standard error and stack traces are where secrets leak, so this position is treated as privileged.

The governing property: **Paper contains no network code. There is no exfiltration path in the shipped binary.**

The `--read` layer (v1.0.21) extends the capture point: with an explicit per-run grant, Paper may also read failing test source files and structured test output. This is the only file-reading Paper performs, and it happens only when the operator passes `--read`. The grant names what Paper will access; `.paper/read_receipt.md` proves afterward what it touched. File reading without the flag does not exist in the binary.

The remaining transfer risk is downstream. A coding agent that reads the local report may submit its contents to an external model API. Paper cannot control the agent's behavior. It can ensure the agent ingests scrubbed output where requested, and prove afterward exactly what was captured. The controls below serve that purpose.

## What Paper does not do

- No network calls of any kind: no telemetry, no analytics, no model API calls, no crash reports, no output uploads, no update checks.
- No background processes or daemons.
- No file reads without an explicit grant: outside `--read`, Paper reads nothing from the repository beyond the git state it already captured (HEAD plus uncommitted-change fingerprint). With `--read`, it reads only failing test source files and the test run's own structured output, within the granted scope.
- No credential handling beyond capture: Paper never reads, stores, or forwards API keys, tokens, or secrets except as they appear in the failing command's own stderr or, with `--read`, in a failing test's source. By default that output is written to the local report verbatim. `paper run --redact` applies an opt-in, offline, deterministic regex pass that replaces known secret patterns (AWS keys, GitHub/GitLab tokens, bearer and api_key assignments, private key blocks, credentialed connection strings, webhook URLs) with `[REDACTED:<label>]` before anything is written. With `--read`, redaction runs on source files before hashing, so the sealed artifact never contains raw PII. The pattern list is fixed and not exhaustive. Reports are written with default file permissions; on shared machines, prefer `--redact`.

## The --read layer

`paper run --read -- <command>` grants Paper permission to read the run's failing test files. `paper run --read <paths> -- <command>` names the files explicitly. The flag is the entire permission: there is no interactive prompt, because the agent harness already obtained confirmation before execution. Passing the flag declares intent ("Paper will access this specific file"); `.paper/read_receipt.md` confirms execution.

Framework detection runs pytest first (structured JSON, then stdout), then JUnit XML. If no test output is parseable, the layer contributes nothing: no error, no reads. A clean pass reads zero files and prints `READ LAYER IDLE`; the grant is still logged in `history.jsonl`.

Hardening, all enforced before any read:

- Path containment: every path is canonicalized, symlinks resolved, `..` collapsed. Anything escaping the granted scope is rejected. In named-file mode, a symlink pointing at an unnamed file is rejected even inside the repo.
- Only regular files: FIFOs, sockets, and device nodes are skipped. Files over 256 KB are rejected. Non-text files are rejected.
- Read-once, hash-immediately: each file is read once and SHA-256 hashed at read time.
- Files open read-only. If a failing test's source was deleted before the read, Paper seals the rest of the capture, records the gap, and exits 66 (`READ SOURCE MISSING`).

Every file touched, every rejection, and the SHA-256 of `read_receipt.md` itself are recorded in `latest.json` under `read_grant`, covered by the receipt HMAC. `paper verify` recomputes the receipt hash and fails closed with `READ SCOPE EXCEEDED` if the touch log lists anything outside the granted scope. The agent instruction block carries the touch record alongside the report pointer, so the agent's first context includes exactly what Paper read.

## Verifiable claims

Each claim below can be checked in about ten minutes against the published package.

1. **Bundle audit.** The npm package ships a single bundled file, `dist/paper.js`, containing no network primitives. Verified against v1.0.12 (2026-10-07): grep of the shipped bundle for `fetch`, `XMLHttpRequest`, `WebSocket`, `http(s).request`, `net.connect`, `dns.lookup`, `tls.connect`, `socket.io`, and any `node:http/https/net/dns/tls/dgram/http2` import returned zero matches. Reproduce: `npm pack @varman96/paper && tar xzf *.tgz && grep -oiE '\b(fetch|XMLHttpRequest|WebSocket)\b' package/dist/paper.js`.
2. **Dependency surface.** `package.json` declares zero runtime dependencies (dev-only tooling: esbuild, wrangler, biome). There is no transitive dependency tree.
3. **Small binary.** The shipped bundle is approximately 27 KB and fully readable.
4. **Receipt verification.** Every capture is sealed with an HMAC-SHA256 receipt under a machine-local key. `paper verify` recomputes the per-field hashes and the HMAC from `.paper/latest.json`, printing `RECEIPT OK` or identifying the altered field (e.g., `stderr was modified after capture`). The per-field hashes cover `signal`, `signal_code`, and `probable_cause` alongside the other captured fields, plus the `--read` fields (`test_results`, `test_source`, `read_grant`) and the `read_receipt.md` hash when the layer runs. Reproduce: modify the stored `stderr` and run `paper verify`.
5. **Read-layer audit.** With `--read`, `paper verify` checks the touch log against the granted scope. Reproduce: hand-edit `read_grant` in `.paper/latest.json` to list a file outside the grant and run `paper verify`; expect `READ SCOPE EXCEEDED`.

## Data flow

All processing occurs on the local machine:

- `paper run <command>` executes the command locally and forwards its output. On failure, Paper writes `.paper/incident_report.md` and `.paper/latest.json` (structured fields plus the full markdown report) in the current repository. On success, nothing is written.
- With `--read`, a failing run also writes `.paper/read_receipt.md`: the grant, every file touched with hashes, rejections, and the evidence payload handed to the agent. Its hash is stamped in `latest.json` under the HMAC. A later failed run without `--read` removes the stale receipt, since its hash no longer matches `latest.json`.
- Each capture appends one JSON line to `.paper/history.jsonl`: timestamp, command, exit code, receipt hash, and flags. Each entry includes an HMAC of the previous entry's hash, forming a hash chain. `paper verify --chain` checks the sequence and reports the index of the first break. Appending is best-effort and never fails a run. Shelving a report does not remove its history entry.
- The receipt key is stored at `<data dir>/receipt.key`: 32 random bytes, mode `0600`, created on first capture (`%LOCALAPPDATA%\Paper\receipt.key` on Windows; `~/.local/share/Paper/receipt.key` or `$XDG_DATA_HOME` elsewhere). Since v1.0.20 the key rotates every run: the new key is written to `<data dir>/receipt-keys/<id>` then moved into place atomically, and each rotation is logged in an append-only, HMAC-chained `rotation.log.jsonl` back to a genesis fingerprint the operator saves once. `paper verify` walks the chain; a broken link reports `ROTATION CHAIN BROKEN`. The key is machine-local: receipts do not verify on other machines, and key loss orphans old receipts (`paper verify` reports `KEY MISSING`, distinct from a tampering finding). On Windows, Node cannot set POSIX-style permissions, so `0600` is best-effort there.
- `paper run --fence` (opt-in) creates a git worktree at `.paper/scratch` on failure, directing diagnosis off the main tree; `paper shelf` removes it. Fencing does not block capture. With `--read`, source files are read from the tree the command ran in, not the worktree.
- `paper shelf` moves the active report out of the repository to the local shelf (`%LOCALAPPDATA%\Paper\shelf\` on Windows; `$XDG_DATA_HOME/Paper/shelf/` or `~/.local/share/Paper/shelf/` elsewhere). Reports do not leave the disk they were written to. `paper shelf` also removes `.paper/read_receipt.md` with the report.
- Paper adds or updates a managed workflow block in the repository's agent-instruction file (e.g., `AGENTS.md`) so coding agents can locate the report. This is local file I/O only. With `--read`, the block also carries the file-touch record; a later run without the flag restores the base block byte-for-byte.

No step in this flow opens a socket.

## Filesystem behavior

- Paper refuses to write through symbolic links. If `.paper`, the report path, or an instruction file resolves to a symlink, Paper aborts instead of following it.
- `.paper/latest.json` is written atomically (temporary file plus rename). A crash mid-write cannot produce a partial artifact that `paper verify` would accept.
- The managed instruction block is confined to its fenced section (`<!-- paper:start -->` … `<!-- paper:end -->`) in agent-instruction files. Content outside the fence is left byte-identical.

## Evidence integrity

Paper does not diagnose or fix failures. The coding agent reads the Incident Report; it never writes it. The report is produced by the wrapper at the OS boundary, outside the agent's control.

This separation matters because agent-written logs are not reliable evidence: Adversa AI's October 2026 coding-agent security roundup documents agents deleting or altering their own execution traces without monitor detection ([Adversa AI, "AI coding agent vulnerabilities, October 2026: GitSpawn and more," October 2026](https://adversa.ai/blog/top-ai-coding-agent-security-resources-october-2026/)). Paper's log is produced out-of-band, so prompt manipulation or context pollution cannot alter it.

The integrity guarantee is cryptographic. A plain hash would be forgeable by anyone able to edit the file and recompute it, so receipts use HMAC-SHA256 under a machine-local key held outside the repository workspace. Forgery requires the key. Per-field hashes identify which field changed.

Where `--redact` is enabled, the HMAC covers the stored (redacted) output, including redacted test source under `--read`: the same content the agent ingested. An auditor can therefore establish that the agent saw only the scrubbed version.

Limitations: a process with shell access could deliberately locate the key outside the workspace; the protection assumes deliberate exfiltration effort rather than accidental edits. Key loss orphans old receipts. The history chain cannot detect removal of trailing entries. Fencing directs the agent to the worktree but cannot enforce it. Full machine compromise (attacker reads all keys, past and present) remains outside the model.

## Scope and responsibility

**1. Capture scope.** For each captured run, Paper records: the exact command and arguments; the exit code or, where the process was terminated by a signal, the signal name, the conventional 128+N signal code, and the fixed one-line probable-cause hint; the child process's stdout and stderr; the working directory; a timestamp; and git state, consisting of HEAD plus a fingerprint of uncommitted changes. With `--read`, Paper additionally records: the parsed test results (failing test names, failure output, counts), the source of failing test files only, and the read grant with the full touch log. Paper records nothing outside this set.

**2. Exclusions.** Without `--read`, Paper does not inspect system state beyond what the operating system reports for the child process. It does not read kernel logs, process filesystems, core dumps, or environment variables beyond those required for command resolution. With `--read`, file access is limited to the granted scope described above. It initiates no network connections and performs no telemetry. Paper does not direct the agent's investigation and does not execute remediation actions.

**3. Allocation of responsibility.** The coding agent is solely responsible for its actions following ingestion of the Incident Report, including any files accessed, commands executed, or remediations applied. Paper's managed instruction block directs the agent to obtain explicit user approval before remediation; enforcement of that directive is the responsibility of the agent harness. Paper makes no representation as to the agent's compliance with that directive.

**4. Verifiability.** The HMAC receipt seals the captured record, or, where `--redact` is enabled, the redacted record the agent was permitted to see. An auditor may thereby establish precisely what the agent was given, and what it was not.

## Reporting a vulnerability

Report bugs or security issues to **varmanvishnu96@gmail.com** or the **#bugs** channel on the Paper Discord. Include the Paper version (`paper --version`) and a minimal reproduction.

---

*Verification log:*
- *v1.0.12 (2026-10-07): bundle network-primitive grep and npm metadata review: zero matches for `fetch`, `XMLHttpRequest`, `WebSocket`, `http(s).request`, `net.connect`, `dns.lookup`, `tls.connect`, `socket.io`; no `node:http/https/net/dns/tls/dgram/http2` imports.*
- *v1.0.13 (2026-10-11): bundle re-audited: zero matches for the same network primitives; all imports are `node:` builtins (`child_process`, `crypto`, `fs`, `os`, `path`, `url`); zero runtime dependencies in `package.json`. Bundle ~25 KB.*
- *v1.0.16 (2026-10-13): bundle re-audited: zero matches for the same network primitives; signal fields (`signal`, `signal_code`, `probable_cause`) added to receipt coverage. Bundle ~27 KB.*
- *v1.0.20 (2026-10-20): key rotation shipped. Rotation log chain verified: genesis fingerprint recorded once, each rotation HMAC-linked to the previous key, `paper verify` walks to genesis. `ROTATION CHAIN BROKEN` and `KEY UNTRUSTED` outcomes tested.*
- *v1.0.21 (2026-10-21): read layer shipped. Bundle re-audited: zero matches for the same network primitives; new module `scripts/read-layer.js` uses `node:` builtins only. `READ SCOPE EXCEEDED` tested against a tampered touch log; `read_receipt.md` hash verified against `latest.json`.*

*Re-audit on each release with the command above.*

