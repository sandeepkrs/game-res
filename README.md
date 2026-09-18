# game-res

A lightweight, automated display resolution and fractional scaling manager for Linux (GNOME / Wayland & X11).

Designed specifically as a wrapper for games (e.g., Steam, Proton, Lutris, Heroic) and applications to switch monitor resolution or scaling on launch and automatically revert back to your original baseline display settings when the game exits.

---

## The Problem

On Linux GNOME (especially on Wayland with HiDPI / 4K displays and fractional scaling like 125% or 150%):
- **Launcher & Anti-Cheat Process Chains**: Modern games frequently launch through intermediate launchers (EA App, Ubisoft Connect, Battle.net) and anti-cheat services (Easy Anti-Cheat, BattlEye). Tools like **Gamescope** or Steam/Proton environment variables often fail, crash, or fail to propagate scaling fixes when the actual game binary is spawned as a detached child process from an anti-cheat daemon.
- **XWayland Scaling Blurriness**: Games rendered through XWayland under fractional scaling (e.g., 125% or 150%) appear noticeably soft or blurry due to compositor upscaling.
- **In-Game Resolution Bugs**: Changing resolution inside games on Wayland frequently leads to black screens, window alignment bugs, or crashes.

### The Solution

Rather than relying on process-injection, nested compositors, or environment variables that break across child process forks, **the most reliable solution is to adjust the host display's resolution or scaling directly at the desktop level**.

`game-res` temporarily switches your desktop resolution and/or forces 100% scaling via GNOME's native Mutter D-Bus interface before launching. Because the display server itself is adjusted, every process—the launcher, the anti-cheat daemon, and the game binary alike—natively inherits crisp 1:1 pixel rendering without requiring any process hooking. The moment the game or launcher exits, `game-res` automatically restores your exact baseline desktop layout.

---

## Features

- **Aspect-Ratio-Aware Presets**: Automatically detects your monitor's native aspect ratio (16:9, 16:10, 21:9 Ultrawide, 32:9 Super Ultrawide) and adapts presets (`FHD`, `2K`, `4K`) without stretching or distortion.
- **Scaling-Only Support**: Change only the scaling (e.g. `-s 100%` or `-s 1.0`) while keeping your native resolution intact.
- **Multi-Monitor Targeting**: Target the `primary` display, `external` display, `all` displays, or a specific monitor by index or connector name (`HDMI-1`, `DP-1`).
- **Baseline Configuration Protection (`game-res.conf`)**:
  - Automatically captures and locks your desktop's normal baseline settings on first run.
  - Baseline configuration is never accidentally overwritten unless explicitly updated with `-u`.
  - Remembers which monitor was last targeted for easy restoration.
- **Guaranteed Automatic Restore**:
  - Automatically restores previous display settings upon normal game exit, crash, or interrupt (`Ctrl+C`, `SIGINT`, `SIGTERM`, `SIGHUP`).
- **Interactive Standalone Mode**:
  - When invoked without a game command (e.g. `game-res -r 2K`), it switches settings and waits for `Enter` or `Ctrl+C` before restoring.
- **Native GNOME Mutter Integration**:
  - Uses GNOME's native `org.gnome.Mutter.DisplayConfig` D-Bus interface. Fully compatible with both Wayland and X11 sessions.

---

## Requirements

- **Operating System**: Linux with GNOME Desktop 3.36+ (Ubuntu, Fedora, Debian, Arch Linux, openSUSE, Pop!_OS, etc.)
- **Python**: Python 3.6+
- **System Dependency**: `PyGObject` (`gi.repository`), pre-installed by default on all standard GNOME desktop distributions.
  - *Ubuntu / Debian*: `python3-gi` (pre-installed)
  - *Fedora*: `python3-gobject` (pre-installed)
  - *Arch Linux*: `python-gobject` (pre-installed)

---

## Installation

