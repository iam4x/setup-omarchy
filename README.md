# Omarchy workstation/server setup prompt

This document is a reusable prompt for reproducing this machine's Omarchy setup on another computer. Give the section below to a capable system-administration agent running on the target machine.

The prompt describes the final configuration. It intentionally excludes experiments that were later reverted and requests that were interrupted before being applied.

## Reusable prompt

You are configuring a fresh Omarchy 4.x / Arch Linux computer as both a development workstation and an always-on server. Complete the work end to end, inspect the target machine before changing it, preserve unrelated user configuration, and verify every change.

### Working rules

- Determine the current username with `id -un`; refer to it as `TARGET_USER`. Do not hardcode `iam4x`.
- Back up every existing user or system configuration file before changing it.
- Use current Omarchy and Hyprland configuration syntax from the installed version. Omarchy uses Lua-based Hyprland configuration on current releases.
- Put user overrides under `~/.config`; never edit `/usr/share/omarchy`.
- Prefer `omarchy pkg add` for official Arch packages.
- Use graphical privilege authorization when the execution environment cannot safely accept a terminal password.
- Never expose SSH directly to the public internet. It is intended to be reached through Tailscale.
- After changing Hyprland Lua files, run `hyprctl reload` and `hyprctl configerrors`, fixing every error.
- Do not blindly copy hardware identifiers from this document. Discover monitor connectors and web-app window classes on the target computer.
- Report exactly what changed, all verification results, and anything requiring manual firmware interaction.

### 1. Install command-line and development packages

Install these packages from the configured Arch repositories:

```text
openssh
tailscale
neovim
git
ripgrep
gcc
base-devel
curl
unzip
tree-sitter-cli
zsh
zsh-antidote
zoxide
fnm
stylua
taplo-cli
lazygit
tig
cloudflared
wl-clipboard
```

Notes:

- The Arch package is `taplo-cli`, while the executable is `taplo`.
- Only install `cloudflared`; do not create a tunnel or service without separate configuration details.
- Enable and connect Tailscale if it is not already connected. Authentication may require the user.
- Verify every requested executable with `command -v` and its version command.

### 2. Configure Zsh, Antidote, zoxide, and fnm

Make `/usr/bin/zsh` the login shell for `TARGET_USER` and configure Foot to launch Zsh. A newly opened terminal must actually start Zsh.

Configure persistent Zsh history so autosuggestions work across sessions:

```zsh
HISTFILE="${XDG_STATE_HOME:-$HOME/.local/state}/zsh/history"
HISTSIZE=100000
SAVEHIST=100000
[[ -d ${HISTFILE:h} ]] || mkdir -p -- "${HISTFILE:h}"
setopt append_history share_history hist_ignore_all_dups hist_ignore_space
```

Load Antidote from `/usr/share/zsh-antidote/antidote.zsh` and load `~/.zsh_plugins.txt`. Include these plugins:

```text
getantidote/use-omz
ohmyzsh/ohmyzsh path:lib/git.zsh
ohmyzsh/ohmyzsh path:lib/directories.zsh
ohmyzsh/ohmyzsh path:plugins/git
ohmyzsh/ohmyzsh path:plugins/bun
ohmyzsh/ohmyzsh path:plugins/common-aliases
ohmyzsh/ohmyzsh path:plugins/sudo
ohmyzsh/ohmyzsh path:plugins/copyfile
ohmyzsh/ohmyzsh path:plugins/copypath
ohmyzsh/ohmyzsh path:plugins/docker
ohmyzsh/ohmyzsh path:plugins/docker-compose
ohmyzsh/ohmyzsh path:plugins/uv
zsh-users/zsh-syntax-highlighting
zsh-users/zsh-autosuggestions
zsh-users/zsh-history-substring-search
djui/alias-tips
aloxaf/fzf-tab
sindresorhus/pure
```

Initialize the navigation and Node tools:

