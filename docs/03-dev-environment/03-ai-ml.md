[Home](../../README.md) · [↑ Dev Environment](README.md) · [← Previous: 3.2 Embedded](02-embedded.md) · **3.3 AI/ML** · [Next: 3.4 Web →](04-web.md)

---

# 3.3 AI/ML: NVIDIA 580 + CUDA 13.2 + cuDNN 9.21 + PyTorch

Proprietary driver, matching CUDA toolkit, cuDNN 9 for CUDA 13, PyTorch with CUDA 13 wheels, Ollama + Open WebUI for local LLMs. All tuned for the RTX 3050 Mobile (4 GB VRAM) hybrid-GPU constraints.

> **Version baseline (verified April 2026, Kubuntu 24.04 LTS / Ubuntu 24.04.4):**
> - NVIDIA driver: **580.126.20** (production branch, noble-updates)
> - CUDA Toolkit: **13.2** (latest; CUDA 12.x lines still available for backward compat)
> - cuDNN: **9.21.0** (packaged as `cudnn9-cuda-13`)
> - PyTorch: **2.11+** (CUDA 13 wheels are the PyPI default; explicit index URL `https://download.pytorch.org/whl/cu130`)

## Why driver 580 (not 550 / not 555 / not nouveau)

- **580** is the current production branch on noble-updates (accepted Jan 2026, revised Feb 2026). `ubuntu-drivers devices` on a fresh Kubuntu 24.04.4 install recommends `nvidia-driver-580`. Required for CUDA 13.x.
- **-open variant** (`nvidia-driver-580-open`) uses NVIDIA's open-source kernel module. Supported on Turing and later (your RTX 3050 Mobile is Ampere → supported). Works; same user-space blobs. Recommend **proprietary** `nvidia-driver-580` for broadest compatibility with kernel updates, suspend/resume, and third-party tooling like `supergfxctl`.
- **Older branches (550, 555, 560, 570)** — still installable if you have a reason, but 580 supersedes all of them and is required for CUDA 13.x. No reason to pin older.
- **nouveau** (the open-source reverse-engineered driver) has no CUDA and poor power management for Ampere. Not an option for AI work.

## Step 1: Install the NVIDIA proprietary driver

```bash
sudo apt update
sudo ubuntu-drivers devices
# Expect output listing nvidia-driver-580 as recommended (may also show -open and -server variants).

sudo apt install -y nvidia-driver-580 libnvidia-gl-580 nvidia-settings nvidia-prime
sudo reboot
```

After reboot, verify the driver is live and the RTX 3050 Mobile is detected:

```bash
nvidia-smi
# Expect: a table with "NVIDIA GeForce RTX 3050 Laptop GPU", driver version 580.126.xx,
# CUDA Version 13.2 (driver-reported capability, not toolkit)

# Hybrid-GPU render-offload test:
prime-run glxinfo | grep "OpenGL renderer"
# Expect: "NVIDIA GeForce RTX 3050 Laptop GPU/PCIe/SSE2"

glxinfo | grep "OpenGL renderer"
# Expect: "AMD RENOIR (...)" — the iGPU. This proves dGPU idles unless asked.
```

If `nvidia-smi` reports "No devices found", switch to Hybrid mode with `supergfxctl -m Hybrid` and reboot (see [2.6](../02-setup/06-asus-hardware.md)).

## Step 2: CUDA Toolkit 13.2

Matches driver 580; required for PyTorch CUDA 13 wheels.

```bash
# Use NVIDIA's own apt repo (noble / ubuntu2404). Ubuntu's own CUDA packages are always behind.
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update

# Install the metapackage for CUDA 13.2 toolkit (no driver — we installed that in Step 1):
sudo apt install -y cuda-toolkit-13-2
rm cuda-keyring_1.1-1_all.deb

# Persist CUDA paths:
tee -a ~/.bashrc >/dev/null <<'EOF'

# CUDA 13.2
export PATH="/usr/local/cuda-13.2/bin${PATH:+:${PATH}}"
export LD_LIBRARY_PATH="/usr/local/cuda-13.2/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}"
EOF
source ~/.bashrc

nvcc --version
# Expect: Cuda compilation tools, release 13.2, V13.2.xxx
```

