# Remote-control Omarchy 4 from macOS Screen Sharing over Tailscale

This guide configures an Omarchy 4 / Hyprland desktop for remote control from
macOS's built-in **Screen Sharing** app using WayVNC. The server listens only on
its private Tailscale address, requires a VNC password, starts with Hyprland,
and translates macOS Command/Option modifiers for Omarchy shortcuts:

- macOS `CMD` triggers Omarchy `SUPER` shortcuts.
- macOS `OPTION` triggers Omarchy `ALT` shortcuts.
- Physical keyboards on the Omarchy machine are unaffected.

Tested with Omarchy 4, Hyprland's Lua configuration, and WayVNC 0.10.1.

## Important security notes

macOS Screen Sharing requires WayVNC's legacy DES authentication mode. That
mode does **not** encrypt the VNC session and uses only the first eight
characters of the password. This guide therefore:

- Binds WayVNC directly to the machine's Tailscale IPv4 address, not
  `0.0.0.0`.
- Relies on Tailscale's WireGuard-based encrypted tunnel for transport
  security.
- Uses an eight-character random password and protects the plaintext config
  with mode `600`.

Both computers must be connected to the same tailnet, and the tailnet policy
must permit TCP port `5900` between them. Do not expose this configuration
directly to the public internet or an untrusted LAN.

## 1. Install WayVNC

On the Omarchy computer:

```bash
omarchy pkg add wayvnc
```

Confirm the installation:

```bash
wayvnc --version
```

WayVNC 0.10 or newer is required for the legacy authentication used by macOS
Screen Sharing.

## 2. Find the server's Tailscale address

```bash
tailscale ip -4
```

Save the returned `100.x.y.z` address. The examples below use
`<TAILSCALE_IPV4>` as a placeholder.

## 3. Configure WayVNC

Create the configuration directory:

```bash
mkdir -p ~/.config/wayvnc
chmod 700 ~/.config/wayvnc
```

Generate an eight-character password:

```bash
openssl rand -hex 4
```

Create `~/.config/wayvnc/config` with the following content, replacing both
placeholders:

```ini
address=<TAILSCALE_IPV4>
port=5900
enable_auth=true
password=<8_CHARACTER_PASSWORD>
relax_encryption=true
allow_broken_crypto=true
```

Protect the file because the password is stored as plaintext:

```bash
chmod 600 ~/.config/wayvnc/config
```

Warnings about `allow_broken_crypto` and relaxed encryption are expected. The
legacy settings are necessary for Apple's VNC client; Tailscale protects the
network path.

## 4. Start WayVNC with Hyprland

Add this to `~/.config/hypr/autostart.lua`:

```lua
-- Share this Hyprland session over the private Tailscale interface.
o.launch_on_start("wayvnc")
```

This starts WayVNC at the beginning of future Hyprland sessions. To start it
immediately, run this from a terminal inside the active Omarchy desktop:

```bash
hyprctl eval 'hl.dispatch(hl.dsp.exec_cmd("wayvnc"))'
```

If an instance is already running and needs restarting:

```bash
wayvncctl wayvnc-exit
hyprctl eval 'hl.dispatch(hl.dsp.exec_cmd("wayvnc"))'
```

WayVNC shares an existing graphical session. It does not provide a display
manager or pre-login screen, so the Hyprland session must already be running.

## 5. Translate Mac Command to Omarchy Super shortcuts

Apple's legacy VNC keyboard events arrive with Command represented as Alt and
Option represented as Super. A normal XKB per-device override is not reliable
for WayVNC's virtual-keyboard protocol. Instead, mirror Hyprland's binding
registry with the modifiers swapped only for the WayVNC device.

Open `~/.config/hypr/hyprland.lua`. Find Omarchy's bootstrap call near the top:

```lua
dofile((os.getenv("OMARCHY_PATH") or "/usr/share/omarchy") .. "/default/hypr/bootstrap.lua")
```

Paste the following block **immediately after the bootstrap call and before**
`require("default.hypr.omarchy")`:

```lua
-- macOS Screen Sharing sends Command as Alt and Option as Super over legacy
-- VNC. Translate those modifiers for every Hyprland shortcut from WayVNC,
-- while leaving physical and other keyboards unchanged.
local wayvnc_keyboard = "hl-virtual-keyboard-wayvnc"
local original_bind = hl.bind
local original_unbind = hl.unbind

local function copy_options(options)
  local copy = {}
  for key, value in pairs(options or {}) do
    copy[key] = value
  end
  return copy
end

local function swap_super_alt(keys)
  return keys
    :gsub("SUPER", "__WAYVNC_COMMAND__")
    :gsub("ALT", "SUPER")
    :gsub("__WAYVNC_COMMAND__", "ALT")
end

hl.bind = function(keys, dispatcher, options)
  local has_swapped_modifier = type(keys) == "string"
    and (keys:find("SUPER", 1, true) or keys:find("ALT", 1, true))

  -- Preserve explicitly device-scoped bindings exactly as authored.
  if not has_swapped_modifier or (options and options.devices) then
    return original_bind(keys, dispatcher, options)
  end

  local non_vnc_options = copy_options(options)
  non_vnc_options.devices = { inclusive = false, list = { wayvnc_keyboard } }
  original_bind(keys, dispatcher, non_vnc_options)

  local vnc_options = copy_options(options)
  vnc_options.devices = { inclusive = true, list = { wayvnc_keyboard } }
  return original_bind(swap_super_alt(keys), dispatcher, vnc_options)
end

hl.unbind = function(keys)
  original_unbind(keys)
  if type(keys) == "string"
      and (keys:find("SUPER", 1, true) or keys:find("ALT", 1, true)) then
    original_unbind(swap_super_alt(keys))
  end
end
```

The placement is important: Omarchy's default bindings are registered by
`require("default.hypr.omarchy")`, so the translation must be installed first.
Do not modify anything under `/usr/share/omarchy`; package updates overwrite
that directory.

This translation applies to Hyprland/Omarchy shortcuts. It does not rewrite
arbitrary modifier input inside applications unless that combination is also
registered as a Hyprland shortcut.

## 6. Validate Hyprland

Check the edited Lua files:

```bash
luac -p ~/.config/hypr/hyprland.lua
luac -p ~/.config/hypr/autostart.lua
```

Reload and check for errors:

```bash
hyprctl reload
hyprctl configerrors
```

`hyprctl configerrors` should print nothing.

To confirm that both the normal and translated bindings exist:

```bash
hyprctl binds -j | jq -c '
  .[]
  | select((.key == "J" or .key == "W" or .key == "RETURN")
      and (.modmask == 8 or .modmask == 64))
  | {modmask, key, description}
'
```

For these modifier masks, `64` is Super and `8` is Alt. You should see both
versions of shortcuts such as Terminal, Close window, and Toggle window split.

## 7. Connect from macOS

Make sure Tailscale is connected on the Mac, then either open **Screen Sharing**
from `/Applications/Utilities` or run:

```bash
open 'vnc://<TAILSCALE_IPV4>'
```

Enter the eight-character password from the WayVNC config. After connecting:

- `CMD+ENTER` opens the terminal.
- `CMD+W` closes the focused Omarchy window.
- `CMD+J` toggles the window split.
- Other Omarchy `SUPER` shortcuts work with Mac `CMD` as well.

## Troubleshooting

### Confirm WayVNC is running and listening only on Tailscale

```bash
pgrep -a wayvnc
ss -ltnp 'sport = :5900'
```

The listener should show the Tailscale address, not `0.0.0.0:5900`.

### Confirm the desktop output is captured

```bash
wayvncctl output-list
```

If no physical monitor is active, Hyprland may expose a fallback output. That
can still be shared remotely.

### Confirm the VNC keyboard exists

Run this while Screen Sharing is connected:

```bash
hyprctl devices -j | jq '
  .keyboards[]
  | select(.name == "hl-virtual-keyboard-wayvnc")
'
```

The virtual keyboard exists only while a VNC client is connected.

### WayVNC does not start over SSH

WayVNC needs the active Wayland session environment. Starting it through
Hyprland's autostart or `hyprctl eval` from a desktop terminal avoids missing
`WAYLAND_DISPLAY` and `XDG_RUNTIME_DIR` values. If administering exclusively
over SSH, starting a new graphical session is a separate concern.

## References

- [WayVNC documentation and macOS legacy authentication](https://github.com/any1/wayvnc/blob/master/README.md)
- [Hyprland per-device bindings](https://wiki.hypr.land/Configuring/Basics/Binds/#per-device-binds)
- [Tailscale CLI: obtaining the device IPv4 address](https://tailscale.com/kb/1080/cli#ip)
- [Tailscale transport encryption](https://tailscale.com/docs/concepts/tailscale-encryption)