```zsh
eval "$(zoxide init --cmd j zsh)"
eval "$(fnm env --use-on-cd --shell zsh)"
```

Enable case-insensitive substring completion and add these aliases/environment settings:

```zsh
zstyle ':completion:*' matcher-list 'm:{a-zA-Z}={A-Za-z}' 'r:|[._-]=* r:|=*' 'l:|=* r:|=*'

alias nvm="fnm"
alias ls="ls --color=auto"
alias vi="nvim"
alias vim="nvim"
alias n="nvim"
alias pbcopy="wl-copy --trim-newline"
alias pbpaste="wl-paste"

export EDITOR="nvim"
```

`pbcopy` must trim the single trailing newline produced by commands such as `pwd`, so `pwd | pbcopy` copies only the path. Verify aliases in a fresh interactive Zsh session. Confirm `zsh-users/zsh-autosuggestions` is loaded and suggesting entries from persistent history.

Install and select the current Node.js LTS with fnm, then install Prettier globally

```text
fnm install --lts --use
fnm default <the installed LTS version>
npm install --global prettier
```

### 3. Install the Neovim configuration

Use the configuration repository at:

```text
https://github.com/iam4x/nvim.git
```

Install it at `~/.config/nvim`. If that path already contains user, overwrite it without explicit confirmation.

The configuration must retain these behaviors:

- Lazy.nvim plugin management.
- `blink.cmp` pinned to stable `version = "1.*"` so its prebuilt fuzzy library is available and insert mode does not report a missing fuzzy matcher.
- Treesitter parsers for Git commit messages and common Git, shell, Lua, web, Markdown, TOML, JSON, and YAML files.
- Correct `gitcommit` highlighting, spelling, wrapping, a 72-column text width, and a guide at column 73.
- Mason/LSP setup for `bashls`, `jsonls`, `lua_ls`, `taplo`, `ts_ls`, and `yamlls`.
- Formatting through Stylua, Taplo, Prettier or Prettierd.
- Lazygit integration.
- No swap files and no backup files:

```lua
vim.opt.swapfile = false
vim.opt.backup = false
vim.opt.writebackup = false
```

Install plugins and parsers with:

```text
nvim --headless "+Lazy! sync" +qa
```

If necessary, repair the Git commit parser with:

```text
nvim --headless "+lua require('nvim-treesitter').install({'gitcommit'}):wait(300000)" +qa
```

Verify `:checkhealth`, Lazy, Mason, LSP attachment, Treesitter highlighting, formatting, Git commit buffers, and that editing/writing files creates no `.bak` files.

### 4. Configure OpenSSH for Tailscale access

Enable and start `tailscaled` and `sshd`. Preserve the target user's existing password and allow username/password login for `TARGET_USER`, with PAM enabled and root login disabled.

Create a dedicated SSH drop-in equivalent to:

```text
PasswordAuthentication yes
UsePAM yes
PermitRootLogin no
AllowUsers TARGET_USER
```

Validate with `sshd -t` before restarting or reloading the service. Ensure TCP port 22 is reachable through `tailscale0` but is not exposed through the public WAN. Respect the firewall already used by the target system rather than replacing its complete policy.

Verify:

- `tailscaled` and `sshd` are enabled and active.
- The machine has a Tailscale address.
- `sshd -T` reports password authentication enabled, root login disabled, and the expected allowed user.
- An SSH login through the Tailscale address works with the username and password.

### 5. Configure Caps Lock

Omarchy's default Hyprland input configuration maps Caps Lock to the Compose key with `compose:caps`. That makes Caps Lock enter accent mode instead of toggling capitalization.

Override `kb_options` in `~/.config/hypr/input.lua` and omit `compose:caps`:

```lua
hl.config({
  input = {
    kb_options = "shift:both_capslock_cancel,altwin:swap_alt_win",
  },
})
```

Keep any other keyboard options that the target machine needs, but do not add `compose:caps`.

