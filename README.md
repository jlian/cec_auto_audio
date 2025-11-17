# Auto-enable system audio mode when switching HDMI-CEC sources

Automatically wake up your AVR and enable System Audio Mode when switching to gaming consoles or media players via HDMI-CEC.

## Why this exists

When you turn on a console connected via HDMI-CEC, your TV switches inputs automatically, but the AVR often doesn't wake up or enable System Audio Mode - leaving you with TV speakers. This script monitors CEC traffic and sends the necessary command to activate your audio system.

## Features

- Watches CEC bus passively without interfering
- Only acts when playback devices (consoles) become active
- Cancels injection if the system resolves naturally
- Rate-limited to prevent command spam
- Dry-run mode for testing

## Requirements

- Python 3.6+
- `cec-client` from libCEC: `sudo apt-get install cec-utils`
- CEC-capable TV, AVR, and a device with CEC adapter (e.g., Raspberry Pi)

## Installation

```bash
git clone https://github.com/jlian/cec_auto_audio.git
cd cec_auto_audio
chmod +x cec_auto_audio.py
```

## Configuration

Edit these variables in `cec_auto_audio.py`:

```python
CONSOLE_LAS = {0x4, 0x8, 0xB}  # Playback device addresses
DENON_LA = 0x5                 # AVR address
PENDING_TIMEOUT_SEC = 0.5      # Wait time before injecting command
DRY_RUN = False                # Set True to test without sending commands
```

Find your device addresses: `echo 'scan' | cec-client -s -d 1`

## Usage

Run the script:
```bash
./cec_auto_audio.py
```

The script monitors CEC traffic and automatically sends System Audio Mode Request (`tx 15:70:00:00`) when needed.

### Running as a systemd service

Create `/etc/systemd/system/cec-auto-audio.service`:

```ini
[Unit]
Description=CEC Auto Audio Helper
After=network.target

[Service]
Type=simple
User=pi
ExecStart=/usr/bin/python3 /home/pi/cec_auto_audio/cec_auto_audio.py
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl enable cec-auto-audio.service
sudo systemctl start cec-auto-audio.service
```

## How it works

1. Monitors CEC traffic using `cec-client -d 8`
2. When a playback device sends Active Source (`0x82`), starts a timer
3. Waits briefly (0.5s) to see if AVR naturally sends Set System Audio Mode (`5f:72:01`)
4. If not, sends System Audio Mode Request (`tx 15:70:00:00`)
5. Rate-limits to prevent spam

## Troubleshooting

**Script doesn't start:** Check `cec-client` is installed and CEC adapter connected (`ls /dev/cec*`)

**AVR doesn't wake:** Verify addresses in config match your devices. Find addresses with: `echo 'scan' | cec-client -s -d 1`

**Too many commands:** Increase `MIN_INJECTION_INTERVAL_SEC` or check for duplicate addresses

## License

MIT License - Copyright (c) 2025 John Lian - see [LICENSE](LICENSE) file