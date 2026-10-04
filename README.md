# Pull Request: Standalone Debian Package (.deb), Application Menu Integration, and Universal Build System

## Summary of Changes

In this pull request, a complete standalone Debian package (`.deb`) and build pipeline have been developed for `nvitop`. This allows users on **Ubuntu (22.04+)**, **Debian (12+)**, **Kali Linux**, and other Debian-based distributions to install and run `nvitop` immediately as a native application without needing `pip`, virtual environments, or manual configuration.

Furthermore, seamless desktop environment integration has been implemented so `nvitop` is available directly in system application menus (GNOME, KDE Plasma, XFCE / Kali Whisker Menu, etc.) with a dedicated application icon and automatic terminal launcher.

---

## What Has Been Done

### 1. Standalone Compilation & Packaging (`build_deb.sh`)
- An automated packaging script, `build_deb.sh`, has been authored.
- The Python runtime and all dependencies (`nvidia-ml-py`, `psutil`) are compiled and frozen using PyInstaller.
- Compilation is performed inside an Ubuntu 22.04 LTS container environment to guarantee backward compatibility with **GLIBC 2.35+**, ensuring the resulting binary executes seamlessly across older and newer distributions alike (Ubuntu 22.04+, Debian 12+, Kali Linux Rolling).
- This completely bypasses the **PEP 668** (`externally-managed-environment`) restriction present on modern Debian, Ubuntu, and Kali installations that blocks `pip install`.
- Both `nvitop` and the `nvisel` (CUDA Visible Devices selection utility) entry points have been bundled into a unified multi-call dispatcher binary to minimize disk footprint.

### 2. Desktop & Application Menu Integration
- A standardized FreeDesktop entry has been created (`/usr/share/applications/nvitop.desktop`).
- A dedicated terminal launcher wrapper has been implemented (`/usr/bin/nvitop-launcher`). When launched from a graphical application menu (GNOME Dash, KDE Kickoff, XFCE Whisker Menu), the launcher automatically identifies and spawns the user's preferred terminal emulator (`x-terminal-emulator`, `gnome-terminal`, `xfce4-terminal`, `konsole`, `alacritty`, `kitty`, etc.). When invoked from an existing terminal, it runs directly.
- High-resolution application icons have been created and installed:
  - Scalable vector icon: `/usr/share/icons/hicolor/scalable/apps/nvitop.svg`
  - Multi-resolution raster icons: `/usr/share/icons/hicolor/{16,32,48,64,128,256}x{16,32,48,64,128,256}/apps/nvitop.png`
  - Pixmaps fallbacks: `/usr/share/pixmaps/nvitop.{svg,png}`

### 3. Debian Package Standards & Automation
- Full package metadata has been defined in `DEBIAN/control` with correct dependencies (`libc6 >= 2.14`).
- Package maintainer scripts (`DEBIAN/postinst` and `DEBIAN/postrm`) have been written to automatically trigger `update-desktop-database` and `gtk-update-icon-cache` upon installation and removal.
- Package ownership has been normalized using `dpkg-deb --root-owner-group` with universal `xz` compression.

---

## Files Included in This Submission

| File | Description |
| :--- | :--- |
| `build_deb.sh` | Shell script to build the complete `.deb` package and assets automatically |
| `nvitop_1.7.1-1_amd64.deb` | The compiled standalone `.deb` package ready for installation (~6.7 MB) |
| `PULL_REQUEST.md` | Detailed documentation of changes, package contents, and app shortcut reference |

### File Hierarchy Inside the Package:
```text
/
├── usr/
│   ├── bin/
│   │   ├── nvitop -> /usr/lib/nvitop/nvitop
│   │   ├── nvisel -> /usr/lib/nvitop/nvisel
│   │   └── nvitop-launcher
│   ├── lib/
│   │   └── nvitop/
│   │       ├── nvitop (compiled ELF binary)
│   │       ├── nvisel -> nvitop
│   │       └── _internal/ (bundled Python runtime, shared libraries, and modules)
│   └── share/
│       ├── applications/
│       │   └── nvitop.desktop
│       ├── icons/
│       │   └── hicolor/
│       │       ├── scalable/apps/nvitop.svg
│       │       ├── 16x16/apps/nvitop.png
│       │       ├── 32x32/apps/nvitop.png
│       │       ├── 48x48/apps/nvitop.png
│       │       ├── 64x64/apps/nvitop.png
│       │       ├── 128x128/apps/nvitop.png
│       │       └── 256x256/apps/nvitop.png
│       ├── pixmaps/
│       │   ├── nvitop.svg
│       │   └── nvitop.png
│       └── doc/
│           └── nvitop/
│               ├── copyright
│               └── README.md
```

---

## Installation & Removal Instructions

### To Install the Debian Package:
```bash
sudo dpkg -i nvitop_1.7.1-1_amd64.deb
# Or via apt to resolve any standard system dependencies automatically:
sudo apt install ./nvitop_1.7.1-1_amd64.deb
```

### To Remove:
```bash
sudo dpkg -r nvitop
# Or:
sudo apt remove nvitop
```

### To Rebuild from Source at Any Time:
```bash
./build_deb.sh
```

---

## Application Documentation & Shortcuts Reference