Validate and apply the change:

```bash
luac -p ~/.config/hypr/input.lua
hyprctl reload
hyprctl configerrors
```

The error list must be empty. Caps Lock must toggle capitalization instead of entering Compose or accent mode.

### 6. Configure terminal shortcuts

In the `[text-bindings]` section of `~/.config/foot/foot.ini`, bind `SUPER+R` to emit `Ctrl+L`:

```ini
\x0c=Mod4+r
```

Keep the browser reload binding non-consuming so Foot can still receive `SUPER+R` outside browser windows. Validate the file with `foot --check-config`.

In `~/.zshrc`, bind Foot's forward-delete sequence in Zsh's insert and Emacs keymaps:

```zsh
bindkey -M viins '^[[3~' delete-char
bindkey -M emacs '^[[3~' delete-char
```

This makes `Delete` remove the character after the cursor instead of inserting `~`. Open a new Foot terminal after changing the file, or run `source ~/.zshrc` in an existing shell. Verify the Zsh binding with:

```zsh
zsh -lic 'bindkey -M viins "^[[3~"'
```

The command must report `delete-char`.

### 7. Configure browser and scratchpad shortcuts

Override Omarchy's default `SUPER+W`, `SUPER+T`, and `SUPER+S` bindings, then add the shortcuts in `~/.config/hypr/bindings.lua`:

```lua
local function active_window_is_browser()
  local window = hl.get_active_window()
  if not window then
    return false
  end

  local class = (window.class or ""):lower()
  return class:match("^brave")
    or class:match("^firefox")
    or class:match("^chromium")
    or class:match("^google%-chrome")
    or class:match("^microsoft%-edge")
    or class:match("^vivaldi")
    or class:match("^opera")
    or class:match("^librewolf")
    or class:match("^zen")
    or class:match("^helium")
end

local function send_shortcut_once(mods, key)
  hl.dispatch(hl.dsp.send_key_state({ mods = mods, key = key, state = "down" }))
  hl.timer(function()
    hl.dispatch(hl.dsp.send_key_state({ mods = mods, key = key, state = "up" }))
  end, { timeout = 50, type = "oneshot" })
end

local function close_browser_tab_or_window()
  if not active_window_is_browser() then
    hl.dispatch(hl.dsp.window.close())
    return
  end

  send_shortcut_once("CTRL", "W")
end

local function open_browser_tab()
  if active_window_is_browser() then
    send_shortcut_once("CTRL", "T")
  end
end

local function previous_browser_page()
  if active_window_is_browser() then
    send_shortcut_once("ALT", "LEFT")
  end
end

local function next_browser_page()
  if active_window_is_browser() then
    send_shortcut_once("ALT", "RIGHT")
  end
end

local function reload_browser_page()
  if active_window_is_browser() then
    send_shortcut_once("CTRL", "R")
  end
end

local function hard_reload_browser_page()
  if active_window_is_browser() then
    send_shortcut_once("CTRL + SHIFT", "R")
  end
end

hl.unbind("SUPER + W")
o.bind("SUPER + W", "Close browser tab / window", close_browser_tab_or_window)

hl.unbind("SUPER + T")
o.bind("SUPER + T", "New browser tab", open_browser_tab)

-- Replace Omarchy's default SUPER+S scratchpad binding with Ctrl+S.
hl.unbind("SUPER + S")
o.bind("SUPER + S", "Save", function()
  send_shortcut_once("CTRL", "S")
end)

-- Disable Omarchy's workspace layout toggle, including scrolling mode.
hl.unbind("SUPER + L")

-- Remove Omarchy's preinstalled Google Maps shortcut.
hl.unbind("SUPER + SHIFT + S")

-- Remove Omarchy's preinstalled Google Photos and Spotify shortcuts.
hl.unbind("SUPER + SHIFT + P")
hl.unbind("SUPER + SHIFT + M")

o.bind("SUPER + bracketleft", "Previous browser page", previous_browser_page)
o.bind("SUPER + bracketright", "Next browser page", next_browser_page)

hl.unbind("SUPER + R")
hl.unbind("SUPER + SHIFT + R")
o.bind("SUPER + R", "Reload browser page", reload_browser_page, { non_consuming = true })
o.bind("SUPER + SHIFT + R", "Hard reload browser page", hard_reload_browser_page, { non_consuming = true })

-- SUPER+~ uses the direct GRAVE key on the configured US keyboard layout.
o.bind("SUPER + GRAVE", "Toggle scratchpad", hl.dsp.workspace.toggle_special("scratchpad"))
o.bind("SUPER + SHIFT + GRAVE", "Move window to scratchpad", hl.dsp.window.move({ workspace = "special:scratchpad", follow = false }))
```