1. Clone or download this repository:
   ```bash
   git clone https://github.com/your-username/game-res.git
   cd game-res
   ```

2. Make the script executable:
   ```bash
   chmod +x game-res
   ```

3. (Optional) Add to your `PATH` or symlink to `~/.local/bin` for system-wide access:
   ```bash
   mkdir -p ~/.local/bin
   ln -s "$(pwd)/game-res" ~/.local/bin/game-res
   ```

---

## Usage Examples

### 1. Launching Games via Command Line
```bash
# Switch primary monitor to 1080p and launch game
game-res -r FHD -- /path/to/game

# Switch external monitor to 2K (1440p) at 100% scaling and launch Steam game
game-res -m external -r 2K -s 100% -- /path/to/game

# Switch all connected monitors to 1080p
game-res -m all -r 1080p -- /path/to/game

# Keep current 4K resolution but temporarily switch scaling from 150% to 100%
game-res -s 100% -- /path/to/game
```

### 2. Using in Steam (Launch Options)
Right-click any game in Steam -> **Properties** -> **General** -> **Launch Options**:

- **Switch to 1440p / 2K at 100% scale while playing**:
  ```bash
  game-res -r 2K -s 100% -- %command%
  ```

- **Keep native resolution, but disable fractional scaling (force 100%)**:
  ```bash
  game-res -s 100% -- %command%
  ```

- **Target an external monitor specifically**:
  ```bash
  game-res -m external -r FHD -- %command%
  ```

When the game closes, your screen will automatically revert to your baseline resolution and scaling.

### 3. Standalone Mode (No Command)
You can use `game-res` without a command. It will apply the settings and hold them until you press **Enter** or **Ctrl+C**:
```bash
# Switch to 2K interactively
game-res -r 2K

# Set scaling to 125% interactively
game-res -s 125%
```

### 4. Status and Hardware Inspection
Run `game-res` without parameters to inspect your display hardware, active modes, and baseline configuration:
```bash
game-res
```

### 5. Restoring and Updating Baseline
```bash
# Restore display(s) to the saved baseline in game-res.conf
game-res --restore

# Restore only the external monitor
game-res --restore -m external

# Update baseline configuration with your current desktop layout
game-res -u
```

---

## CLI Options

| Flag | Argument | Description |
| :--- | :--- | :--- |
| `-r`, `--res`, `--resolution` | `FHD`, `2K`, `4K`, etc. | Preset tier. Automatically adapts to the screen's aspect ratio. |
| `-s`, `--scale` | `1.0`, `1.5`, `100%`, `150%` | Target display scale factor. |
| `-m`, `--monitor` | `primary`, `external`, `all`, index, name | Target monitor. Defaults to `primary`. |
| `-w`, `--width` | Integer (e.g. `1920`) | Custom target width in pixels (overrides preset). |
| `-h`, `--height` | Integer (e.g. `1080`) | Custom target height in pixels (overrides preset). |
| `-u`, `--update-conf` | None | Updates `game-res.conf` baseline with the active display setup. |
| `--restore` | None | Restores display settings to the saved baseline configuration. |
| `--` | Command | Delimiter separating `game-res` options from the command to launch. |

---

## Aspect Ratio Support

Presets dynamically select the optimal resolution for your monitor's native aspect ratio:

| Preset | 16:9 Standard | 16:10 Productivity | 21:9 Ultrawide | 32:9 Super Ultrawide |
| :--- | :--- | :--- | :--- | :--- |
| **`FHD` / `1080P`** | 1920 × 1080 | 1920 × 1200 | 2560 × 1080 | 3840 × 1080 |
| **`2K` / `1440P` / `QHD`** | 2560 × 1440 | 2560 × 1600 | 3440 × 1440 | 5120 × 1440 |
| **`4K` / `UHD`** | 3840 × 2160 | 3840 × 2400 | 5120 × 2160 | 7680 × 2160 |

---

## License

This project is licensed under the [MIT License](LICENSE).
