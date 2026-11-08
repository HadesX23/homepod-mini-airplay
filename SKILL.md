---
name: homepod-mini-airplay
description: "Trigger: HomePod Mini, AirPlay, PipeWire, Owntone, audio output. Enable HomePod Mini as AirPlay 2 audio output on Linux via PipeWire + Owntone bridge."
license: Apache-2.0
metadata:
  author: "hadesx23"
  version: "1.0"
---

## Activation Contract

Load this skill when the user wants to:
- Set up HomePod Mini (or any AirPlay 2 speaker) as audio output on Linux
- Fix broken AirPlay streaming (buffer underruns, session drops, no audio)
- Control HomePod volume via multimedia keys (XF86AudioRaiseVolume/LowerVolume/Mute)
- Auto-reconnect AirPlay session when switching audio outputs back to HomePod
- Disable competing audio processors (Easy Effects) that hijack streams
- Control playback via Siri ("Hey Siri, pause/play music") using Homebridge
- Handle iPhone/other device AirPlay takeover gracefully

## Hard Rules

- **PipeWire sink must be default** — `pactl set-default-sink homepod_airplay` (or WirePlumber metadata)
- **Easy Effects must be stopped/disabled** — it intercepts all streams to `easyeffects_sink`
- **Owntone API port 3689** — bridge and volume control require Owntone running with pipe input
- **Bridge watches `homepod_airplay` sink** — triggers on `active_streams > 0` via `parec` on monitor
- **Volume keys unbind Omarchy defaults** — `hl.unbind` before `o.bind` to avoid dual control
- **AirPlay reconnection on return** — bridge restarts `owntone.service` when `WAS_IDLE=true` and streams reappear
- **GRACE=2s idle timeout** — release session quickly when output changes away
- **Volume clamp 0–100** — Owntone API rejects negatives; mute = 0 with toggle restore
- **Firewall must allow Homebridge**: `51826/tcp` (HAP) + `5353/udp` (mDNS)

## Decision Gates

| Situation | Action |
|-----------|--------|
| No `homepod_airplay` sink | `systemctl --user start homepod-bridge.service owntone.service` |
| Audio plays but not on HomePod | `pavucontrol` → Playback tab → set app to "HomePod Mini" |
| Volume keys change PipeWire volume | `hl.unbind("XF86AudioRaiseVolume")` + `hl.unbind("XF86AudioLowerVolume")` + `hl.unbind("XF86AudioMute")` then bind custom |
| "Incomplete read" in Owntone logs | `systemctl --user restart owntone.service` (buffer underrun) |
| Switch output away then back | Bridge auto-restarts owntone + session (GRACE=2s) |
| openclaw watchdog restarts service | `AGENT_STACK_NO_RECOVER=openclaw-gateway.service` in `~/.config/agent-stack/host.env` |

## Execution Steps

1. **Install dependencies**: `owntone`, `pipewire`, `wireplumber`, `jq`, `curl`
2. **Create PipeWire null sink**: `homepod_airplay` (via WirePlumber config or `pw-cli`)
3. **Configure Owntone** (`~/.config/owntone/owntone.conf`):
   - `library.directories = { "/home/user/.local/share/homepod-pipe" }`
   - `pipe_autostart = true`, `pipe_sample_rate = 44100`, `pipe_bits_per_sample = 16`
   - `audio.type = "disabled"` (no local output)
4. **Deploy bridge script** → `~/.local/bin/homepod-bridge` (executable)
   - Watches `homepod_airplay` sink, starts `parec` → FIFO → Owntone pipe → AirPlay 2
   - On idle→active: if `WAS_IDLE` restart `owntone.service` then `player/play`
   - `GRACE=2`, `COOLDOWN=6`, `POLL=0.25`
5. **Deploy volume script** → `~/.local/bin/homepod-volume` (executable)
   - `up`/`down`/`set N`/`mute`/`get` via Owntone API `/api/outputs/{id}`
   - Persists last volume in `/tmp/homepod-volume-state` for mute toggle
6. **Create systemd user services**:
   - `homepod-bridge.service` (Type=simple, Restart=on-failure)
   - `owntone.service` (Type=simple, Restart=on-failure)
7. **WirePlumber metadata**: set default sink to `homepod_airplay`
   - `pw-metadata -n settings 0 default.audio.sink <sink-id>`