In a browser, `SUPER+W` sends `Ctrl+W`, so it closes the active tab. When the browser has no tab left, `Ctrl+W` closes the browser window. In other applications, `SUPER+W` keeps its window-close behavior.

In a browser, `SUPER+T` sends `Ctrl+T` to open a new tab. Outside browsers, `SUPER+T` does nothing and cannot change the tile layout.

`SUPER+L` remains unbound, so it cannot toggle the workspace layout or enter scrolling workspace mode.

In a browser, `SUPER+[` sends `Alt+Left` to go to the previous page. `SUPER+]` sends `Alt+Right` to go to the next page. Outside browsers, both shortcuts do nothing.

In a browser, `SUPER+R` sends `Ctrl+R` to reload the current page. `SUPER+SHIFT+R` sends `Ctrl+Shift+R` to perform a hard reload. Outside browsers, `SUPER+R` remains available to Foot for its `Ctrl+L` binding.

`SUPER+GRAVE` toggles the scratchpad. On a US keyboard, `GRAVE` is the key
labeled `~`; press it without `SHIFT`. `SUPER+SHIFT+GRAVE` moves the focused
window to the scratchpad. `SUPER+S` sends `Ctrl+S` to the focused application.
`SUPER+SHIFT+S` must remain unbound and must not open Google Maps.
`SUPER+SHIFT+P` and `SUPER+SHIFT+M` must remain unbound and must not open
Google Photos or Spotify.

Fcitx5 Quick Phrase also uses `Super+grave` by default. To leave that key for
Hyprland, back up `~/.config/fcitx5/conf/quickphrase.conf` if it exists, then
create it with this content:

```ini
[TriggerKey]
0=Super+semicolon
```

This removes `Super+grave` from Quick Phrase and keeps `Super+semicolon` as its
remaining trigger. Restart Fcitx5 after changing the file.

After changing the file, run `hyprctl reload` and `hyprctl configerrors`. The error list must be empty.

### 8. Route application links to the active workspace

Omarchy routes app links through `omarchy-launch-browser`, which starts the
browser in the background and then focuses the most recently used Brave window
on any workspace. That races with workspace-aware routing and can pull you to
another workspace, so links go straight to a user-level launcher instead.

The launcher at `~/.local/bin/brave` handles URL launches as follows:

- It reads the target workspace from the focused monitor, preferring an open
  special workspace, so the scratchpad works (`hyprctl activeworkspace` ignores
  special workspaces).
- If a regular Brave window is on that workspace, it focuses that exact
  Hyprland client, waits for the focus to land, and hands the URL to Brave,
  which opens it as a tab there.
- Otherwise it opens the URL with `--new-window` and moves the new Hyprland
  client to the target workspace.
- The URL is only ever passed as an argument, never typed with `wtype`, so a
  focus change cannot leak it into another app. Launches are serialized with
  `flock` so concurrent clicks cannot claim each other's window.
- Private-window and ordinary non-URL launches keep the normal Brave behavior.

The full launcher is:

