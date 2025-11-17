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

- A Raspberry Pi (or some other CEC-capable device) connected to your AVR or TV with HDMI (USB-HDMI adaptors won't work) on the CEC-enabled port (usually 0)
- Python 3.6+
- `cec-client` from libCEC: `sudo apt-get install cec-utils`

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

You can temporarily set `DRY_RUN = True` while sniffing behavior; you’ll see log lines like:

```text
[AUTO 00:18:19] Playback/console at logical B became Active Source (phys 36:00).
[AUTO 00:18:20] [DRY RUN] Would send: tx 15:70:00:00
```

Once you’re happy, flip `DRY_RUN = False`.

### Running as a systemd service

On the device, create a service file:

```bash
sudo nano /etc/systemd/system/cec-auto-audio.service
```

Example unit:

```ini
[Unit]
Description=CEC auto audio helper (Denon + consoles)
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /opt/cec-auto-audio/cec_auto_audio.py
Restart=on-failure
User=pi
Group=pi
WorkingDirectory=/opt/cec-auto-audio
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl enable cec-auto-audio.service
sudo systemctl start cec-auto-audio.service

# Tail logs
journalctl -u cec-auto-audio.service -f
```

Because we print both our own `[INFO]` / `[AUTO]` lines and the raw `cec-client` output, the journal doubles as a trace buffer. `journald` handles rotation automatically; you don’t need to babysit log files.

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
