# 00 · Environment Setup

How the development machine was set up for this project, with the reasoning behind each step
and the problems encountered along the way.

## System

| Component | Version / Details |
|---|---|
| Laptop | HP OMEN MAX 16 (ak0xxx) |
| GPU | NVIDIA GeForce RTX 5070 Ti Laptop GPU, 12 GB VRAM (Blackwell) + AMD integrated GPU (hybrid) |
| OS | Ubuntu 24.04 LTS (noble), HWE kernel 7.0.0-38-generic |
| NVIDIA driver | 595.99.02, open kernel modules (`nvidia-driver-595-open`) |
| ROS 2 | Jazzy |
| Python tooling | uv 0.13.1 |

## 1. GPU identification

```bash
lspci | grep -iE "vga|3d"
nvidia-smi
```

- `lspci` showed `NVIDIA Corporation Device 2f18`: the local PCI ID database predates the GPU,
  so only the raw ID is shown. `0x2F18` = RTX 5070 Ti Laptop GPU.
- **Blackwell implications:** requires NVIDIA's *open* kernel modules (driver ≥ 570) and PyTorch builds
  with CUDA ≥ 12.8 (`sm_120` kernels). Older builds fail with
  `no kernel image is available for execution on the device`.
- The "CUDA Version" in `nvidia-smi` is the *maximum* CUDA version the driver supports,
  not an installed toolkit. PyTorch ships its own CUDA runtime; only the driver is required.

## 2. System update

```bash
sudo apt update
sudo apt upgrade
```

Done on the fresh system for security fixes and consistent ROS 2 package versions.

## 3. Problem: NVIDIA driver lost after kernel update

**Symptom:** after `apt upgrade` and a reboot, `nvidia-smi` failed with
`NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver`.

**Diagnosis** (read-only checks before changing anything):

```bash
uname -r                       # running kernel: 7.0.0-38-generic (new)
dkms status                    # dkms not installed → driver uses precompiled modules
mokutil --sb-state             # Secure Boot disabled → signing ruled out
lsmod | grep nvidia            # no nvidia module loaded
find /lib/modules -name "nvidia.ko*"
# → module only present for the old kernel 7.0.0-31
apt-cache policy linux-modules-nvidia-595-open-$(uname -r) \
                 linux-modules-nvidia-595-open-generic-hwe-24.04
# → module for -38 available but not installed; meta-package stuck at -31
```

**Root cause:** Ubuntu installs the driver as *precompiled* kernel modules built for one exact kernel.
A meta-package (`linux-modules-nvidia-595-open-generic-hwe-24.04`) is meant to pull in the matching
module on kernel updates. `apt upgrade` never installs *new* packages, so it held the meta-package back:
the new kernel was installed without its NVIDIA module.

**Fix:**

```bash
sudo apt install linux-modules-nvidia-595-open-generic-hwe-24.04
sudo reboot
```

Installing the meta-package (not just the `-38` module) keeps future kernel updates tracked.
The driver was upgraded to 595.99.02; kernel module and user-space libraries must match exactly.

**Lesson:** use `sudo apt full-upgrade`, or read the "kept back" list after `apt upgrade`.

**Side effect:** the desktop now renders on the AMD iGPU (on-demand mode). The NVIDIA GPU is free
for compute (~15 MiB used at idle instead of ~560 MiB).

## 4. Base development tools

```bash
sudo apt install -y build-essential git cmake curl ffmpeg \
  libgl1 libglfw3 libglew-dev libosmesa6-dev
```

| Package | Purpose |
|---|---|
| `build-essential`, `cmake` | Compiling native Python extensions and ROS 2 packages |
| `ffmpeg` | LeRobot stores camera data as compressed video |
| `libgl1`, `libglfw3`, `libglew-dev` | OpenGL + windowing for the MuJoCo viewer |
| `libosmesa6-dev` | Off-screen rendering fallback |

## 5. Python tooling: uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Chosen for speed, built-in Python version management and lockfiles (`uv.lock`) for reproducibility.

## 6. Isolating ROS 2 from ML environments

ROS 2 was sourced in `~/.bashrc`, which put `/opt/ros/jazzy/...` on `PYTHONPATH` in every terminal.
`PYTHONPATH` takes priority over a virtual environment's packages, which can cause version clashes
(e.g. numpy) with PyTorch.

**Fix:** removed the `source /opt/ros/jazzy/setup.bash` line from `~/.bashrc`.
ROS is now sourced manually only in ROS terminals.

**Note:** existing shells keep their environment; changes to `.bashrc` only affect new terminals.

## 7. Git and GitHub

- `git config --global` for name, email, `init.defaultBranch main`
- SSH key (`ed25519`, passphrase-protected) for git push/pull; host fingerprint verified
- GitHub CLI (`gh`) with OAuth device login for API actions (creating repos, releases)
- Third-party repositories (mjlab, LeRobot) are cloned next to this repo and git-ignored;
  their exact commits will be recorded here for reproducibility.

## Design decisions

- **One environment per heavy framework** (mjlab, LeRobot): avoids conflicting PyTorch/Warp pins.
  Components communicate via ROS 2 topics and exported policies, not shared imports.
- **Python 3.12 everywhere:** matches ROS 2 Jazzy, so ROS nodes can later import LeRobot.
- **No Docker for now:** venvs already isolate Python; Docker would add GPU, GUI and DDS networking
  friction without solving a real conflict. A Dockerfile may be added at the end for reproducibility.

## Next

- Install mjlab (RL locomotion) and LeRobot + SmolVLA, and verify GPU access from PyTorch.

## 8. mjlab (RL framework)

```bash
git clone https://github.com/mujocolab/mjlab.git   # commit 45b1e263e (2026-10-12)
cd mjlab
uv sync --extra cu130
uv run --extra cu130 demo                           # viewer at http://localhost:8080
```

- **`--extra cu130`** selects the PyTorch build for CUDA 13.0, which contains `sm_120` kernels for Blackwell.
  The `cu126` extra predates Blackwell. Verified: `torch 2.14.0+cu130`, arch list includes `sm_120`.
- uv chose **Python 3.13** for this environment. Fine, because mjlab never imports ROS;
  trained policies leave it as exported files. The Python 3.12 rule applies to the LeRobot environment.
- **Problem:** the first `uv sync` failed with `network unreachable` / DNS errors on an unstable connection.
  Retried with `UV_CONCURRENT_DOWNLOADS=4` to reduce parallel downloads.
- First run JIT-compiles the MuJoCo Warp kernels for `sm_120` (~25 s), cached in `~/.cache/warp`.
