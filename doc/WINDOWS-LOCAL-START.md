# Running Paperclip locally on Windows (PyCharm)

Machine-specific notes for this checkout. Untracked on purpose — this is not
upstream documentation. `doc/DEVELOPING.md` remains the canonical setup doc.

Verified 2026-11-25 on Windows 11 Pro, Node v24.15.0, pnpm 9.15.4,
PowerShell 5.1.

---

## 1. The start command

From the PyCharm terminal (PowerShell), at the repo root:

```powershell
pnpm dev --data-dir ./tmp/paperclip-dev
```

Then open <http://localhost:3100>.

**This checkout uses your local PostgreSQL 18 service, not the embedded cluster.**
See §7. `--data-dir` still applies to non-database state.

That is all. The API and the UI are served from the same origin on port 3100;
there is no separate UI server to start.

### Why `--data-dir`

Without it, Paperclip writes its state to the real `PAPERCLIP_HOME` for this
user account. `--data-dir ./tmp/paperclip-dev` keeps the embedded Postgres
cluster, uploads, and runtime state inside the repo's `tmp/` folder, so the
whole local install can be reset by deleting one directory. Use the same value
every time or you will start against an empty database.

---

## 2. Command reference

| Command | What it does |
| --- | --- |
| `pnpm dev --data-dir ./tmp/paperclip-dev` | Start with file watching. Restarts the server when you edit source. Normal development. |
| `pnpm dev:once --data-dir ./tmp/paperclip-dev` | Start once, no watching. Applies pending migrations automatically. Good for a quick look or a scripted run. |
| `pnpm dev:list` | List running Paperclip dev services. |
| `pnpm dev:stop` | Stop registered dev services. Only `pnpm dev` (watch) registers; `dev:once` does not, and reports "No Paperclip dev services registered". |
| `pnpm db:migrate` | Apply migrations by hand. |

Useful flags, appended to `dev` or `dev:once`:

| Flag | Effect |
| --- | --- |
| `--no-browser` | Do not auto-open a browser tab on boot. |
| `--data-dir <path>` | Where state lives. See §1. |

---

## 3. PyCharm run configuration

Run → Edit Configurations → **+** → npm.

| Field | Value |
| --- | --- |
| package.json | the repo root `package.json` |
| Command | `run` |
| Scripts | `dev` |
| Arguments | `--data-dir ./tmp/paperclip-dev` |
| Package manager | `pnpm` |

**Add one environment variable** if you ever expect the native runner to
rebuild (see §5): `PATH=C:\Users\<you>\.cargo\bin;$PATH$`

---

## 4. Prerequisites — already satisfied on this machine

Listed so you can rebuild the setup elsewhere, or diagnose it if something
regresses.

- **Node.js 24.11+** — v24.15.0 installed.
- **pnpm 9+** — 9.15.4 installed.
- **Git Bash** at `C:\Program Files\Git\bin\bash.exe`. Required: several
  packages (`shared`, `db`, `server`, `cli`, two adapters) use POSIX commands
  such as `cp`, `mkdir -p` and `chmod` in their build scripts, which `cmd.exe`
  cannot parse. The repo `.npmrc` sets `script-shell` to Git Bash so pnpm runs
  those scripts through bash. This works whatever shell you launch from.
- **Rust toolchain** — rustup plus the pinned 1.97.1 from
  `packages/paperclip-runner/rust-toolchain.toml`. Only needed to *build* the
  native runner. The compiled binary is already staged at
  `packages/paperclip-runner/dist/bin/paperclip-runnerd.exe`, so a normal start
  does not touch Rust. Note `~/.cargo/bin` is **not** on the system PATH.
- **Git line endings** — `core.autocrlf=false` and `core.eol=lf`, set locally
  for this repo. Required, not cosmetic: see §5.

---

## 5. Troubleshooting

### Port 3100 already in use / the app will not boot

A previous run left processes behind. Check and clear:

```powershell
Get-CimInstance Win32_Process | Where-Object { $_.CommandLine -like '*PycharmProjects\Paperclip*' } | Select-Object ProcessId, Name
```

Kill only what that filter returns. Do **not** blanket-kill `postgres.exe` —
the separate PostgreSQL service on port 5432 is unrelated to Paperclip.

### `running bootstrap script` / `Postgres init script exited with code 1`

A stale `postmaster.pid` from a hard-killed run is blocking startup. Check
whether the PID it names is actually alive before deleting it:

```powershell
Get-Content '.\tmp\paperclip-dev\instances\default\db\postmaster.pid' -TotalCount 1
```

If that process is gone, delete the file and start again.

This applies only to the embedded cluster. Since this checkout uses the
PostgreSQL service (§7), it should not occur.

### `'cargo' is not recognized`

