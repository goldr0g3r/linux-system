[Home](../README.md) · [↑ 04 Daily Driver](README.md) · [← Previous: 4.2 Video + audio](02-video-audio-pipewire.md) · **4.3 Bluetooth headset** · [Next: 4.4 Music →](04-music-streaming-local.md)

---

# 4.3 Bluetooth Headset — LDAC / AptX HD, Mic Switching, Auto-Reconnect

The single most tweakable sub-system of Linux audio in 2026. Bluetooth audio on PipeWire 1.x is excellent when configured correctly: **LDAC or AptX HD on A2DP for music, mSBC or LC3 on HSP/HFP for calls, automatic profile switching when a mic becomes active, and auto-reconnect on power-on.**

## The codec landscape

On A2DP (music-grade, one-way output) the Bluetooth spec allows multiple codecs, each negotiated between source (laptop) and sink (headset):

| Codec      | Bitrate       | Quality                   | Supported by                                          | License         |
| ---------- | ------------- | ------------------------- | ----------------------------------------------------- | --------------- |
| **SBC**    | 128–345 kbps  | Baseline                  | Every BT headset ever                                 | Free            |
| **SBC-XQ** | 453–552 kbps  | Transparent on most music | Most recent headsets via modified SBC tables          | Free            |
| **AAC**    | 128–256 kbps  | Good; codec-of-choice on Apple gear | Airpods, JBL, some Sony             | Free in Linux (decoder only; laptop encodes AAC via Fraunhofer FDK) |
| **aptX**   | 352 kbps      | Good                      | Qualcomm-licensed; most non-Apple headsets            | Free in Linux via `libopenaptx` |
| **aptX HD**| 576 kbps      | Very good                 | Higher-end headsets (Sony, B&W, Sennheiser PXC550)    | Free via `libfreeaptx` |
| **aptX LL**| 352 kbps low-lat | Good                    | Gaming headsets (Razer, some Bose)                    | Free via `libfreeaptx` |
| **aptX Adaptive** | 280–420 kbps variable | Very good      | Qualcomm's newer tier                                 | Proprietary, not in Linux out-of-box |
| **LDAC**   | 330 / 660 / 990 kbps | Very good to transparent | Sony, tech-savvy headsets (Bose QC45 as of 2024 fw) | Free via `libldac` |
| **LC3**    | ~160-345 kbps | Very good (LE Audio)      | Latest BT LE Audio headsets (2024+)                   | Free via `liblc3` |
| **LHDC / LLAC** | 400-900 kbps | Transparent            | Huawei / Panasonic / FiiO                             | Proprietary, partial Linux support |

**In practice on this TUF A17 + PipeWire 1.x + BlueZ 5 stack, your A2DP codec priority should be:**

```
LDAC > AptX HD > AptX LL > AptX > AAC > SBC-XQ > SBC
```

On HSP/HFP (bidirectional, mic + speaker, for calls):

| Codec  | Quality      | Notes                                                                  |
| ------ | ------------ | ---------------------------------------------------------------------- |
| **CVSD** | Terrible (8 kHz narrowband) | Legacy; avoid                                                  |
| **mSBC** | Decent (16 kHz wideband) | Works on virtually all HSP/HFP-1.7+ headsets                     |
| **LC3**  | Very good (LE Audio)     | Only on LE Audio headsets (2024+)                                |

Priority: `LC3 > mSBC > CVSD`.

## 4.3.1 Install the codec libraries

```bash
sudo apt install -y \
  libldac2 libldac-dev \
  libfreeaptx0 libfreeaptx-dev \
  liblc3-1 liblc3-dev \
  libspa-0.2-bluetooth \
  bluez bluez-tools blueman
```

Restart BlueZ to pick up the libraries:

```bash
sudo systemctl restart bluetooth.service
systemctl --user restart wireplumber.service pipewire.service
```

## 4.3.2 Enable high-quality A2DP in WirePlumber

