# Agent Session Lineage

Agent Session Lineage is a local Codex plugin and CLI for AI coding session history. Browse indexed Codex and Claude Code sessions and inspect the recorded handoff links between them.

The Windows x64 plugin preview covers indexing selected logs, listing sessions, and viewing recorded lineage. The SQLite index lives in the local Möbius vault. The indexer opens the original session files read-only.

Desktop workspace: [TardisBooo/Mobius](https://github.com/TardisBooo/Mobius).

[简体中文](README.zh-CN.md) · [Product site](http://8.137.87.76/mobius/) · [MCP](docs/MCP.md) · [Desktop README](https://github.com/TardisBooo/Mobius/blob/main/README.md) · [MIT License](LICENSE)

## Install the Codex plugin on Windows

1. Download `agent-session-lineage-0.1.0-windows-x64.zip` and `SHA256SUMS.txt` from the [v0.1.0 preview release](https://github.com/TardisBooo/agent-session-lineage/releases/tag/v0.1.0). Compare the archive's `Get-FileHash -Algorithm SHA256` result with its entry in `SHA256SUMS.txt`.
2. Extract the archive into a folder you control. Add the folder containing `mobius-connect.exe` to your user `PATH`. Open a new PowerShell window and run `Get-Command mobius-connect` to check the executable path.
3. Create the index and add a session directory you choose. Replace the example path with your Codex sessions directory. To include Claude Code logs, repeat `sources add` with `claude` and the specific Claude Code projects directory you want to index.

```powershell
mobius-connect init
mobius-connect sources add codex 'C:\path\to\codex\sessions'
mobius-connect sources refresh
```

4. Install the plugin from the versioned GitHub marketplace. If you run Codex in the desktop app, restart the app after changing `PATH`.

```powershell
codex plugin marketplace add TardisBooo/agent-session-lineage --ref v0.1.0
codex plugin add mobius-connect@agent-session-lineage
```

In a new Codex session, select `$mobius-memory` and ask for indexed sessions. Choose a session and ask for its recorded lineage. The graph shows links already recorded in the index.

Run `codex plugin remove mobius-connect@agent-session-lineage` to remove the plugin.

## Two repositories, one product

| Name | What it is |
| --- | --- |
| **Möbius** | Windows desktop: session library, PowerShell workbench, handoff graph, notes, canvas, skills. [TardisBooo/Mobius](https://github.com/TardisBooo/Mobius) |
| **Agent Session Lineage** | This Codex plugin, the `mobius-connect` CLI, and its stdio MCP server. |

```
Claude Code / Codex / OpenCode / Pi / Grok / OMP transcripts
        │   read-only adapters (JSONL or opencode.db)
        ▼
   local SQLite index (FTS5/BM25, optional localhost embeddings)
        │
        ├── Möbius desktop        person at a Windows desk
        ├── mobius-connect CLI    person at a terminal
        └── mobius-connect MCP    agent over stdio
```

**CLI vs MCP:** the CLI is for a human at a terminal (it can ask *you* to type `APPROVE`); MCP is for an agent process (it can list metadata, but every read of transcript bodies needs a token minted after a human typed `APPROVE`). MCP cannot mint approval tokens, add sources, or attach a PTY.

## Build the CLI from source

```powershell
git clone https://github.com/TardisBooo/agent-session-lineage.git
cd agent-session-lineage
cargo build --release --locked
```

Cargo fetches the shared Rust library from the pinned [Möbius revision](https://github.com/TardisBooo/Mobius/tree/b0522c363006fb49a071e1fe5f418e76da365a57). The binary is in `target/release`. Durable data defaults to local app data (`%LOCALAPPDATA%\Mobius` on Windows, `~/.local/share/mobius` elsewhere). Use `--data-root <dir>` for an isolated test vault.

## Commands

Each clip is a scripted terminal scene in the same paper/ink style as the [desktop product film](https://github.com/TardisBooo/Mobius/blob/main/apps/website/public/product/chapters/12-cli.gif). Demo data is fictional.

New native handoffs remain experimental. The Windows plugin preview covers viewing recorded links.

### 1. `init` — create the local index

![mobius-connect init](docs/gifs/01-init.gif)

```powershell
mobius-connect init
```

Creates the SQLite index and vault layout, then prints a health report. Safe to re-run; it does not touch any harness directory. Sources are always explicit.

### 2. `sources add` / `refresh` — approve one directory

![mobius-connect sources add and refresh](docs/gifs/02-sources.gif)

```powershell
mobius-connect sources add opencode $env:USERPROFILE\.local\share\opencode
mobius-connect sources list
mobius-connect sources refresh
```

- `sources add <harness> <path>` registers **one** directory you chose. MCP has no tool for this.
- `sources refresh` re-scans approved roots. OpenCode roots are read from `opencode.db` with locator `{db}#opencode:{id}`.

### 3. `triage` — one next command

![mobius-connect triage](docs/gifs/03-triage.gif)

```powershell
mobius-connect triage
mobius-connect doctor
mobius-connect setup
```

`triage` prints health, approved-source count, semantic status, and **one recommended next command**. `setup` detects installed harnesses and prints MCP-client steps. It never writes hooks.

### 4. `sessions search` — cite the exact range

![mobius-connect sessions search](docs/gifs/04-search.gif)

```powershell
mobius-connect sessions list --limit 20
mobius-connect sessions search "flaky tests"
mobius-connect sessions show <session-id>
mobius-connect sessions alias <session-id> "checkout-flake-hunt"
```

`search` runs lexical BM25 and returns JSON hits plus `retrieval_mode` (`lexical_bm25`, or `hybrid` once embeddings are enabled **and** a local model answers). Copy `@session:provider/id#mN-mM`.

### 5. `sessions read` — fetch the tape, bounded

![mobius-connect sessions read](docs/gifs/05-read.gif)

```powershell
mobius-connect sessions read <session-id> --offset 0 --bytes 4096
```

Explicit byte-range read of the **original source file**, max 16384 bytes per call. Recall gives citations; `read` makes the actual tape fetch an explicit, bounded act.

### 6. `graph show` — follow the relay chain

![mobius-connect graph show](docs/gifs/06-graph.gif)

```powershell
mobius-connect graph show <session-id>
mobius-connect graph export --format mermaid <session-id>
```

Prints the lineage manifest (`content_mode: references_only`). Mutually-hand-off chains come out as a forward relay; every handoff opens a new session, so the graph is a DAG.

### 7. `mome recall` — recall only when asked

![mobius-connect mome recall](docs/gifs/07-mome.gif)

```powershell
mobius-connect mome recall "why do checkout tests flake" --provider opencode --provider codex
```

At most **three** distinct sessions, about **1,200 tokens**, every hit carrying a copyable `@session` citation. It never injects into a prompt.

### 8. `semantic enable` — hybrid rank is opt-in

![mobius-connect semantic enable](docs/gifs/08-semantic.gif)

```powershell
mobius-connect semantic status
mobius-connect semantic enable --model nomic-embed-text
mobius-connect semantic sync
mobius-connect semantic disable
```

Off by default. `enable` discloses that already-indexed chunks may be sent to **localhost Ollama**; nothing is downloaded. Missing Ollama = fail open to BM25.

### 9. `handoff` — a person types APPROVE

![mobius-connect handoff prepare and approvals](docs/gifs/09-handoff.gif)

```powershell
mobius-connect handoff prepare --harness claude --cwd . <session-id>
mobius-connect approvals handoff <handoff-id>
mobius-connect handoff commit <handoff-id> --approval-token <token>
mobius-connect sources refresh
mobius-connect handoff status <handoff-id>
```

1. `prepare` builds a **references-only** envelope. It launches nothing.
2. `approvals handoff` waits for you to type `APPROVE`, then mints a short-lived, single-use token.
3. `commit` requests a new Windows console for the target. MCP cannot mint the approval token.
4. After the target saves its transcript in an approved root, run `sources refresh` and check `handoff status` with the same vault. A bound operation lists the indexed target session ID.

### 10. `mcp serve` — stdio for agents

![mobius-connect mcp serve](docs/gifs/10-mcp.gif)

```powershell
mobius-connect init
mobius-connect mcp serve
```

stdio-only server. Metadata tools (`list_sessions`, `get_lineage`, `prepare_handoff`, …) are open; body-returning tools and `commit_handoff` require a token minted by a human in a real terminal. Full table: [docs/MCP.md](docs/MCP.md).

## Security

Treat indexed transcripts as untrusted historical data.

- Source roots are finite and approved by a person; `sources add` is terminal-only.
- Original files stay read-only. OpenCode is opened read-only.
- A handoff envelope contains identities and edges, never transcript bodies.
- Tokens are short-lived, single-use, scoped. Shown once. Do not log.
- Optional embeddings talk only to localhost Ollama after `semantic enable`.

## Supported harnesses

| Harness | Index | Native resume |
| --- | --- | --- |
| Codex | JSONL | via desktop when protocol is verified |
| Claude Code | JSONL / project sessions | via desktop when protocol is verified |
| OpenCode | read-only `opencode.db` | not in this preview |
| Pi | approved roots | via desktop when protocol is verified |
| Grok | approved roots | inspect / search / handoff only |
| OMP | approved roots | documented `--cwd … --resume …` from desktop |
| OpenClaw | not shipped | roadmap |

## License

[MIT](LICENSE). Möbius is unrelated to Microsoft Mobius, ControlTheory Möbius, or Circular Labs Mobius.