### 1. Display Modes & Visual Bar Charts
`nvitop` features real-time colorized visual bar charts and monitoring views:

- **Auto Mode (Default):** Dynamically adjusts the visual presentation according to the dimensions of the terminal window.
  ```bash
  nvitop -m auto       # Shortcut inside app: 'a'
  ```
- **Full Mode:** Displays comprehensive bar charts for all metrics (GPU utilization, GPU memory percentage, power cap, temperature, and CPU/memory gauges).
  ```bash
  nvitop -m full       # Shortcut inside app: 'f'
  ```
- **Compact Mode:** Displays a minimal overview table suitable for narrow terminals or side panes.
  ```bash
  nvitop -m compact    # Shortcut inside app: 'c'
  ```
- **Colorful Spectrum Mode:** Activates 256-color gradient spectrum charts where visual bars transition smoothly from green (low) to yellow (moderate) to red (high load).
  ```bash
  nvitop --colorful
  ```
- **One-Shot Snapshot Mode:** Prints device metrics once and exits immediately (analogous to `nvidia-smi`).
  ```bash
  nvitop -1            # or: nvitop --once
  ```

---

### 2. Process Contexts (Graphics vs. Compute)
Workloads are classified into two execution contexts under the `TYPE` column:
- **`G` (Graphics):** 3D, desktop window management, and display processes (e.g. Xorg, Wayland, browser rendering).
- **`C` (Compute):** Pure CUDA / AI compute processes (e.g. PyTorch, TensorFlow, Ollama).
- **`C+G`:** Mixed workloads utilizing both contexts.

#### Process Metrics:
- **`%SM`:** Percentage of GPU Streaming Multiprocessors (compute engines) utilized by the process.
- **`%GMBW`:** Percentage of GPU Memory Bandwidth utilized.
- **`%CPU` / `%MEM`:** Host CPU and RAM percentage consumed by the process.

#### Process Filtering Options:
```bash
nvitop --graphics       # Show processes with graphics context ('G' or 'C+G') [shortcut: -g]
nvitop --only-graphics  # Show exclusively pure graphics processes ('G' only) [shortcut: -G]
nvitop --compute        # Show processes with compute context ('C' or 'C+G') [shortcut: -c]
nvitop --only-compute   # Show exclusively pure compute processes ('C' only) [shortcut: -C]
nvitop --user [USER]    # Filter processes by username (default: current $USER) [shortcut: -u]
nvitop --pid [PID ...]  # Filter processes by process ID [shortcut: -p]
```

---

### 3. Interactive Keyboard Shortcuts (Monitor Mode)

When `nvitop` is actively running, the following keyboard controls are supported:

| Key | Action |
| :---: | :--- |
| **`<Enter>`** / **`<Return>`** | **Open Process Metrics & Live History Graph Screen:** Displays real-time timeline graphs of CPU %, Memory %, and GPU utilization for the selected process |
| **`t`** | **Toggle Process Tree View:** Displays hierarchical parent-child process relationships |
| **`f`** | Switch to **Full** display mode (complete graphical bars) |
| **`c`** | Switch to **Compact** display mode |
| **`a`** | Switch to **Auto** display mode |
| **`h`** / **`?`** | Open the interactive **Help & Keybindings** screen |
| **`q`** | Quit and return to the terminal |
| **`<Up>`** / **`<Down>`** | Navigate and highlight a process |
| **`<Home>`** / **`<End>`** | Jump to the first / last process |
| **`<Space>`** | Tag / untag the highlighted process |
| **`<Esc>`** | Clear process selection |
| **`Ctrl-C`** / **`I`** | Send `SIGINT` (Interrupt) to the selected process |
| **`T`** | Send `SIGTERM` (Terminate) to the selected process |
| **`K`** | Send `SIGKILL` (Force Kill) to the selected process |
| **`e`** | Inspect the environment variables of the selected process |
| **`r`** / **`<F5>`** | Force refresh the terminal screen |

#### Sorting Controls:
| Key | Sort Action |
| :---: | :--- |
| **`,`** / **`.`** | Move sort column left / right |
| **`/`** | Invert current sort order (ascending / descending) |
| **`on`** / **`oN`** | Sort by GPU ID (natural order) |
| **`og`** / **`oG`** | Sort by GPU memory usage (`GPU-MEM`) |
| **`os`** / **`oS`** | Sort by Streaming Multiprocessor utilization (`%SM`) |
| **`oc`** / **`oC`** | Sort by CPU percentage (`%CPU`) |
| **`om`** / **`oM`** | Sort by RAM percentage (`%MEM`) |
| **`ou`** / **`oU`** | Sort by username (`USER`) |
| **`op`** / **`oP`** | Sort by process ID (`PID`) |
| **`ot`** / **`oT`** | Sort by elapsed execution time (`TIME`) |

---

### 4. CUDA Visible Devices Selector (`nvisel`)
The package also provides `nvisel` for programmatic GPU selection in scripts:
```bash
# Print sorted available GPUs:
nvisel

# Select 2 devices with at least 8 GiB free memory and max 50% GPU utilization:
nvisel --min-count 2 --min-free-memory 8GiB --max-gpu-utilization 50

# Set environment variable automatically:
export CUDA_VISIBLE_DEVICES="$(nvisel -c 1 -f 10GiB)"
```