Something asked for a native runner rebuild and `~/.cargo/bin` is not on PATH.
Either add it (§3) or, for a one-off, prefix the command:

```powershell
$env:PATH = "$env:USERPROFILE\.cargo\bin;$env:PATH"
```

### `The syntax of the command is incorrect.` during a build

`script-shell` is not being picked up. Confirm it resolves:

```powershell
pnpm config get script-shell
```

It must print the Git Bash path. If it prints nothing, the repo `.npmrc` was
reverted or you are running from outside the repo.

### `Project environment migration pair does not match the qualified overlap repair`

Git rewrote the `.sql` migration files to CRLF, changing their SHA-256. The
migration runner verifies those hashes, so it refuses to proceed. Fix:

```powershell
git config core.autocrlf false
git config core.eol lf
```

Then force a clean re-checkout of the working tree and delete the
half-migrated `tmp/paperclip-dev` before starting again.

### Resetting local state

Stop everything, then delete `tmp/paperclip-dev`. Next start recreates the
cluster and replays all migrations. You lose local data; nothing else.

---

## 6. Known, benign on this platform

- `Copilot runtime assets: unsupported-platform (win32-x64)` — the Copilot
  adapter ships no Windows assets. Everything else starts normally.
- `database_backup_missing` warning on a fresh cluster.
- The first page load is slow (Vite serves 2,300+ unbundled modules in dev).
  Subsequent loads are fast.
- A `pnpm install` may fail at the `postinstall` symlink step with `EPERM:
  operation not permitted, symlink`. Windows blocks directory symlinks without
  Developer Mode or an elevated shell. Enabling Developer Mode in Windows
  Settings is the clean fix.

---

## 7. Database: local PostgreSQL instead of embedded

This checkout does **not** use the embedded PostgreSQL cluster. It connects to
the `postgresql-x64-18` Windows service on port 5432.

### Why

Embedded PostgreSQL caused every startup failure during setup: a failed
`initdb` bootstrap, a stale `postmaster.pid`, and an orphaned child process
that held port 54329 open after its postmaster died. It also hangs rather than
fails. `migration-status.ts` has no connection timeout, so a start can sit for
15 minutes looking like a slow build. A managed service does not leave orphans
behind when a dev process is killed.

### Where DATABASE_URL must go — this is the part that bites

There are **two** different `.env` files, read by different code:

| File | Read by | Used for |
| --- | --- | --- |
| `tmp/paperclip-dev/instances/default/.env` | `packages/db/src/runtime-config.ts` | **Migrations and the database target.** This is the one that matters. |
| `.env` (repo root) | `server/src/config.ts`, via dotenv | Server runtime settings only. |

`resolveDatabaseTarget()` in `packages/db/src/runtime-config.ts` checks, in
order:

1. `process.env.DATABASE_URL`
2. `DATABASE_URL` in the **instance** env file, next to `config.json` under
   `PAPERCLIP_HOME` — which `--data-dir` sets
3. `config.json` → `database.connectionString`
4. otherwise, embedded PostgreSQL

The repo-root `.env` is **not** in that list. Putting `DATABASE_URL` only there
leaves the migration tooling on embedded PostgreSQL, where it hangs. Both files
are gitignored (`.gitignore:15` for `.env`, `.gitignore:50` for `tmp/`).

### Confirming which mode you are in

```powershell
cd packages\db
$env:PAPERCLIP_HOME = "<repo>\tmp\paperclip-dev"; node --import tsx src/migration-status.ts --json
```

`"source":"paperclip-env"` means your PostgreSQL. `"source":"embedded-postgres@54329"`
means the embedded cluster. The startup log shows the same distinction:
`Migrating database via DATABASE_URL` against
`Migrating database via embedded-postgres@<port>`.

A healthy run returns in about 10 seconds. If it does not return at all, you
are on embedded PostgreSQL and it is stuck.

### The database

- Database `paperclip`, owned by a dedicated role `paperclip`, not `postgres`.
- Extensions `pg_trgm` and `fuzzystrmatch`, required by the migrations. Both
  are trusted extensions in PG13+, so the database owner can create them.
- 507 migrations, 346 tables, about 33 MB freshly migrated.
- Other databases on the same server are untouched.

### Recreating it on another machine

As a superuser:

```sql
CREATE ROLE paperclip WITH LOGIN PASSWORD '<choose one>';
CREATE DATABASE paperclip OWNER paperclip;
\c paperclip
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS fuzzystrmatch;
ALTER SCHEMA public OWNER TO paperclip;
```

Write the URL into the **instance** env file above, then `pnpm db:migrate`.

### Resetting