WirePlumber's default policy already negotiates the best available codec, but explicitly pin the order to be safe:

```bash
mkdir -p ~/.config/wireplumber/wireplumber.conf.d
```

Create `~/.config/wireplumber/wireplumber.conf.d/51-bluez-codec-priority.conf`:

```lua
monitor.bluez.properties = {
  -- Enable all high-quality codecs:
  bluez5.enable-sbc-xq = true
  bluez5.enable-msbc = true
  bluez5.enable-hw-volume = true

  -- Codec priority list (first matching supported by the device wins):
  bluez5.codecs = [ ldac aptx_hd aptx_ll aptx aac sbc_xq sbc ]

  -- HSP/HFP backend; "native" (PipeWire-implemented) is faster than oFono.
  bluez5.hfphsp-backend = "native"

  -- Auto-connect profile when headset powers on:
  bluez5.auto-connect = [ hfp_hf hsp_hs a2dp_sink ]

  -- Prefer A2DP over HSP/HFP on connect (save battery by not enabling mic path)
  bluez5.default.rate = 48000
  bluez5.default.channels = 2
}
```

Apply:

```bash
systemctl --user restart wireplumber.service
```

## 4.3.3 Pair the headset

First time:

```bash
# Interactive pairing (easier than the GUI for verification):
bluetoothctl
```

Inside the `bluetoothctl` shell:

```
[bluetooth]# power on
[bluetooth]# agent on
[bluetooth]# default-agent
[bluetooth]# scan on
# Put your headset in pairing mode. Wait for it to appear:
# [NEW] Device XX:XX:XX:XX:XX:XX My Headset
[bluetooth]# pair XX:XX:XX:XX:XX:XX
[bluetooth]# trust XX:XX:XX:XX:XX:XX
[bluetooth]# connect XX:XX:XX:XX:XX:XX
[bluetooth]# scan off
[bluetooth]# quit
```

`trust` is important — without it, the headset won't auto-connect when it powers on next.

### Verify the active codec

```bash
# What codec is the current A2DP stream using?
pactl list sinks | grep -A 25 "Name: bluez_output"
# Look for the "api.bluez5.codec" property.
# Expect: "ldac" (or whatever your headset's highest-supported codec is).
```

Or via `qpwgraph`'s node properties inspector.

## 4.3.4 Switch profile between A2DP and HSP/HFP (music vs call)

When music is playing, you want A2DP (best quality, output only). When a call starts, you need HSP/HFP (bidirectional with mic). On PipeWire, switching is fast (< 1 s) but not always automatic. Two options:

### Option A: automatic via WirePlumber (recommended)

Create `~/.config/wireplumber/wireplumber.conf.d/52-bluez-auto-profile-switch.conf`:

```lua
-- Automatic profile switching: when any app opens a stream that wants
-- audio input (e.g. the mic), switch the headset from A2DP to HSP/HFP.
-- When the mic stream ends, switch back.

bluez_monitor.rules = {
  {
    matches = { { { "device.name", "matches", "bluez_card.*" } } },
    apply_properties = {
      ["bluez5.auto-connect-profile-switch"] = true,
      ["bluez5.roles"] = "[ hsp_hs hfp_hf a2dp_sink ]",
    },
  },
}
```

Restart wireplumber:

```bash
systemctl --user restart wireplumber.service
```

Test: open Discord or a video call tab in Firefox with mic enabled → headset should briefly drop audio, switch to HSP/HFP, resume. Close the call → switches back to A2DP.

### Option B: manual via `pactl`

If you want explicit control:

```bash
# List the Bluetooth card's profiles:
pactl list cards short
# Find bluez_card.XX_XX_XX_XX_XX_XX

pactl list cards | grep -A 30 "Name: bluez_card.XX_XX_XX_XX_XX_XX"

# Switch to A2DP (music):
pactl set-card-profile bluez_card.XX_XX_XX_XX_XX_XX a2dp-sink-ldac

# Switch to HSP/HFP (call):
pactl set-card-profile bluez_card.XX_XX_XX_XX_XX_XX headset-head-unit-msbc
```

