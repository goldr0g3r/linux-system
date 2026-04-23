[Home](../README.md) · [↑ 07 AI/ML](README.md) · [← Previous: 7.1 CUDA & cuDNN](01-cuda-13-cudnn-native.md) · **7.2 Python, uv, PyTorch, JAX** · [Next: 7.3 Local LLMs →](03-local-llms-ollama-openwebui.md)

---

# 7.2 Python via `uv`, Pinned 3.12 venv, PyTorch + JAX

`uv` is the Rust-based replacement for `pip + venv + poetry + pyenv + pipx`, all in one binary. It's ~10-100x faster than pip and handles everything from "install one CLI" to "manage a monorepo of 30 packages." On 26.04, it is the correct default.

Python 3.14 is the host default on Kubuntu 26.04. But PyTorch / scipy / opencv-python wheels for 3.14 lag ~2-4 weeks behind the CPython release (as of 2026-04-23, torch 2.11 has 3.14 wheels in nightly, stable in ~2 weeks). **We pin AI workspaces to Python 3.12** until 3.14 wheels mature.

## 7.2.1 Install uv

```bash
# Official installer:
curl -LsSf https://astral.sh/uv/install.sh | sh

# Add to PATH in ~/.zshrc if the installer didn't:
export PATH="$HOME/.local/bin:$PATH"

# Verify:
uv --version
# Expect: uv 0.5.x or higher.
```

## 7.2.2 Install Python 3.12 via uv (alongside the host's 3.14)

```bash
uv python install 3.12

# List installed versions:
uv python list

# Host's system Python (3.14) is always available too:
python3.14 --version
# Expect: Python 3.14.x
```

`uv` installs its Pythons to `~/.local/share/uv/python/` — they do not touch `/usr/bin/python3` and do not interfere with the system.

## 7.2.3 Create the AI workspace with a pinned 3.12 venv

```bash
mkdir -p ~/work/ai && cd ~/work/ai

# Initialise a uv project with pinned Python 3.12:
uv init --python 3.12 --name ai-experiments

# uv created: pyproject.toml, .python-version, a .venv, a hello.py stub.

# Activate the venv:
source .venv/bin/activate

# Verify:
python --version
# Expect: Python 3.12.x

which python
# Expect: /home/<user>/work/ai/.venv/bin/python
```

## 7.2.4 Install PyTorch with CUDA 13 wheels

```bash
# Using uv (from inside the activated venv or via `uv pip` without activation):
uv pip install torch torchvision torchaudio \
  --index-url https://download.pytorch.org/whl/cu130

# Verify:
python -c "
import torch
print('torch version:', torch.__version__)
print('CUDA available:', torch.cuda.is_available())
print('Device:', torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'none')
print('CUDA version:', torch.version.cuda)
print('cuDNN version:', torch.backends.cudnn.version())
"
```

Expect:

```
torch version: 2.11.x+cu130
CUDA available: True
Device: NVIDIA GeForce RTX 3050 Laptop GPU
CUDA version: 13.0
cuDNN version: 9xxxx
```

If `CUDA available: False`, verify:

- `nvidia-smi` works outside Python — driver is live.
- `torch.version.cuda` is set (not None) — PyTorch was compiled with CUDA.
- `supergfxctl -g` reports `Hybrid` — dGPU is available.

## 7.2.5 The core ML stack

```bash
uv pip install \
  numpy scipy pandas matplotlib scikit-learn \
  jupyterlab ipykernel ipywidgets \
  transformers datasets accelerate peft trl bitsandbytes \
  opencv-python pillow \
  mlflow wandb tensorboard \
  pyarrow polars \
  rich typer
```

Package rationale:

- **numpy, scipy, pandas, matplotlib, scikit-learn** — classical ML + data manipulation.
- **jupyterlab, ipykernel, ipywidgets** — notebooks.
- **transformers, datasets, accelerate, peft, trl, bitsandbytes** — the Hugging Face stack for LLM inference, fine-tuning, LoRA.
- **opencv-python, pillow** — image processing.
- **mlflow, wandb, tensorboard** — experiment tracking. Pick one for a given project; I use `wandb` when it's a hosted run and `tensorboard` when offline.
- **pyarrow, polars** — modern DataFrame tooling, ~10x faster than pandas on large datasets.
- **rich, typer** — CLI formatting and argument parsing.

## 7.2.6 JAX with CUDA 13

JAX is Google's functional-style tensor library; great for research-y code with vectorised transformations (`vmap`, `pmap`). Optional but install if you do any scientific computing beyond PyTorch.

```bash
uv pip install \
  "jax[cuda13_local]" \
  --index-url https://storage.googleapis.com/jax-releases/jax_cuda_releases.html

# Verify:
python -c "
import jax
import jax.numpy as jnp
print('JAX devices:', jax.devices())
x = jnp.arange(10)
print('JAX on CUDA:', jax.device_put(x).device())
"
```