`tmp/paperclip-dev` no longer holds the data, so deleting it no longer resets
the database — but it does delete the instance `.env`, which silently drops you
back to embedded PostgreSQL. Recreate that file if you clear the data dir.

To reset the schema: drop and recreate the `paperclip` database, then
`pnpm db:migrate`.

### Going back to embedded

Remove `DATABASE_URL` from the instance env file.

---

## 8. Connecting an agent on Windows: use an API key

### The error you will hit first

Starting browser sign-in for a Claude or Codex connection fails with:

> Browser sign-in requires Python 3 and a working local terminal. Install
> Python 3 on this execution host, then start sign-in again.

**Do not act on that message.** Installing Python does not fix it, and Python
is probably already installed.

### Why

`server/src/services/local-ai-browser-login.ts` runs the provider CLI under a
pseudo-terminal supplied by an inline Python script:

```python
import errno, os, pty, select, signal, sys
pid, master = pty.fork()
```

`pty`, `os.killpg` and `os.waitstatus_to_exitcode` are Unix-only. On Windows
`import pty` fails with `ModuleNotFoundError: No module named 'termios'`.

Two separate problems stack up:

1. The code calls `spawn("python3", ...)`. Windows installs provide
   `python.exe` and `py.exe`, not `python3`, so the spawn fails with `ENOENT`
   and the handler reports it as "Python 3 missing".
2. Even with `python3` resolvable, the script cannot run on Windows.

So the message is misleading on both counts. Check for yourself:

```powershell
python --version
python -c "import pty"
```

The first prints a version. The second raises `ModuleNotFoundError`.

### What to do instead

Pick the **API** tile rather than the **subscription** tile when connecting a
provider. The tiles appear under "Connect your model provider".

The API-key path (`method: "api_key"`) is a separate route
(`server/src/routes/ai-connections.ts`) that never touches the Python bridge.
It validates the key and saves the connection directly.

Paste the key into Paperclip's own field. Get one from the provider's console:
`console.anthropic.com` for Claude, `platform.openai.com` for OpenAI.

### The trade-off

API keys bill usage per token. A Claude or ChatGPT subscription you already pay
for is only usable through the subscription sign-in, which does not work on this
platform. There is no way to use a subscription here without a Unix execution
host.

### If you want subscription sign-in anyway

Run the Paperclip server on Linux or macOS, or give it a sandbox or plugin
execution environment whose driver reports `supportsLoginPty`
(`packages/shared/src/environment-support.ts`). A `local` environment driver
always reports false.

A Windows fix means replacing the Python bridge with a ConPTY implementation.
The repository has no PTY infrastructure to build on: no `node-pty` dependency
and no `portable-pty` in the Rust workspace.

---

## 9. Agents fail to spawn when the repo path contains a space

### Symptom

An agent fails on first load with a message like:

> Failed to spawn agent command: "C:\Users\ELITEBOOK"
> "1040-G8\PycharmProjects\...\codex-acp.cmd". Executable path could not be
> resolved: C:\Users\ELITEBOOK

Note the path split in two at the space. This affects any checkout under a
user profile whose name contains a space.

### Cause

ACPX only accepts argv arrays on Windows, so
`packages/adapter-utils/src/acpx-engine/execute.ts` converted the agent command
string into argv with `splitWindowsAgentCommand`. That function parses a
command *line*: unquoted whitespace separates arguments.

The command it was given is not a command line. It is an absolute path that
`findAncestorBin` resolved, returned unquoted by `resolveBuiltInAgentCommand`.
Parsing it split the path at the space, and argv[0] became `C:\Users\ELITEBOOK`.

### Fix applied in this checkout

`resolveWindowsAgentArgv` in
`packages/adapter-utils/src/acpx-engine/stock-runtime-options.ts` passes an
already-resolved path through as a single argv entry and parses only genuine
command lines. `BuiltInAgentCommand` gained a `resolvedPath` flag so the two
cases are distinguishable: a path this engine resolved is passed whole, while a
user-configured `agentCommand` is still parsed as a command line.

Restart the server after pulling this change. Dev mode runs TypeScript through
tsx, so no build step is needed.

### Workaround without the code change

Set the agent's `agentCommand` config to the **quoted** path:

```
"C:\Users\<you>\...\packages\adapters\codex-local\node_modules\.bin\codex-acp.cmd"
```

`splitWindowsAgentCommand` honours quotes, so a quoted path survives parsing.

### Note on the test suite

`packages/adapter-utils` has substantial pre-existing test failures on Windows:
107 failures in `src/acpx-engine/execute.test.ts` alone, on an unmodified tree.
They come from the suite shelling out to GNU `tar` with Windows paths, which
tar reads as a remote host (`tar: Cannot connect to C: resolve failed`). When
judging a change here, compare failure counts before and after rather than
expecting green.