Bind these to Konsole aliases:

```bash
# In ~/.zshrc or ~/.bashrc:
bt_music()  { pactl set-card-profile bluez_card.XX_XX_XX_XX_XX_XX a2dp-sink-ldac; }
bt_call()   { pactl set-card-profile bluez_card.XX_XX_XX_XX_XX_XX headset-head-unit-msbc; }
```

Replace the MAC address for your headset. `pactl list cards short` shows the right name.

## 4.3.5 Auto-reconnect on headset power-on

With `trust` set in [4.3.3](#433-pair-the-headset), BlueZ attempts reconnect when the headset advertises. But Linux does not by default listen for Bluetooth page scans continuously when idle (saves radio power). Fix with BlueZ's `AutoEnable`:

Edit `/etc/bluetooth/main.conf`:

```bash
sudoedit /etc/bluetooth/main.conf
```

Set or add:

```ini
[Policy]
AutoEnable=true

[General]
# Enable auto-pairing request:
ReconnectAttempts=7
ReconnectIntervals=1,2,4,8,16,32,64

# Prefer A2DP on connect — mic will only engage when an app requests it:
JustWorksRepairing=always
```

Restart:

```bash
sudo systemctl restart bluetooth.service
```

Test: turn off the headset, wait 10 seconds, turn it back on. It should auto-reconnect in 1–5 seconds.

## 4.3.6 Troubleshooting playbook

### Headset connects but no audio

```bash
# Is it the default sink?
pactl get-default-sink
# If not bluez_output.*:
pactl set-default-sink bluez_output.XX_XX_XX_XX_XX_XX.a2dp-sink
```

Or via Plasma system tray speaker icon → switch output device.

### Codec is SBC instead of LDAC

Either the headset doesn't support LDAC (check the manufacturer page — many "Sony" labelled headsets DON'T have LDAC) or the connection is noisy (LDAC 990 falls back to 660 / 330 or out to AptX / SBC). Check RSSI:

```bash
bluetoothctl info XX:XX:XX:XX:XX:XX | grep RSSI
# Closer to 0 = stronger signal. -60 is fine, -80 is weak.
```

### Crackling / popping

Usually buffer underruns. Edit `~/.config/pipewire/pipewire.conf` (created in [4.2.7](02-video-audio-pipewire.md#427-audio-defaults--sample-rate-buffer-size)) and raise `default.clock.quantum` to 2048 or 4096. Larger = more latency, fewer underruns.

Or, for bluetooth specifically:

```lua
-- ~/.config/wireplumber/wireplumber.conf.d/53-bluez-buffer.conf
bluez5.default.quantum = 2048
```

### HSP/HFP audio is terrible

Make sure mSBC is in use, not CVSD:

```bash
pactl list cards | grep -A 5 "headset-head-unit"
# Look for "headset-head-unit-msbc" or "headset-head-unit-lc3" profile.
# If only "headset-head-unit" (plain) is listed, the headset doesn't report mSBC.
# Workaround: verify the headset firmware is up-to-date on the manufacturer app.
```

### My headset mic works in Discord but not in Firefox calls

Firefox 148 on Wayland uses xdg-desktop-portal for mic access. Make sure `xdg-desktop-portal-kde` is installed and running. If still broken, the fallback is `MOZ_WAYLAND_USE_VAAPI=1 MOZ_ENABLE_WAYLAND=1 firefox`.

## 4.3.7 Snapshot

```bash
sudo timeshift --create --comments "post-bluetooth-headset $(date -I)" --tags D
```

Proceed to [4.4 Music streaming and local libraries](04-music-streaming-local.md).

---

[Home](../README.md) · [↑ 04 Daily Driver](README.md) · [← Previous: 4.2 Video + audio](02-video-audio-pipewire.md) · **4.3 Bluetooth headset** · [Next: 4.4 Music →](04-music-streaming-local.md)
