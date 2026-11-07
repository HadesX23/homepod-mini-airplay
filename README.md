# homepod-mini-airplay

**Enable HomePod Mini (or any AirPlay 2 speaker) as audio output on Linux via PipeWire + Owntone bridge.**

## What this does

- Creates a PipeWire null sink (`homepod_airplay`) that appears as "HomePod Mini" in your audio settings
- Bridges audio from that sink → Owntone (fork of mt-daapd/Forked-daapd) → AirPlay 2 → HomePod Mini
- Auto-reconnects AirPlay session when you switch audio output away and back (2-second grace period)
- Multimedia keys (volume up/down/mute) control **actual HomePod volume** via Owntone API, not PipeWire sink volume
- Disables Easy Effects which otherwise hijacks all streams

## Requirements

- Linux with PipeWire + WirePlumber
- `owntone` (AirPlay 2 sender, formerly forked-daapd)
- `jq`, `curl`, `parec` (from `pipewire-audio` / `pipewire-pulse`)
- HomePod Mini (or any AirPlay 2 receiver) on same network

## Quick Install

```bash
# 1. Install dependencies (Arch/CachyOS)
sudo pacman -S owntone pipewire wireplumber jq curl pipewire-audio pipewire-pulse

# 2. Clone this skill
git clone https://github.com/hadesx23/homepod-mini-airplay.git
cd homepod-mini-airplay

# 3. Deploy scripts
mkdir -p ~/.local/bin ~/.local/share/homepod-pipe
cp assets/homepod-bridge ~/.local/bin/
cp assets/homepod-volume ~/.local/bin/
chmod +x ~/.local/bin/homepod-bridge ~/.local/bin/homepod-volume

# 4. Configure Owntone
mkdir -p ~/.config/owntone
cp references/owntone.conf.example ~/.config/owntone/owntone.conf
# Edit paths in owntone.conf to match your $HOME

# 5. Deploy systemd services
mkdir -p ~/.config/systemd/user
cp references/homepod-bridge.service ~/.config/systemd/user/
cp references/owntone.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now homepod-bridge.service owntone.service

# 6. Set default sink to HomePod Mini
pactl set-default-sink homepod_airplay
# Or via WirePlumber metadata (persists across reboots):
# pw-metadata -n settings 0 default.audio.sink <sink-id>

# 7. Add Hyprland keybindings (see references/bindings.lua.snippet)
# Add to ~/.config/hypr/bindings.lua

# 8. Disable Easy Effects (it intercepts streams)
mv ~/.config/autostart/com.github.wwmm.easyeffects.desktop{,.disabled} 2>/dev/null
pkill easyeffects 2>/dev/null

# 9. Verify
wpctl status  # Should show homepod_airplay as default sink
# Play audio → check wpctl status shows streams → homepod_airplay
```

## How it works

```
App (Brave/Spotify/etc.)
    │
    ▼
PipeWire sink: homepod_airplay (default)
    │
    ▼
homepod-bridge (parec on .monitor)
    │
    ▼
FIFO: ~/.local/share/homepod-pipe/homepod.pcm
    │
    ▼
Owntone pipe input → AirPlay 2 (RAOP) → HomePod Mini
```

### Auto-reconnection

When you switch output away from HomePod:
1. Bridge detects 0 active streams on `homepod_airplay`
2. After `GRACE=2s`: stops parec, marks `WAS_IDLE=true`, logs "idle, session released"

When you switch back to HomePod:
1. Bridge detects new stream on `homepod_airplay`
2. Sees `WAS_IDLE=true` → **restarts `owntone.service`** (clean RAOP reconnect)
3. Starts new parec → selects output → `player/play`
4. Audio flows again automatically

## Volume Control

The `homepod-volume` CLI controls **Owntone output volume** (0–100), not PipeWire sink volume:

```bash
homepod-volume up       # +5%
homepod-volume down     # -5%
homepod-volume set 50   # absolute
homepod-volume set 0    # mute (toggle: set 0 again to restore)
homepod-volume mute     # toggle mute/unmute
homepod-volume get      # show current
```

Multimedia keys bound in Hyprland:
- `XF86AudioRaiseVolume` → `homepod-volume up`
- `XF86AudioLowerVolume` → `homepod-volume down`
- `XF86AudioMute` → `homepod-volume mute`

**Critical**: Default Omarchy volume bindings are unbound first to avoid dual control.

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| No `homepod_airplay` sink | `systemctl --user restart homepod-bridge.service owntone.service` |
| Audio plays on wrong device | `pavucontrol` → Playback → set app to "HomePod Mini" |
| Volume keys change PipeWire volume | Unbind defaults in Hyprland: `hl.unbind("XF86AudioRaiseVolume")` etc. |
| "Incomplete read" in Owntone logs | `systemctl --user restart owntone.service` (buffer underrun) |
| No audio after switching back | Bridge auto-restarts owntone; wait 3s or `systemctl --user restart owntone.service homepod-bridge.service` |
| HomePod not discovered | Check mDNS/Avahi: `avahi-browse -rt _airplay._tcp` |

## Files

```
skills/homepod-mini-airplay/
├── SKILL.md                    # Main skill (this doc)
├── assets/
│   ├── homepod-bridge          # Bridge daemon (executable)
│   └── homepod-volume          # Volume CLI (executable)
└── references/
    ├── owntone.conf.example    # Owntone config template
    ├── homepod-bridge.service  # systemd unit
    ├── owntone.service         # systemd unit
    └── bindings.lua.snippet    # Hyprland keybindings
```

## License

Apache-2.0