```bash
#!/bin/bash

# Workspace-aware Brave launcher for links opened from other apps.
#
# - A Brave window on the current workspace (scratchpad included): focus it and
#   hand the URL to Brave, which opens it as a tab in the last active window.
# - No Brave window there: open the URL with --new-window, then move that new
#   window to the current workspace.
#
# The URL is always passed to Brave as an argument, never typed with keystrokes,
# so a focus change can never send it to another app.

set -u

real_browser=/usr/bin/brave
handoff=${OMARCHY_PATH:-/usr/share/omarchy}/bin/omarchy-cmd-browser-handoff

url=
for argument in "$@"; do
  case $argument in
    --incognito | --private | --private-window | --inprivate) exec "$real_browser" "$@" ;;
    http://* | https://*) url=$argument ;;
  esac
done

if [[ -z $url ]] || ! command -v hyprctl >/dev/null || ! command -v jq >/dev/null; then
  exec "$real_browser" "$@"
fi

# Processes started outside the Hyprland session (systemd units, Brave's native
# host) may not inherit the instance signature.
if [[ -z ${HYPRLAND_INSTANCE_SIGNATURE:-} ]]; then
  for socket in "${XDG_RUNTIME_DIR:-/run/user/$(id -u)}"/hypr/*/.socket.sock; do
    [[ -S $socket ]] || continue
    HYPRLAND_INSTANCE_SIGNATURE=$(basename "$(dirname "$socket")")
    export HYPRLAND_INSTANCE_SIGNATURE
    break
  done
fi

dispatch() {
  hyprctl dispatch "$1" >/dev/null 2>&1
}

focused_address() {
  hyprctl activewindow -j 2>/dev/null | jq -r '.address // empty'
}

browser_windows() {
  # Regular Brave windows only; webapps have their own brave-<app> class.
  hyprctl clients -j 2>/dev/null | jq -c '[.[] | select(.class == "brave-browser")]'
}

# Send the command line to the running Brave over its singleton socket, or
# start Brave when none is running. The started browser must not inherit the
# lock below.
open_in_brave() {
  "$handoff" "$real_browser" "$@" 2>/dev/null && return
  "$real_browser" "$@" >/dev/null 2>&1 9>&- &
}

# One link at a time, so concurrent launches cannot claim each other's window.
exec 9>"${XDG_RUNTIME_DIR:-/tmp}/brave-workspace-launcher.lock"
flock -w 10 9

# activeworkspace ignores an open special workspace, so read the focused
# monitor instead and prefer its special workspace when one is shown.
target=$(
  hyprctl monitors -j 2>/dev/null | jq -c '
    first(.[] | select(.focused))
    | if (.specialWorkspace.id // 0) != 0 then .specialWorkspace else .activeWorkspace end
    | { name, selector: (if .id > 0 then (.id | tostring) elif (.name | startswith("special:")) then .name else "name:" + .name end) }
  '
)
target_name=$(jq -r '.name // empty' <<<"$target" 2>/dev/null)
target_selector=$(jq -r '.selector // empty' <<<"$target" 2>/dev/null)

if [[ -z $target_name ]]; then
  open_in_brave "$@"
  exit 0
fi

windows=$(browser_windows)

existing_address=$(
  jq -r --arg ws "$target_name" '
    [.[] | select(.workspace.name == $ws)] | sort_by(.focusHistoryID) | first.address // empty
  ' <<<"$windows"
)

if [[ -n $existing_address ]]; then
  dispatch "hl.dsp.focus({ window = \"address:$existing_address\" })"

  # Brave opens external URLs in the window it last saw activated, so wait
  # until the compositor has focused ours and give Brave a moment to notice.
  for _ in {1..20}; do
    [[ $(focused_address) == "$existing_address" ]] && break
    sleep 0.025
  done
  sleep 0.1

  open_in_brave "$url"
  exit 0
fi

had_browser=$(jq 'length > 0' <<<"$windows")
known_addresses=$(jq -c 'map(.address)' <<<"$windows")

open_in_brave --new-window "$url"

# A cold start opens on the active workspace by itself, and may restore
# several session windows that should stay where they are.
[[ $had_browser == true ]] || exit 0

new_address=
for _ in {1..100}; do
  new_address=$(
    browser_windows | jq -r --argjson known "$known_addresses" '
      [.[] | select(.address as $a | $known | index($a) | not)] | first.address // empty
    '
  )
  [[ -n $new_address ]] && break
  sleep 0.05
done

[[ -n $new_address ]] || exit 0

dispatch "hl.dsp.window.move({ workspace = \"$target_selector\", follow = false, window = \"address:$new_address\" })"
dispatch "hl.dsp.focus({ window = \"address:$new_address\" })"
```

