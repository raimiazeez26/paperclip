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
| `pnpm dev:stop` | Stop them. Prefer this over killing the terminal (see §5). |
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

If that process is gone, delete the file and start again. Preferring
`pnpm dev:stop` over Ctrl+C / closing the terminal avoids this entirely.

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