If you need CUDA 12.x side-by-side (e.g. for a legacy PyTorch pin or an old TensorFlow build), install `cuda-toolkit-12-8` alongside; each lands in its own `/usr/local/cuda-<ver>` directory and you switch via `PATH` / `LD_LIBRARY_PATH`.

## Step 3: cuDNN 9.21 for CUDA 13

```bash
sudo apt install -y cudnn9-cuda-13
```

The metapackage pulls in `libcudnn9-cuda-13` (runtime) and `libcudnn9-dev-cuda-13` (headers). Only one cuDNN CUDA major can be installed at a time; if you previously installed `cudnn9-cuda-12`, apt will offer to replace it.

Verify:

```bash
dpkg -l | grep cudnn
# Expect: libcudnn9-cuda-13 9.21.x, libcudnn9-dev-cuda-13 9.21.x
```

## Step 4: `uv` as your Python manager

Kills `pip`, `venv`, `poetry`, `pyenv` — one static Rust binary handles all of it.

```bash
# Install uv (Rust-based, single binary).
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.bashrc

uv --version
```

### Create the AI workspace

```bash
# Python 3.12 is PyTorch 2.11's supported release (3.13 also works as of PyTorch 2.11).
mkdir -p ~/work/ai && cd ~/work/ai
uv venv --python 3.12
source .venv/bin/activate

# PyTorch with CUDA 13 wheels. As of PyTorch 2.11, CUDA 13 is the PyPI default —
# so `uv pip install torch torchvision torchaudio` without an index URL works.
# Explicit URL is clearer and future-proof:
uv pip install \
  torch torchvision torchaudio \
  --index-url https://download.pytorch.org/whl/cu130

# Core ML stack.
uv pip install \
  numpy scipy pandas matplotlib scikit-learn \
  jupyterlab ipykernel ipywidgets \
  transformers datasets accelerate peft trl bitsandbytes \
  opencv-python pillow \
  mlflow wandb tensorboard
```

### Verify CUDA is visible to PyTorch

```bash
python -c "import torch; print('CUDA:', torch.cuda.is_available(), '| Device:', torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'none'); print('cuDNN:', torch.backends.cudnn.version())"
# Expect: CUDA: True | Device: NVIDIA GeForce RTX 3050 Laptop GPU | cuDNN: 9xxxx
```

If that prints `CUDA: False`, the most common causes are: (1) wrong wheel — re-run with `--index-url https://download.pytorch.org/whl/cu130`, not the default PyPI (though on PyTorch 2.11 the default is cu130 anyway); (2) dGPU is off — check `supergfxctl -g` shows `Hybrid`; (3) driver / toolkit mismatch — run `nvidia-smi` and confirm the reported CUDA Version is ≥13.0.

## Hybrid-GPU UX on the TUF A17

The RTX 3050 Mobile lives in a hybrid setup with the Ryzen 4800H's iGPU. Day-to-day usage patterns:

| Scenario | Mode | Launch pattern |
|----------|------|----------------|
| Default desktop | `supergfxctl -m Hybrid` | Apps use iGPU; dGPU idles at 0 W |
| CUDA training / inference | `Hybrid` | `prime-run python train.py` or `__NV_PRIME_RENDER_OFFLOAD=1 python train.py` |
| Gazebo / RViz / heavy OpenGL | `Hybrid` | `prime-run gz sim ...`, `prime-run rviz2` |
| On-battery travel mode | `supergfxctl -m Integrated` (logout required) | Apps use iGPU only; NVIDIA driver disabled; 1.5–2x battery life |
| Max GPU for games | `supergfxctl -m AsusMuxDgpu` (reboot required) | All rendering on dGPU; battery life ~halved |
| VFIO passthrough to Windows VM | `supergfxctl -m Vfio` (reboot required) | dGPU bound to vfio-pci |

