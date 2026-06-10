<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://capsule-render.vercel.app/api?type=waving&height=180&color=0:1a1b27,100:3b82f6&text=Keeper&fontColor=f0f0f0&fontSize=60&section=header&reversal=false&descAlignY=55">
  <img alt="Keeper — macOS Tool Update Guardian" src="https://capsule-render.vercel.app/api?type=waving&height=180&color=0:1a1b27,100:3b82f6&text=Keeper&fontColor=1a1b27&fontSize=60&section=header&reversal=false&descAlignY=55" width="100%">
</picture>

<p align="center">
  <strong>One script to track, check, and update every CLI tool on your Mac.</strong><br>
  <em>No package managers. No daemons. No dependencies. Just your shell.</em>
</p>

<p align="center">
  <a href="#quick-start"><img src="https://img.shields.io/badge/Quick_Start-1a1b27?style=for-the-badge"></a>
  <a href="#what-it-tracks"><img src="https://img.shields.io/badge/What_It_Tracks-3b82f6?style=for-the-badge"></a>
  <a href="docs/how-to-add-a-tool.md"><img src="https://img.shields.io/badge/Adding_Tools-6366f1?style=for-the-badge"></a>
  <a href="#automation"><img src="https://img.shields.io/badge/Automation-22c55e?style=for-the-badge"></a>
</p>

---

## Why Keeper?

If you manage your own tools the homelab way — raw binaries, git clones, tarballs in `~/.local/opt/` — you know the pain:

- **You don't remember** what versions you're running
- **You don't know** when updates are available
- **You skip security patches** because checking each tool manually is tedious

Keeper is a **single zsh script** that solves all three. It knows every tool you've installed manually, checks their upstream for new releases, and can update them all with one command.

### Zero package-manager lock-in

Unlike solutions built on top of package managers (brew, pip, npm), Keeper doesn't require you to change how you install things. It simply observes what you already have and checks upstream. Whether you downloaded a tarball, cloned a repo, or compiled from source — Keeper can track it.

---

## Quick Start

```bash
# Download the script
curl -fsSL https://raw.githubusercontent.com/irfancode/keeper/main/bin/update-tools \
  -o ~/.local/bin/update-tools && chmod +x ~/.local/bin/update-tools

# Run a check
update-tools
```

