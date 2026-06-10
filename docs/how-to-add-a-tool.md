# How to Add Any CLI Tool to Keeper

*A practical guide to extending the Keeper update system — with two real-world examples.*

---

If you have a CLI tool you love that isn't in Keeper's default list, adding it takes about **15 minutes** and requires writing exactly **two zsh functions**. This guide shows you exactly how.

## The Pattern

Every tool in Keeper follows this contract:

| Function | Purpose | Return |
|----------|---------|--------|
| `check_<name>()` | Detect current version, fetch latest from upstream | `0` = up to date, `1` = update available |
| `update_<name>()` | Download + install the new version | any non-zero = failure |

Once written, you register the tool name in the `TOOLS` array at the bottom of the script.

---

## Example 1: Adding `bat`

[bat](https://github.com/sharkdp/bat) is a `cat` clone with syntax highlighting. Let's add it step by step.

### Step 1: Find the GitHub repo and release asset name

```bash
# Check what assets the latest release provides
gh release view latest -R sharkdp/bat --json assets --jq '.assets[].name'
```

Output includes `bat-v0.25.0-aarch64-apple-darwin.tar.gz` — that's our target.

### Step 2: Write `check_bat`

```zsh
TOOL_BAT_REPO="sharkdp/bat"

check_bat() {
  local current latest
  # bat --version outputs something like "bat 0.25.0"
  current=$(cmd_exists bat && bat --version | awk '{print $2}' || echo "not-installed")
  # Fetch the latest tag from GitHub, strip the leading 'v'
  latest=$(latest_gh_release "$TOOL_BAT_REPO" | sed 's/^v//')
  [ "$current" = "$latest" ] && { ok "bat     ${current}"; return 0; } \
                            || { warn "bat     ${current} → ${latest}"; return 1; }
}
```

**What's happening here?**
1. `cmd_exists bat` checks if the binary is on `$PATH` — handles the "not installed yet" case gracefully
2. `bat --version` outputs `bat 0.25.0` → `awk '{print $2}'` extracts `0.25.0`
3. `latest_gh_release` uses `gh release view --json tagName --jq .tagName` — returns `v0.25.0`, then `sed 's/^v//'` strips the `v`
4. The one-liner at the end prints a color-coded status line and returns `0` (current) or `1` (stale)

### Step 3: Write `update_bat`

```zsh
update_bat() {
  local ver
  ver=$(latest_gh_release "$TOOL_BAT_REPO" | sed 's/^v//')
  log "Downloading bat ${ver}..."
  local archive="bat-v${ver}-aarch64-apple-darwin.tar.gz"
  gh release download "v${ver}" -R "$TOOL_BAT_REPO" -p "$archive" --clobber
  tar xzf "$archive" -C /tmp
  cp "/tmp/bat-v${ver}-aarch64-apple-darwin/bat" "$TOOLS_DIR/bat"
  chmod +x "$TOOLS_DIR/bat"
  rm -f "$archive"
  rm -rf "/tmp/bat-v${ver}-aarch64-apple-darwin"
  ok "bat     updated to ${ver}"
}
```

**What's happening here?**
1. Gets the latest version tag from GitHub
2. Downloads the release asset (`gh release download`)
3. Extracts the tarball to `/tmp`
4. Copies the single `bat` binary to `~/.local/bin/`
5. Makes it executable
6. Cleans up

### Step 4: Register it

Find the `TOOLS` array near the bottom of `update-tools` and add your snake_case name:

```zsh
TOOLS=(
  nvim
  node
  fzf
  rg
  starship
  zoxide
  opencode
  gh
  go
  homebrew
  npm_global
  pip
  bat              # <-- new
)
```

### Step 5: Test

```bash
update-tools              # should show "▲ bat  (not-installed) → 0.25.0"
update-tools --update bat # downloads and installs
bat --version             # verify
update-tools              # should now show "● bat 0.25.0"
```

---

## Example 2: Adding `fd`

[fd](https://github.com/sharkdp/fd) is a fast `find` alternative. Let's add it — but this time the asset name is slightly different, so we adapt.

### Step 1: Check asset names

```bash
gh release view latest -R sharkdp/fd --json assets --jq '.assets[].name'
```

Output: `fd-v10.2.0-aarch64-apple-darwin.tar.gz`

### Step 2: Write `check_fd`

```zsh
TOOL_FD_REPO="sharkdp/fd"

check_fd() {
  local current latest
  # fd --version outputs "fd 10.2.0"
  current=$(cmd_exists fd && fd --version | awk '{print $2}' || echo "not-installed")
  latest=$(latest_gh_release "$TOOL_FD_REPO" | sed 's/^v//')
  [ "$current" = "$latest" ] && { ok "fd      ${current}"; return 0; } \
                            || { warn "fd      ${current} → ${latest}"; return 1; }
}
```

### Step 3: Write `update_fd`

```zsh
update_fd() {
  local ver
  ver=$(latest_gh_release "$TOOL_FD_REPO" | sed 's/^v//')
  log "Downloading fd ${ver}..."
  local archive="fd-v${ver}-aarch64-apple-darwin.tar.gz"
  gh release download "v${ver}" -R "$TOOL_FD_REPO" -p "$archive" --clobber
  tar xzf "$archive" -C /tmp
  cp "/tmp/fd-v${ver}-aarch64-apple-darwin/fd" "$TOOLS_DIR/fd"
  chmod +x "$TOOLS_DIR/fd"
  rm -f "$archive"
  rm -rf "/tmp/fd-v${ver}-aarch64-apple-darwin"
  ok "fd      updated to ${ver}"
}
```

### Step 4: Register + test

```zsh
TOOLS+=(fd)
```

```bash
update-tools              # check it
update-tools --update fd  # install it
```

---

## Common Asset Patterns

Here's a cheat sheet for finding the right binary asset name for different projects:

| Project | Asset Pattern (aarch64 Apple) |
|---------|------------------------------|
| **starship** | `starship-aarch64-apple-darwin.tar.gz` |
| **ripgrep** | `ripgrep-{ver}-aarch64-apple-darwin.tar.gz` |
| **zoxide** | `zoxide-{ver}-aarch64-apple-darwin.tar.gz` |
| **bat** | `bat-v{ver}-aarch64-apple-darwin.tar.gz` |
| **fd** | `fd-v{ver}-aarch64-apple-darwin.tar.gz` |
| **eza** | `eza_aarch64-apple-darwin.tar.gz` |
| **delta** | `delta-{ver}-aarch64-apple-darwin.tar.gz` |
| **doggo** | `doggo_({ver})_darwin_arm64.tar.gz` |

To find yours:

```bash
gh release view latest -R owner/repo --json assets --jq '.assets[].name' | grep -i "darwin\|apple\|macos"
```

---

## Beyond Binaries: Non-GitHub Sources

Some tools don't distribute through GitHub releases. Keeper handles these too:

### Git-based (like fzf)

```zsh
TOOL_EXAMPLE_REPO="owner/example"

check_example() {
  local current latest
  current=$(cmd_exists example && example --version | awk '{print $2}' || echo "not-installed")
  latest=$(latest_gh_release "$TOOL_EXAMPLE_REPO" | sed 's/^v//')
  [ "$current" = "$latest" ] && { ok "example ${current}"; return 0; } \
                            || { warn "example ${current} → ${latest}"; return 1; }
}

update_example() {
  local tag
  tag=$(latest_gh_release "$TOOL_EXAMPLE_REPO")
  (cd "$HOME/.local/src/example" \
    && git fetch --tags origin \
    && git checkout -f "$tag" \
    && make \
    && cp bin/example "$TOOLS_DIR/example")
}
```

### Tarball from CDN (like Node.js)

```zsh
check_cdn_tool() {
  # Use curl to fetch version from an API endpoint
  local current latest
  current=$(cdn-tool --version)
  latest=$(curl -fsL "https://cdn.example.com/latest-version" | tr -d ' \n')
  # ...compare and report
}
```

---

## Best Practices

1. **Handle the "not installed" case** — use `cmd_exists binary_name || echo "not-installed"` so the check doesn't crash on missing tools
2. **Clean up after yourself** — always `rm -f "$archive"` and clean temp dirs in `update_*` functions
3. **Keep old versions** — Keeper's Node.js updater keeps the last 2 versions for rollback; follow the same pattern
4. **Use `--clobber`** with `gh release download` to avoid hanging on retries
5. **Test the update path** — run `update-tools --update your-tool` to verify the full flow
6. **Match the display width** — Keeper pads tool names to 8 characters for alignment (e.g., `"bat     "`, `"fd      "`)

---

## The Full Template

Here's a copy-paste template to get started:

```zsh
# ── <Your tool> ────────────────────────────────────────────────
TOOL_MYTOOL_REPO="owner/repo"

check_mytool() {
  local current latest
  current=$(cmd_exists mytool && mytool --version | awk '{print $2}' || echo "not-installed")
  latest=$(latest_gh_release "$TOOL_MYTOOL_REPO" | sed 's/^v//')
  [ "$current" = "$latest" ] && { ok "mytool  ${current}"; return 0; } \
                            || { warn "mytool  ${current} → ${latest}"; return 1; }
}

update_mytool() {
  local ver
  ver=$(latest_gh_release "$TOOL_MYTOOL_REPO" | sed 's/^v//')
  log "Downloading mytool ${ver}..."
  local archive="mytool-${ver}-aarch64-apple-darwin.tar.gz"
  gh release download "v${ver}" -R "$TOOL_MYTOOL_REPO" -p "$archive" --clobber
  tar xzf "$archive" -C /tmp
  cp "/tmp/mytool-${ver}-aarch64-apple-darwin/bin/mytool" "$TOOLS_DIR/mytool"
  chmod +x "$TOOLS_DIR/mytool"
  rm -f "$archive"
  rm -rf "/tmp/mytool-${ver}-aarch64-apple-darwin"
  ok "mytool  updated to ${ver}"
}

# Then add "mytool" to the TOOLS array
```

---

*Have questions or want to contribute? Open an issue or PR on the [keeper repo](https://github.com/irfancode/keeper).*