**Practical rule of thumb:**

- Plugged in, coding: `Hybrid` + `asusctl profile -P Balanced`.
- Plugged in, training: `Hybrid` + `asusctl profile -P Performance`, `prime-run` your Python.
- On battery, lecture / travel: `Integrated` + `Quiet` profile + `asusctl -c 80`.

## Step 5: Local LLM stack (M.Tech experimentation, offline Copilot)

Ollama — easiest local inference server, has a model library, CPU+GPU automatic.

```bash
curl -fsSL https://ollama.com/install.sh | sh
sudo systemctl enable --now ollama
ollama --version
```

Models that fit 4 GB of VRAM on the RTX 3050 Mobile (quantized to Q4_K_M by default):

```bash
ollama pull qwen2.5-coder:3b        # good coding assistant in ~2 GB
ollama pull qwen3:4b                # newer Qwen 3 family; fits 4 GB
ollama pull llama3.2:3b             # general chat
ollama pull nomic-embed-text        # embeddings for RAG (stays on CPU)
ollama pull deepseek-r1:1.5b        # reasoning model, tiny, CPU-friendly
```

Larger models (7B/8B/13B) will run but **spill to CPU**, cutting tokens/sec to single digits. Fine for overnight jobs, frustrating interactively.

### Open WebUI (ChatGPT-like front-end) via Podman

```bash
podman run -d --name openwebui \
  --network=host \
  -v ~/.config/openwebui:/app/backend/data \
  -e OLLAMA_BASE_URL=http://127.0.0.1:11434 \
  --restart unless-stopped \
  ghcr.io/open-webui/open-webui:main
# Browse to http://localhost:8080
```

For a long-running setup, convert this to a Quadlet systemd service — see [3.1 Containers](01-containers.md).

### VS Code → Ollama integration

Install the `continue.continue` extension. `~/.continue/config.json`:

```json
{
  "models": [
    {
      "title": "Qwen 2.5 Coder 3B (local)",
      "provider": "ollama",
      "model": "qwen2.5-coder:3b"
    }
  ],
  "tabAutocompleteModel": {
    "title": "Qwen 2.5 Coder 3B",
    "provider": "ollama",
    "model": "qwen2.5-coder:3b"
  }
}
```

Local autocomplete, free, works offline. Noticeably slower than GitHub Copilot (human typing speed vs 500ms); noticeably private.

## JupyterLab as a separate workspace

JupyterLab is already installed in the AI venv. Launch:

```bash
cd ~/work/ai
source .venv/bin/activate
prime-run jupyter lab
# --no-browser + --port 8889 if you want to SSH-tunnel from another machine
```

For a cleaner experience, install `marimo` instead (reactive, git-friendly, no `.ipynb` hell):

```bash
uv pip install marimo
marimo new my_notebook.py
```

## Verification summary

```bash
nvidia-smi                           # dGPU detected, driver 580.xxx, CUDA Version 13.x
nvcc --version                       # CUDA 13.2
dpkg -l | grep cudnn                 # cudnn9-cuda-13 9.21.x
uv --version                         # uv 0.x
source ~/work/ai/.venv/bin/activate
python -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0), torch.version.cuda)"
# Expect: True, NVIDIA GeForce RTX 3050 Laptop GPU, 13.0 (or 13.x)
ollama list                          # pulled models shown
curl -s localhost:8080 | head -5     # Open WebUI HTML
```

Proceed to [3.4 Next.js blog workflow](04-web.md).

---

[Home](../../README.md) · [↑ Dev Environment](README.md) · [← Previous: 3.2 Embedded](02-embedded.md) · **3.3 AI/ML** · [Next: 3.4 Web →](04-web.md)