That's it. Your shell's `$PATH` just needs `~/.local/bin/` on it (add it if not — it's standard on macOS).

```bash
# See what needs updating
update-tools

# Apply all updates
update-tools --update

# Update just one tool
update-tools --update node

# Silence everything (for cron/launchd)
update-tools --quiet
```

---

## What It Tracks

Keeper comes pre-configured for **12 tool categories** spanning 30+ actual packages:

| Category | Tools Managed | Update Source |
|----------|--------------|---------------|
| **neovim** | nvim | GitHub releases — nvim-macos-arm64.tar.gz |
| **node** | node, npm, npx | nodejs.org/dist/ — version-pinned to your major line |
| **fzf** | fzf | Git tags in ~/.fzf/ + build |
| **ripgrep** | rg | GitHub releases — aarch64 binary |
| **starship** | starship | GitHub releases — aarch64 binary |
| **zoxide** | zoxide | GitHub releases — aarch64 binary |
| **opencode** | opencode | GitHub releases — darwin-arm64.zip |
| **gh** | gh (GitHub CLI) | GitHub releases — macOS arm64.zip |
| **go** | go, gofmt | go.dev — darwin-arm64 tarball |
| **brew (casks)** | ghostty, office, notion, zoom, etc. | Homebrew — casks only |
| **npm global** | corepack, neovim client | npm registry — `npm update -g` |
| **pip3** | pynvim, greenlet, msgpack | PyPI — user packages only, system pkgs skipped |

### Architecture overview

```
┌─────────────────────────────────────────────┐
│              update-tools                    │
│  ┌─────────┐  ┌─────────┐  ┌─────────────┐ │
│  │ check_* │  │ check_* │  │  check_*     │ │
│  │ nvim    │  │ node    │  │  ... 12 more │ │
│  └────┬────┘  └────┬────┘  └──────┬──────┘ │
│       │            │              │         │
│       ▼            ▼              ▼         │
│  ┌──────────────────────────────────────┐   │
│  │  latest_gh_release()                 │   │
│  │  gh release view --json tagName     │   │
│  └──────────────────────────────────────┘   │
│       │            │              │         │
│       ▼            ▼              ▼         │
│  ┌──────────────────────────────────────┐   │
│  │  update_* functions                 │   │
│  │  Download → Extract → Symlink       │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

Each tool is defined by **two zsh functions**: one that checks the current version against upstream, and one that performs the update. The `gh` CLI (which Keeper manages itself) handles all GitHub API calls with your authenticated session — no rate limits, no tokens to manage.

---

## Automation

### Daily shell check (lightweight)

Add this to your `.zshrc` to get a notification once per day:

```zsh
_irfan_daily_tool_check() {
  local stamp="$HOME/.local/state/tool-update-check"
  mkdir -p "$HOME/.local/state"
  if [ ! -f "$stamp" ] || [ "$(date +%j)" != "$(date -r "$stamp" +%j 2>/dev/null)" ]; then
    touch "$stamp"
    update-tools --quiet
  fi
}
_irfan_daily_tool_check
```

### Weekly launchd agent (macOS-native)

```bash
# Enable — runs every Monday at 10 AM
tool-updater-toggle on

# Disable — stops the agent
tool-updater-toggle off
```

The LaunchAgent lives at `~/Library/LaunchAgents/com.irfan.tool-updater.plist` and logs to `~/.local/state/tool-update.log`.

### Alias shortcuts

```zsh
alias ut='update-tools'
alias ut-update='update-tools --update'
```

---

## Adding Your Own Tools

Keeper is designed to be extended. Adding a new tool takes **~15 minutes** and requires writing two small functions.

[**Read the full guide →**](docs/how-to-add-a-tool.md)

Here's the gist:

```zsh
# 1. Check function — compares current vs latest
check_my_tool() {
  local current latest
  current=$(my-tool --version | awk '{print $NF}')
  latest=$(latest_gh_release "owner/repo" | sed 's/^v//')
  [ "$current" = "$latest" ] && { ok "my-tool ${current}"; return 0; } \
                            || { warn "my-tool ${current} → ${latest}"; return 1; }
}

# 2. Update function — downloads and installs the binary
update_my_tool() {
  local ver
  ver=$(latest_gh_release "owner/repo" | sed 's/^v//')
  gh release download "v${ver}" -R "owner/repo" -p "my-tool-darwin-arm64.tar.gz"
  tar xzf "my-tool-darwin-arm64.tar.gz" -C "$TOOLS_DIR"
  ok "my-tool updated to ${ver}"
}

# 3. Register it
TOOLS+=(my_tool)
```

---

## Security

- **System packages are never touched** — macOS system pip packages (`pip`, `setuptools`, `wheel`, `six`, etc.) are detected and skipped automatically
- **Authenticated API calls** — uses your `gh` CLI session. No anonymous API scraping, no tokens in scripts, no rate limits
- **No root required** — everything installs to `~/.local/` under your user
- **Immutable infrastructure pattern** — old versions are cleaned up but the last two are always kept for rollback

---

## Project Structure

```
~/.local/bin/update-tools      # The script (on your PATH)
~/.local/bin/tool-updater-toggle # Enable/disable the LaunchAgent
~/.local/opt/                   # Versioned installations (nvim, node, go, etc.)
~/.local/state/tool-update.log  # LaunchAgent logs
~/.local/share/zsh/site-functions/_gh   # Completions
~/.local/share/zsh/site-functions/_starship
```

---

## Requirements

- **macOS** (ARM64) — the pre-built binary patterns target `aarch64-apple-darwin`
- **zsh** — macOS default shell
- **gh CLI** — Keeper can bootstrap itself: `brew install gh` initially, then Keeper takes over management
- **curl, unzip, tar** — macOS built-ins

---

<p align="center">
  <sub>Built for the homelab developer who wants control without the busywork.</sub><br>
  <sub>MIT License · © 2026 Irfan</sub>
</p>