Expect to see a `cuda:0` device.

## 7.2.7 JupyterLab as a separate workspace

JupyterLab is already installed. Launch:

```bash
cd ~/work/ai
source .venv/bin/activate
jupyter lab
# Opens Firefox/default browser at http://localhost:8888 with a token.
```

For GPU-bound work, launch via `prime-run` to nudge it onto the dGPU (optional — Python processes don't actually use the GPU until they `torch.tensor(...).cuda()`; then CUDA context opens regardless of prime-run):

```bash
prime-run jupyter lab
```

### Marimo — reactive notebook alternative

[Marimo](https://marimo.io) is a modern alternative to Jupyter: reactive (cells re-run when dependencies change), git-friendly (stored as Python files), no `.ipynb` merge conflicts.

```bash
uv pip install marimo
marimo new my_experiment.py
```

Try it on a fresh project; it's especially good for exploratory data analysis.

## 7.2.8 Multiple AI projects — per-project venv pattern

For each new AI project:

```bash
mkdir ~/work/ai/new-project && cd ~/work/ai/new-project
uv init --python 3.12 --name new-project

# pyproject.toml → add your deps
uv add torch --extra-index-url https://download.pytorch.org/whl/cu130
uv add transformers datasets accelerate

uv sync
```

`uv sync` installs exactly what's in `pyproject.toml` + `uv.lock`, reproducible across machines. Commit `pyproject.toml` and `uv.lock`, gitignore `.venv/`.

## 7.2.9 Inline script dependencies (PEP 723)

For short scripts that don't warrant a full project, use `uv`'s inline script metadata:

```python
# /// script
# requires-python = ">=3.12"
# dependencies = [
#   "torch>=2.11",
#   "transformers",
# ]
# ///

import torch
from transformers import pipeline

pipe = pipeline("sentiment-analysis")
print(pipe("Kubuntu 26.04 is pretty great."))
```

Run without a venv:

```bash
uv run sentiment.py
# uv creates an ephemeral venv, installs deps, runs, exits. Caches for repeat runs.
```

## 7.2.10 CLI tools install pattern

For Python CLI tools (like `yt-dlp`, `httpie`, `pgcli`):

```bash
uv tool install yt-dlp
uv tool install httpie
uv tool install pgcli

# List installed tools:
uv tool list

# Update:
uv tool upgrade --all

# Uninstall:
uv tool uninstall httpie
```

Each tool lives in its own isolated venv under `~/.local/share/uv/tools/`. No pollution of your project venvs.

## 7.2.11 Verify CUDA + cuDNN benchmark

Quick speed test:

```python
cat > /tmp/cuda_bench.py <<'EOF'
# /// script
# requires-python = ">=3.12"
# dependencies = ["torch>=2.11"]
# ///

import time
import torch

device = "cuda" if torch.cuda.is_available() else "cpu"
print(f"Device: {device}")

# 4k x 4k matrix multiply, 100 iterations:
N = 4096
a = torch.randn(N, N, device=device)
b = torch.randn(N, N, device=device)

# Warmup:
for _ in range(3):
    torch.matmul(a, b)
if device == "cuda":
    torch.cuda.synchronize()

t0 = time.time()
for _ in range(100):
    c = torch.matmul(a, b)
if device == "cuda":
    torch.cuda.synchronize()
dt = time.time() - t0

tflops = 100 * 2 * N**3 / dt / 1e12
print(f"Time: {dt:.2f}s, TFLOPS: {tflops:.1f}")
EOF

uv run /tmp/cuda_bench.py
```

Expected on RTX 3050 Mobile 4 GB: ~6-8 TFLOPS (mixed precision), ~3-4 TFLOPS (FP32). The RTX 3050 Mobile is a ~2x-on-paper relative to an 8-core Ryzen; in practice much more, because matmuls are one of CUDA's sweet spots.

## 7.2.12 When you encounter "Python 3.14 wheels not yet available"

Some package says "no wheel for 3.14, building from source" and takes 20 minutes or fails:

```bash
# Pin the venv to 3.12 instead:
rm -rf .venv
uv venv --python 3.12
source .venv/bin/activate
uv pip install <the-package>
```

Or for a single-run script:

```bash
uv run --python 3.12 my_script.py
```

By mid-2026, most wheels will be on 3.14. Until then, 3.12 is the pragmatic default.

## 7.2.13 Snapshot

```bash
sudo timeshift --create --comments "post-python-pytorch $(date -I)" --tags D
```

Proceed to [7.3 Local LLMs — Ollama + Open WebUI + Continue.dev](03-local-llms-ollama-openwebui.md).

---

[Home](../README.md) · [↑ 07 AI/ML](README.md) · [← Previous: 7.1 CUDA & cuDNN](01-cuda-13-cudnn-native.md) · **7.2 Python, uv, PyTorch, JAX** · [Next: 7.3 Local LLMs →](03-local-llms-ollama-openwebui.md)
