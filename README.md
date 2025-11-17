# cec_auto_audio

Automatic HDMI-CEC audio system management for home theater setups with AVRs (Audio/Video Receivers) and gaming consoles.

## Overview

`cec_auto_audio` is a Python script that watches HDMI-CEC bus traffic and automatically enables System Audio Mode on your AVR when switching to gaming consoles or other playback devices. This solves the common issue where AVRs don't automatically wake up or switch to the correct audio source when you turn on your console.

## The Problem It Solves

In many home theater setups with a TV, AVR, and multiple gaming consoles connected via HDMI-CEC:
- When you turn on a console, it sends an "Active Source" CEC command
- The TV switches to that input automatically
- **But** the AVR sometimes doesn't wake up or enable System Audio Mode, leaving you with TV speakers instead of your home theater audio

This script monitors CEC traffic and automatically sends a "System Audio Mode Request" command to the AVR when it detects a console becoming the active source, ensuring your audio system wakes up and engages properly.

## Features

- **Passive monitoring**: Watches CEC bus traffic without interfering with normal operations
- **Smart detection**: Only acts when playback devices (consoles) become active
- **Avoids conflicts**: Cancels injection if the system naturally resolves itself
- **Rate limiting**: Prevents command spam with configurable timeouts
- **Dry-run mode**: Test behavior without sending actual commands
- **Configurable**: Easily adjust device addresses, timeouts, and behavior

## Requirements

- Python 3.6 or later (uses f-strings and type hints)
- `cec-client` from the libCEC library
- A CEC-capable TV, AVR, and connected devices
- A Raspberry Pi, Linux PC, or other system with CEC adapter support

### Installing cec-client

**On Debian/Ubuntu/Raspberry Pi OS:**
```bash
sudo apt-get update
sudo apt-get install cec-utils
```

