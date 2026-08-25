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

Obtain the four OTF fonts from:

```text
https://github.com/willfore/vscode_operator_mono_lig/tree/master/fonts
```

Install them under `~/.local/share/fonts/OperatorMonoLig` and rebuild the font cache. Use the Book face as the normal monospaced font and configure `Operator Mono Lig` as the Omarchy monospace family, including `monospace` and `ui-monospace` fontconfig aliases.

Set terminal font size to 10 pt in Foot and any installed Omarchy terminal configurations. For Foot, use this ordered font list:

```ini
font=Operator Mono Lig:size=10,JetBrainsMono Nerd Font:size=10
```

Operator Mono Lig lacks some combining accent glyphs, so JetBrains Mono Nerd Font must be the explicit monospaced fallback. Verify precomposed and decomposed French accents such as `é è à ç ô ê ë` in a newly opened Foot terminal.

Keep normal Linux font rendering. Do not reintroduce the reverted macOS-style fontconfig rendering override.

### 6. Configure Foot terminal shortcuts

In the `[text-bindings]` section of `~/.config/foot/foot.ini`, bind `SUPER+R` to emit `Ctrl+L`:

```ini
\x0c=Mod4+r
```

Before relying on the binding, confirm `SUPER+R` is not consumed by a Hyprland global binding. Validate the file with `foot --check-config`. Foot does not dynamically reload font and key configuration, so test in a newly opened terminal.

### 7. Configure browser tab shortcuts

Override Omarchy's default `SUPER+W` and `SUPER+T` bindings, then add browser navigation bindings in `~/.config/hypr/bindings.lua`:

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

hl.unbind("SUPER + W")
o.bind("SUPER + W", "Close browser tab / window", close_browser_tab_or_window)

hl.unbind("SUPER + T")
o.bind("SUPER + T", "New browser tab", open_browser_tab)

o.bind("SUPER + bracketleft", "Previous browser page", previous_browser_page)
o.bind("SUPER + bracketright", "Next browser page", next_browser_page)
```

In a browser, `SUPER+W` sends `Ctrl+W`, so it closes the active tab. When the browser has no tab left, `Ctrl+W` closes the browser window. In other applications, `SUPER+W` keeps its window-close behavior.

In a browser, `SUPER+T` sends `Ctrl+T` to open a new tab. Outside browsers, `SUPER+T` does nothing and cannot change the tile layout.

In a browser, `SUPER+[` sends `Alt+Left` to go to the previous page. `SUPER+]` sends `Alt+Right` to go to the next page. Outside browsers, both shortcuts do nothing.

After changing the file, run `hyprctl reload` and `hyprctl configerrors`. The error list must be empty.

### 8. Keep the pointer in place when new windows open

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

### 9. Final verification

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
- Operator Mono Lig Book is the default monospace face at 10 pt, with correct accent fallback.
- `SUPER+R` acts like `Ctrl+L` in a newly opened Foot terminal.
- `SUPER+W` closes one browser tab at a time and closes the browser window after the last tab. In other applications, it closes the active window.
- `SUPER+T` opens a new browser tab and does nothing in other applications.
- `SUPER+[` goes to the previous browser page, and `SUPER+]` goes to the next browser page. Both do nothing in other applications.
- Opening a link in a new browser window leaves the pointer at its previous position.
- `hyprctl configerrors` is empty.

Finish with a concise report separating completed software configuration from any manual firmware step that remains.