Make it executable:

```bash
chmod +x ~/.local/bin/brave
```

Create `~/.local/share/applications/brave-browser.desktop` so the user-level
desktop entry invokes the launcher. Replace `TARGET_USER` with the actual
username in the absolute `Exec` paths:

```ini
[Desktop Entry]
Version=1.0
Name=Brave
GenericName=Web Browser
Comment=Access the Internet
Exec=/home/TARGET_USER/.local/bin/brave %U
Icon=brave-desktop
Terminal=false
Type=Application
Categories=Network;WebBrowser;
MimeType=text/html;x-scheme-handler/http;x-scheme-handler/https;
StartupNotify=true
StartupWMClass=brave-browser
Actions=new-window;new-private-window;

[Desktop Action new-window]
Name=New Window
Exec=/home/TARGET_USER/.local/bin/brave

[Desktop Action new-private-window]
Name=New Private Window
Exec=/home/TARGET_USER/.local/bin/brave --incognito
```

Set the default handlers in `~/.config/mimeapps.list`:

```ini
[Default Applications]
text/html=brave-browser.desktop
x-scheme-handler/http=brave-browser.desktop
x-scheme-handler/https=brave-browser.desktop
```

Omarchy also ships `~/.config/hyprland-mimeapps.list`, which is read before
`mimeapps.list` on Hyprland and points links at `omarchy-browser.desktop`.
Point it at the launcher instead:

```ini
[Default Applications]
x-scheme-handler/http=brave-browser.desktop
x-scheme-handler/https=brave-browser.desktop
```

Refresh and verify the desktop integration:

```bash
update-desktop-database ~/.local/share/applications
desktop-file-validate ~/.local/share/applications/brave-browser.desktop
bash -n ~/.local/bin/brave
xdg-mime query default x-scheme-handler/https
```

The last command must report `brave-browser.desktop`. Test both paths: open a
link while a regular Brave window is already on the active workspace, then
open a link from a workspace without a regular Brave window. The launcher
requires `jq`, `hyprctl`, `flock`, and `socat`; Omarchy already provides these on
this setup.

Chromium web apps are a separate path. An app launched with `--app=...`
shares Brave's profile and browser process, so an external link clicked inside
the app can bypass the XDG handler and go straight to the most recently used
regular browser window. Standalone webapps are handled with a small unpacked
extension and a native-messaging host:

- `~/.config/brave-extensions/x-external-links/` matches HTTP(S) pages but
  installs its click handler only in standalone webapp windows. Same-origin
  navigation stays inside the webapp; cross-origin HTTP(S) links go to the
  extension service worker. It handles both ordinary primary clicks and
  mouse-wheel middle clicks and `CTRL`+clicks, including links without
  `target="_blank"`.
- `~/.config/BraveSoftware/Brave-Browser/NativeMessagingHosts/com.omarchy.x_external_links.json`
  registers the native host for the unpacked webapp extension ID.
- `~/.local/bin/omarchy-x-link-native-host` validates the URL and invokes
  `~/.local/bin/brave`, so the same active-workspace routing is used.
- The extension directory must be included in
  `~/.config/brave-flags.conf` with `--load-extension=...`.