**On other systems:**
Follow the [libCEC installation guide](https://github.com/Pulse-Eight/libcec)

## Installation

1. Clone this repository:
```bash
git clone https://github.com/jlian/cec_auto_audio.git
cd cec_auto_audio
```

2. Make the script executable:
```bash
chmod +x cec_auto_audio.py
```

3. (Optional) Install as a systemd service for automatic startup - see [Running as a Service](#running-as-a-service)

## Configuration

Edit the configuration variables at the top of `cec_auto_audio.py`:

```python
# Logical addresses of your playback devices (consoles, media players)
CONSOLE_LAS = {0x4, 0x8, 0xB}  # Adjust for your devices

# Logical address of your AVR (Audio device)
DENON_LA = 0x5  # Change if your AVR uses a different address

# Wait time before injecting command (seconds)
PENDING_TIMEOUT_SEC = 0.5

# Minimum time between injections to prevent spam (seconds)
MIN_INJECTION_INTERVAL_SEC = 3.0

# Set to True to see what would happen without sending commands
DRY_RUN = False
```

### Finding Your Device Addresses

To discover the logical addresses of your devices:

```bash
echo 'scan' | cec-client -s -d 1
```

Common logical addresses:
- `0x0`: TV
- `0x1`: Recording Device 1 (often the CEC client itself)
- `0x4`: Playback Device 1 (game console, media player)
- `0x5`: Audio System (AVR/soundbar)
- `0x8`: Playback Device 2
- `0xB`: Playback Device 3

## Usage

### Basic Usage

Start the script:
```bash
./cec_auto_audio.py
```

Or with Python explicitly:
```bash
python3 cec_auto_audio.py
```

The script will:
1. Start monitoring CEC traffic
2. Display all TRAFFIC events for debugging
3. Watch for Active Source changes from configured playback devices
4. Automatically send System Audio Mode Request when needed

### Dry-Run Mode

To test without sending any actual CEC commands:

1. Edit the script and set `DRY_RUN = True`
2. Run the script and verify it detects events correctly
3. Once satisfied, set `DRY_RUN = False` and restart

### Running as a Service

To run automatically on system startup, create a systemd service:

1. Create `/etc/systemd/system/cec-auto-audio.service`:
```ini
[Unit]
Description=CEC Auto Audio Helper
After=network.target

[Service]
Type=simple
User=pi
ExecStart=/usr/bin/python3 /home/pi/cec_auto_audio/cec_auto_audio.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

2. Enable and start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable cec-auto-audio.service
sudo systemctl start cec-auto-audio.service
```

3. Check status:
```bash
sudo systemctl status cec-auto-audio.service
```

4. View logs:
```bash
sudo journalctl -u cec-auto-audio.service -f
```

## How It Works

The script implements a smart state machine that:

1. **Monitors CEC Traffic**: Uses `cec-client -d 8` to watch all CEC bus messages

2. **Detects Active Source Changes**: When a configured playback device (console) sends an "Active Source" command (opcode `0x82`), the script starts a timer

3. **Waits for Natural Resolution**: Gives the system a brief moment (default 0.5 seconds) to see if the AVR naturally sends "Set System Audio Mode" (`5f:72:01`)

4. **Injects Command if Needed**: If the AVR doesn't respond within the timeout, sends a "System Audio Mode Request" command (`tx 15:70:00:00`) to wake up the audio system

5. **Prevents Spam**: Rate-limits injections to avoid overwhelming the CEC bus if something goes wrong

### CEC Commands Used

- **Monitored**: `0x82` (Active Source) - Indicates a device wants to become the active input
- **Monitored**: `0x72` with data `0x01` (Set System Audio Mode ON) - AVR enables audio
- **Sent**: `tx 15:70:00:00` - System Audio Mode Request from client (0x1) to AVR (0x5) on behalf of TV (0x0)

## Troubleshooting

### Script doesn't start
- Verify `cec-client` is installed: `which cec-client`
- Check CEC adapter is connected: `ls /dev/cec*`
- Try running with more verbose logging: Edit script and change `-d 8` to `-d 31`

### AVR still doesn't wake up
- Verify AVR logical address matches `DENON_LA` setting
- Check that your console's address is in `CONSOLE_LAS`
- Run in dry-run mode to see if events are detected
- Increase `PENDING_TIMEOUT_SEC` if your AVR is slow to respond

### Too many messages / commands being sent
- Increase `MIN_INJECTION_INTERVAL_SEC`
- Check for duplicate device addresses in `CONSOLE_LAS`
- Verify CEC bus isn't experiencing noise or errors

### Finding the right timeout values
- Start with defaults and observe behavior
- If AVR usually responds but sometimes misses it, increase `PENDING_TIMEOUT_SEC`
- If system is too responsive, decrease the timeout
- Monitor logs to see timing of natural `5f:72:01` messages

### Script stops responding
- Check if `cec-client` process crashed: `ps aux | grep cec-client`
- Look for "BrokenPipeError" in output
- Restart the script
- Consider running as systemd service with auto-restart

## Technical Details

**Default Setup Assumptions:**
- TV: Logical address `0x0`
- AVR (Audio System): Logical address `0x5`
- libCEC client: Logical address `0x1` (Recording Device 1)
- Consoles/Players: Logical addresses `0x4`, `0x8`, `0xB` (Playback devices)

**CEC Command Format:**
The script sends `tx 15:70:00:00` which breaks down as:
- `tx` - Transmit command
- `15` - Header byte: From `0x1` (client) to `0x5` (AVR)
- `70` - Opcode: System Audio Mode Request
- `00:00` - Physical address of TV (source of audio)

## Contributing

Contributions are welcome! Please feel free to submit issues or pull requests.

## License

MIT License - see [LICENSE](LICENSE) file for details.

Copyright (c) 2025 John Lian

## Acknowledgments

- Built using [libCEC](https://github.com/Pulse-Eight/libcec) by Pulse-Eight
- Inspired by the need for reliable CEC automation in modern home theater setups