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
noto-fonts
ttf-jetbrains-mono-nerd-basic
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

### 5. Fonts

Obtain every OTF variant from:

```text
https://github.com/kingRayhan/operator-mono-lig
```

Install all 12 OTF files under `~/.local/share/fonts/OperatorMono` and rebuild the font cache. Use the upstream `Operator Mono` Book face as the normal monospaced font and configure it for both `monospace` and `ui-monospace` requests. The repository's two `OperatorMonoLig` files are retained alongside the Book, Medium, Bold, Light, and XLight variants.

Set the Foot terminal font size to 11 pt and use the Book face:

```ini
font=Operator Mono:style=Book:size=11
```

Verify precomposed and decomposed French accents such as `é è à ç ô ê ë` in a newly opened Foot terminal.

Keep normal Linux font rendering. Do not reintroduce the reverted macOS-style fontconfig rendering override.

### 6. Configure Caps Lock

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

### 7. Configure terminal shortcuts

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

### 8. Configure browser shortcuts

Override Omarchy's default `SUPER+W` and `SUPER+T` bindings, then add browser shortcuts in `~/.config/hypr/bindings.lua`:

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

o.bind("SUPER + bracketleft", "Previous browser page", previous_browser_page)
o.bind("SUPER + bracketright", "Next browser page", next_browser_page)

hl.unbind("SUPER + R")
hl.unbind("SUPER + SHIFT + R")
o.bind("SUPER + R", "Reload browser page", reload_browser_page, { non_consuming = true })
o.bind("SUPER + SHIFT + R", "Hard reload browser page", hard_reload_browser_page, { non_consuming = true })
```

In a browser, `SUPER+W` sends `Ctrl+W`, so it closes the active tab. When the browser has no tab left, `Ctrl+W` closes the browser window. In other applications, `SUPER+W` keeps its window-close behavior.

In a browser, `SUPER+T` sends `Ctrl+T` to open a new tab. Outside browsers, `SUPER+T` does nothing and cannot change the tile layout.

In a browser, `SUPER+[` sends `Alt+Left` to go to the previous page. `SUPER+]` sends `Alt+Right` to go to the next page. Outside browsers, both shortcuts do nothing.

In a browser, `SUPER+R` sends `Ctrl+R` to reload the current page. `SUPER+SHIFT+R` sends `Ctrl+Shift+R` to perform a hard reload. Outside browsers, `SUPER+R` remains available to Foot for its `Ctrl+L` binding.

After changing the file, run `hyprctl reload` and `hyprctl configerrors`. The error list must be empty.

### 9. Route application links to the active workspace

Brave's native-Wayland external-link path can reuse a browser window on another
workspace without activating the workspace. Omarchy's XDG browser selection is
correct, but the browser hand-off needs to be workspace-aware.

The user-level launcher at `~/.local/bin/brave` handles URL launches as follows:

- If a regular Brave window is already on the active workspace, it focuses that
  exact Hyprland client and lets Brave open the URL there.
- Otherwise, it records the existing Brave clients, creates a blank browser
  window through the existing process (or starts one when no process exists),
  waits for its Hyprland address, moves that address silently to the active
  workspace, focuses it, and enters the URL directly with `Ctrl+L` and
  `wtype`.
- Private-window and ordinary non-URL launches keep the normal Brave behavior.

The full launcher is:

```bash
#!/bin/bash

set -u

real_browser=/usr/bin/brave
has_url=false
requested_url=

for argument in "$@"; do
  case "$argument" in
    http://* | https://*)
      has_url=true
      requested_url=$argument
      break
      ;;
  esac
done

# Keep ordinary browser launches and private-window launches unchanged.
if [[ $has_url != true ]] \
    || ! command -v hyprctl >/dev/null 2>&1 \
    || ! command -v jq >/dev/null 2>&1 \
    || ! command -v wtype >/dev/null 2>&1; then
  exec "$real_browser" "$@"
fi