The current unpacked extension ID is
`cekkfhenfgpfjdeahnphhkbiblkfljgc`; if the extension path changes, update the
`allowed_origins` value in the native-messaging manifest. Fully exit Brave
and relaunch it after changing the load-extension flag. Test with any webapp
on a workspace that has no regular Brave window while a regular Brave window
is open on another workspace: clicking an off-site link must create the new
Brave client on the webapp's workspace and leave that workspace active. Normal
browser tabs are not affected.

### 9. Keep the pointer in place when new windows open

Add this user override to `~/.config/hypr/looknfeel.lua`:

```lua
hl.config({
  cursor = {
    no_warps = true,
  },
})
```

This keeps the pointer in place when Hyprland focuses a newly opened window,
such as a browser window opened from a link. The new window still receives
focus.

After changing the file, run `luac -p ~/.config/hypr/looknfeel.lua`, then run
`hyprctl reload` and `hyprctl configerrors`. The error list must be empty.

### 10. Configure the topbar clock

Set the Omarchy topbar clock to show the day, abbreviated month, four-digit
year, and time:

```bash
omarchy bar set omarchy.clock format 'd MMM yyyy HH:mm'
```

The clock should display values such as `12 Sept 2026 11:58`.

### 11. Final verification

Before declaring completion, verify all of the following:

- The login shell and new Foot terminals use Zsh.
- Antidote plugins load, including autosuggestions.
- `j` invokes zoxide and `nvm` invokes fnm.
- `ls` automatically uses color when writing to a terminal.
- `vi`, `vim`, and `n` invoke Neovim.
- `pwd | pbcopy` pastes without a trailing newline; `pbpaste` reads the Wayland clipboard.
- Neovim starts without Blink fuzzy-library errors, highlights Git commit messages, and creates no `.bak` files.
- `stylua`, `taplo`, `lazygit`, `tig`, `cloudflared`, `fnm`, and `zoxide` are installed.
- SSH password authentication works only through the intended Tailscale path, root SSH login is disabled, and both services start at boot.
- `SUPER+R` acts like `Ctrl+L` in a newly opened Foot terminal.
- `Delete` removes the character after the cursor in a new Foot terminal instead of inserting `~`.
- `SUPER+S` sends `Ctrl+S` to the focused application.
- In a browser, `SUPER+R` reloads the current page and `SUPER+SHIFT+R` performs a hard reload.
- `SUPER+W` closes one browser tab at a time and closes the browser window after the last tab. In other applications, it closes the active window.
- `SUPER+T` opens a new browser tab and does nothing in other applications.
- `SUPER+L` is unbound and does not toggle the workspace layout or scrolling workspace mode.
- `SUPER+[` goes to the previous browser page, and `SUPER+]` goes to the next browser page. Both do nothing in other applications.
- `SUPER+GRAVE` toggles the scratchpad, and `SUPER+SHIFT+GRAVE` moves the focused window to it.
- `SUPER+SHIFT+S` is unbound and does not open Google Maps.
- `SUPER+SHIFT+P` is unbound and does not open Google Photos.
- `SUPER+SHIFT+M` is unbound and does not open Spotify.
- The topbar clock shows the day, abbreviated month, year, and time.
- Fcitx5 Quick Phrase does not use `Super+grave`; `Super+semicolon` remains its trigger.
- Application HTTP and HTTPS links open in a regular Brave window on the active workspace, or create one there when none exists.
- Off-site links clicked in any standalone webapp, including mouse-wheel
  middle-clicks and `CTRL`+clicks, use the same active-workspace routing instead
  of jumping to a Brave window on another workspace.
- Opening a link in a new browser window leaves the pointer at its previous position.
- `hyprctl configerrors` is empty.
- Caps Lock toggles capitalization instead of entering Compose or accent mode.

Finish with a concise report separating completed software configuration from any manual firmware step that remains.
