# aipicam-config

Ships `/etc/aipicam/streams.conf`, the single source of truth for `picam-raw`'s raw camera stream dimensions and ports.

## Why this exists

`picam-raw` streams raw YUV420 video over a custom chunked-UDP protocol with **no width/height field in the wire format** (see [picam-raw's README](https://github.com/jacohanekom/picam-raw)). Every reader of that stream — `picam-orchestrator`, `picam-hailo`, `picam-recorder` — has to already know the frame dimensions out of band, from its own config file.

Before this package existed, each of those four packages shipped its own hand-copied width/height/port defaults. They drifted: `picam-orchestrator` shipped `1920x1080` for the main stream while `picam-raw` actually captures and sends `2304x1296`. Nothing errors on this — the receiver just misreads every row and both chroma planes at the wrong offset, which looks like random frame corruption and gives no indication the actual cause is a config mismatch.

This package makes the values a single, shared fact instead of four independently-editable copies.

## Contents

```ini
[stream]
main_width      = 2304
main_height     = 1296
lores_width     = 640
lores_height    = 360

main_port       = 8560
lores_port      = 8561
telemetry_port  = 8555
command_port    = 8556
```

## Who reads this

| Package | Reads |
|---|---|
| `picam-raw` | all of it — this is what it actually captures/serves |
| `picam-orchestrator` | all of it |
| `picam-hailo` | `lores_width`, `lores_height`, `lores_port` |
| `picam-recorder` | `main_width`, `main_height`, `main_port` |

Each package's own config file can still explicitly set these same keys to override the shared file for local debugging — but the packaged defaults no longer duplicate these values, so a fresh install of any combination of these packages always agrees.

## Editing

Edit `/etc/aipicam/streams.conf` directly, then restart every camera-pipeline service:

```bash
sudo systemctl restart picam-raw picam-orchestrator picam-hailo picam-recorder
```

## Install

### From the APT repository

```bash
curl -fsSL https://apt.aipicam.com/pubkey.asc | sudo gpg --dearmor -o /usr/share/keyrings/aipicam.gpg
echo "deb [signed-by=/usr/share/keyrings/aipicam.gpg] https://apt.aipicam.com main main" | sudo tee /etc/apt/sources.list.d/aipicam.list
sudo apt-get update
sudo apt-get install aipicam-config
```

It's also pulled in automatically as a dependency of `picam-raw`, `picam-orchestrator`, `picam-hailo`, and `picam-recorder`.

### Build the .deb yourself

```bash
dpkg-buildpackage -us -uc -b
sudo dpkg -i ../aipicam-config_*.deb
```

This package installs no binary and runs no service — it's just a conffile.