for argument in "$@"; do
  case "$argument" in
    --incognito | --private | --private-window | --inprivate)
      exec "$real_browser" "$@"
      ;;
  esac
done

# Apps launched through uwsm-app can lack HYPRLAND_INSTANCE_SIGNATURE even
# though they still have a Wayland display. Always pass the instance explicitly
# so workspace queries and dispatches do not depend on inherited shell state.
hyprctl_instance_args=()
if [[ -n ${HYPRLAND_INSTANCE_SIGNATURE:-} ]]; then
  hyprctl_instance_args=(-i "$HYPRLAND_INSTANCE_SIGNATURE")
else
  hyprland_instance=$(
    hyprctl -j instances 2>/dev/null |
      jq -r --arg wayland_display "${WAYLAND_DISPLAY:-}" '
        if $wayland_display != "" then
          (map(select(.wl_socket == $wayland_display)) | .[0].instance) // empty
        else
          (sort_by(.time) | last | .instance) // empty
        end
      ' 2>/dev/null
  )

  # In a UWSM-launched scope, hyprctl may not be able to enumerate instances
  # without a signature. The runtime directory still contains the live
  # instance socket, which is sufficient when this user has one Hyprland
  # session (the normal Omarchy setup).
  if [[ -z $hyprland_instance ]]; then
    hypr_runtime_dir="${XDG_RUNTIME_DIR:-/run/user/$(id -u)}/hypr"
    hyprland_instance=$(
      for instance_dir in "$hypr_runtime_dir"/*; do
        if [[ -S "$instance_dir/.socket.sock" ]]; then
          basename "$instance_dir"
        fi
      done | head -1
    )
  fi

  if [[ -n $hyprland_instance ]]; then
    hyprctl_instance_args=(-i "$hyprland_instance")
  fi
fi

hyprctl_cmd() {
  hyprctl "${hyprctl_instance_args[@]}" "$@"
}

current_workspace=$(hyprctl_cmd activeworkspace -j 2>/dev/null | jq -r '.id // empty' 2>/dev/null)
if [[ ! $current_workspace =~ ^[0-9]+$ ]]; then
  exec "$real_browser" --new-window "$@"
fi

browser_address_on_workspace=$(
  hyprctl_cmd clients -j 2>/dev/null |
    jq -r --arg workspace "$current_workspace" '
      .[]
      | select(.class == "brave-browser")
      | select((.workspace.id | tostring) == $workspace)
      | [.focusHistoryID, .address]
      | @tsv
    ' 2>/dev/null |
    sort -n -k1,1 |
    awk 'NF >= 2 { print $2; exit }'
)

if [[ -n $browser_address_on_workspace ]]; then
  hyprctl_cmd dispatch \
    "hl.dsp.focus({ window = \"address:$browser_address_on_workspace\" })" \
    >/dev/null 2>&1 || true
  exec "$real_browser" "$@"
fi

existing_browser_addresses=$(
  hyprctl_cmd clients -j 2>/dev/null |
    jq -c '[.[] | select(.class == "brave-browser") | .address]' 2>/dev/null
) || existing_browser_addresses='[]'

if [[ -z $existing_browser_addresses || $existing_browser_addresses != \[*\] ]]; then
  existing_browser_addresses='[]'
fi

find_new_browser_window() {
  local existing_addresses=$1
  local candidate=

  for _ in {1..100}; do
    candidate=$(
      hyprctl_cmd clients -j 2>/dev/null |
        jq -r --argjson existing "$existing_addresses" '
          [
            .[]
            | select(.class == "brave-browser")
            | . as $client
            | select(($existing | index($client.address)) == null)
          ]
          | sort_by(.focusHistoryID // -1)
          | last
          | .address // empty
        ' 2>/dev/null
    )

    if [[ -n $candidate ]]; then
      printf '%s\n' "$candidate"
      return 0
    fi

    sleep 0.1
  done

  return 1
}

load_url_into_window() {
  local address=$1
  local focused_address

  hyprctl_cmd dispatch \
    "hl.dsp.window.move({ workspace = \"$current_workspace\", follow = false, window = \"address:$address\" })" \
    >/dev/null 2>&1 || return 1
  hyprctl_cmd dispatch \
    "hl.dsp.focus({ window = \"address:$address\" })" \
    >/dev/null 2>&1 || return 1

  focused_address=$(hyprctl_cmd activewindow -j 2>/dev/null | jq -r '.address // empty' 2>/dev/null)
  [[ $focused_address == "$address" ]] || return 1

  # Brave's native-Wayland external-link path can route the URL to an older
  # window on another workspace. Navigate the exact new window instead of
  # handing the URL back to Brave as a second external-link request.
  hyprctl_cmd dispatch \
    "hl.dsp.send_shortcut({ mods = \"CTRL\", key = \"L\", window = \"address:$address\" })" \
    >/dev/null 2>&1 || return 1
  sleep 0.05
  wtype -- "$requested_url" >/dev/null 2>&1 || return 1
  wtype -k Return >/dev/null 2>&1 || return 1
}

# If another Brave window exists, create a real new window through the most
# recently used one. This avoids Brave turning --new-window URL requests into
# background tabs in that remote process.
global_browser_address=$(
  hyprctl_cmd clients -j 2>/dev/null |
    jq -r '
      .[]
      | select(.class == "brave-browser")
      | [.focusHistoryID, .address]
      | @tsv
    ' 2>/dev/null |
    sort -n -k1,1 |
    awk 'NF >= 2 { print $2; exit }'
)

new_browser_address=
if [[ -n $global_browser_address ]]; then
  hyprctl_cmd dispatch \
    "hl.dsp.send_shortcut({ mods = \"CTRL\", key = \"N\", window = \"address:$global_browser_address\" })" \
    >/dev/null 2>&1 || true
  new_browser_address=$(find_new_browser_window "$existing_browser_addresses") || new_browser_address=
else
  # With no existing browser process, start a blank window on the current
  # workspace and put the URL into it after Hyprland has assigned an address.
  "$real_browser" --new-window >/dev/null 2>&1 &
  new_browser_address=$(find_new_browser_window "$existing_browser_addresses") || new_browser_address=
fi

if [[ -n $new_browser_address ]] && load_url_into_window "$new_browser_address"; then
  exit 0
fi

# Preserve the normal browser behavior if a compositor/window-system detail
# prevented the exact-window hand-off from completing.
exec "$real_browser" --new-window "$@"
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
requires `jq`, `hyprctl`, and `wtype`; Omarchy already provides these on
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

### 10. Keep the pointer in place when new windows open

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
- Operator Mono Book is the Foot font at 11 pt, with all upstream variants installed and correct accent rendering.
- `SUPER+R` acts like `Ctrl+L` in a newly opened Foot terminal.
- `Delete` removes the character after the cursor in a new Foot terminal instead of inserting `~`.
- In a browser, `SUPER+R` reloads the current page and `SUPER+SHIFT+R` performs a hard reload.
- `SUPER+W` closes one browser tab at a time and closes the browser window after the last tab. In other applications, it closes the active window.
- `SUPER+T` opens a new browser tab and does nothing in other applications.
- `SUPER+[` goes to the previous browser page, and `SUPER+]` goes to the next browser page. Both do nothing in other applications.
- Application HTTP and HTTPS links open in a regular Brave window on the active workspace, or create one there when none exists.
- Off-site links clicked in any standalone webapp, including mouse-wheel
  middle-clicks and `CTRL`+clicks, use the same active-workspace routing instead
  of jumping to a Brave window on another workspace.
- Opening a link in a new browser window leaves the pointer at its previous position.
- `hyprctl configerrors` is empty.
- Caps Lock toggles capitalization instead of entering Compose or accent mode.

Finish with a concise report separating completed software configuration from any manual firmware step that remains.