8. **Hyprland bindings** (`~/.config/hypr/bindings.lua`):
   - Unbind defaults FIRST, then bind `XF86AudioRaiseVolume/LowerVolume/Mute` to `homepod-volume`
9. **Disable Easy Effects autostart**: `mv ~/.config/autostart/com.github.wwmm.easyeffects.desktop{,.disabled}`
10. **Optional: Homebridge for Siri control**:
    - Install: `npm install -g homebridge homebridge-config-ui-x homebridge-http-webhooks`
    - Deploy config: `cp references/homebridge-config.json ~/.homebridge/config.json`
    - Deploy service: `cp references/homebridge.service ~/.config/systemd/user/`
    - Firewall: `sudo ufw allow 51826/tcp && sudo ufw allow 5353/udp`
    - Start: `systemctl --user daemon-reload && systemctl --user enable --now homebridge.service`
    - Pair: Open Home app on iPhone → Add Accessory → Enter PIN 123-45-678
11. **Verify**: play audio → check `wpctl status` shows streams → `homepod_airplay` → Owntone `state=play`

## Output Contract

Return a working HomePod Mini AirPlay 2 setup with:
- `homepod_airplay` as default PipeWire sink
- Auto-reconnect on output switch (2s grace)
- Multimedia keys control HomePod volume (0–100, mute toggle)
- No Easy Effects interference
- Owntone stable (no buffer underruns)

## Error Resolutions (from debugging)

| Error / Symptom | Root Cause | Fix |
|-----------------|------------|-----|
| Audio plays but not on HomePod | Easy Effects intercepts all streams to `easyeffects_sink` | `pkill easyeffects` + disable autostart (`mv ~/.config/autostart/com.github.wwmm.easyeffects.desktop{,.disabled}`) |
| Volume keys change PipeWire sink volume instead of HomePod | Omarchy default bindings conflict with custom bindings | `hl.unbind("XF86AudioRaiseVolume")` + `hl.unbind("XF86AudioLowerVolume")` + `hl.unbind("XF86AudioMute")` BEFORE `o.bind()` |
| "Incomplete read, wanted 1764, got 268" in Owntone logs | Buffer underrun / broken RAOP pipe after output switch | Bridge restarts `owntone.service` on idle→active transition (`WAS_IDLE=true` triggers full reconnect) |
| HomePod plays then stops after volume change | PipeWire stream-restore module restores old sink routing | `pactl unload-module module-stream-restore && pactl load-module module-stream-restore restore_device=false` |
| Volume goes below 0 / keeps decreasing past mute | Double volume control (PipeWire + Owntone) | Unbind Omarchy defaults; clamp Owntone volume 0–100 in script |
| openclaw restarts after `systemctl --user disable --now` | Agent-stack watchdog with `Restart=always`, `StartLimitIntervalSec=0` | `AGENT_STACK_NO_RECOVER=openclaw-gateway.service` in `~/.config/agent-stack/host.env` |
| No audio after switching back to HomePod | Owntone RAOP session stale, pipe broken | Bridge detects `WAS_IDLE=true` → `systemctl --user restart owntone.service` → new session |
| `write() failed: Broken pipe` in bridge logs | parec writing to FIFO after Owntone closed pipe | Bridge restarts session on next active stream detection |
| iPhone takes over AirPlay, PC audio stops and doesn't recover | HomePod only supports one AirPlay source; Owntone session stale when iPhone releases | Bridge polls HomePod IP, waits for availability before restarting owntone (`HOMEPOD_IP` env var) |
| Brave/YouTube Music clicks don't work (only keyboard) | Vulkan+Wayland conflict in Chromium | Launch Brave with `--disable-features=Vulkan` (keeps GPU acceleration) or disable Vulkan in `brave://flags` |

## References

- `assets/homepod-bridge` — bridge daemon script (with WAS_IDLE auto-reconnect, HomePod availability polling)
- `assets/homepod-volume` — volume CLI (0–100 clamp, mute toggle, state persistence)
- `references/owntone.conf.example` — Owntone config template
- `references/homepod-bridge.service` — systemd unit
- `references/owntone.service` — systemd unit
- `references/bindings.lua.snippet` — Hyprland keybindings (unbinds Omarchy defaults first)
- `references/homebridge-config.json` — Homebridge config for Siri control ("Hey Siri, pause")
- `references/homebridge.service` — Homebridge systemd unit