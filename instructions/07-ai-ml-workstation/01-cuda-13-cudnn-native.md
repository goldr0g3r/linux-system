[Home](../README.md) · [↑ 07 AI/ML](README.md) · [← Previous: 07 AI/ML (section)](README.md) · **7.1 CUDA & cuDNN** · [Next: 7.2 Python & PyTorch →](02-python-uv-pytorch-jax.md)

---

# 7.1 CUDA 13 + cuDNN Native Install on Kubuntu 26.04

This page is a focused recap of the CUDA install from [2.2](../02-post-install-foundations/02-nvidia-driver-580-and-cuda.md) plus the extras needed for AI/ML (samples verification, optional CUDA 12.x alongside for legacy PyTorch pins, cross-linking to the AI workspace).

## What should already be in place from section 02

- NVIDIA driver 580 (proprietary) installed via `ubuntu-drivers install`. Verified by `nvidia-smi`.
- CUDA toolkit 13.2 installed via the native Ubuntu `nvidia-cuda-toolkit` package. Verified by `nvcc --version`.
- cuDNN 9.21 installed via `cudnn9-cuda-13`.
- `supergfxctl -g` reports `Hybrid` (so the dGPU is available for CUDA).

If any of that is missing, return to [2.2](../02-post-install-foundations/02-nvidia-driver-580-and-cuda.md) first.

## 7.1.1 Confirm the stack

```bash
# Driver:
nvidia-smi | grep "Driver Version"
# Expect: Driver Version: 580.x

# CUDA runtime reported by driver:
nvidia-smi | grep "CUDA Version"
# Expect: CUDA Version: 13.2

# CUDA toolkit (nvcc):
nvcc --version | grep release
# Expect: release 13.2

# cuDNN:
dpkg -l | grep -E "cudnn9-cuda-13|libcudnn9-cuda-13"
# Expect: libcudnn9-cuda-13 9.21.x, libcudnn9-dev-cuda-13 9.21.x

# GPU mode:
supergfxctl -g
# Expect: Hybrid (not Integrated)
```

If all four check out, skip to [7.2](02-python-uv-pytorch-jax.md).

## 7.1.2 CUDA samples — a full CUDA smoke test

```bash
# Install:
sudo apt install -y nvidia-cuda-samples

# The samples live in /usr/share/nvidia-cuda-samples/samples/:
ls /usr/share/nvidia-cuda-samples/samples/
# You'll see classic directories like 1_Utilities, 2_Concepts_and_Techniques, etc.

# Build and run deviceQuery:
cd /tmp
cp -a /usr/share/nvidia-cuda-samples/samples/1_Utilities/deviceQuery .
cd deviceQuery
make
./deviceQuery
```

Expected output tail:

```
deviceQuery, CUDA Driver = CUDART, CUDA Driver Version = 13.2, CUDA Runtime Version = 13.2, NumDevs = 1
Result = PASS
```

Also run bandwidthTest (verifies PCIe throughput to the dGPU):

```bash
cd /tmp
cp -a /usr/share/nvidia-cuda-samples/samples/1_Utilities/bandwidthTest .
cd bandwidthTest
make
./bandwidthTest
# Expect:
# Host to Device Bandwidth: ~12-13 GB/s (PCIe 3.0 x8 for the 3050 Mobile)
# Device to Host: ~12-13 GB/s
# Device to Device: ~100-150 GB/s (VRAM to VRAM)
```

If bandwidth is substantially lower, the dGPU might be in a low-power state or PCIe link is degraded. Check `nvidia-smi -q | grep -i "link\|generation"`.

## 7.1.3 Add CUDA 12.x alongside (optional, for legacy PyTorch / TensorFlow pins)

PyTorch 2.11 default wheels target CUDA 13 (cu130). If you pin an older PyTorch (2.3 for a reproducibility study, say), it wants CUDA 12.x. Install 12.8 side-by-side using the NVIDIA APT repo:

```bash
# This does NOT replace the native nvidia-cuda-toolkit. Both coexist.

wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2604/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update

# Install just the toolkit, not the driver:
sudo apt install -y cuda-toolkit-12-8

rm cuda-keyring_1.1-1_all.deb
```

Now you have:

- `/usr/bin/nvcc` → CUDA 13.2 (native, default `PATH`).
- `/usr/local/cuda-12.8/bin/nvcc` → CUDA 12.8 (side-loaded).

Switch at runtime per-project:

```bash
# In a legacy project's venv-activate or a .env.sh:
export PATH="/usr/local/cuda-12.8/bin:$PATH"
export LD_LIBRARY_PATH="/usr/local/cuda-12.8/lib64:${LD_LIBRARY_PATH:-}"
export CUDA_HOME="/usr/local/cuda-12.8"
```

When you exit the shell / terminal, PATH reverts to system default (CUDA 13).

### Versions to consider pinning vs not

| CUDA version  | Use                                              | Why                            |
| ------------- | ------------------------------------------------ | ------------------------------ |
| **13.2** (native) | Default for new projects, current PyTorch     | Single-command install         |
| 12.8          | PyTorch 2.3-2.7 wheels, old TensorFlow SDKs      | Install only if pinned         |
| 11.8          | Very old code (< 2023)                            | Only if you have to            |

## 7.1.4 Environment for the AI workspace

Set defaults for your `~/work/ai/` workspace. Add to `~/.zshrc` (or a project-specific `.envrc` with `direnv`):

```bash
cat >> ~/.zshrc <<'EOF'

# ----- CUDA (native) -----
# nvidia-cuda-toolkit ships nvcc in /usr/bin, so no PATH changes needed for default.
# If you need to set CUDA_HOME for packages that expect it:
export CUDA_HOME=/usr
export CUDA_VISIBLE_DEVICES=0   # only one GPU; 0 is the RTX 3050

# Force PyTorch to skip JIT warm-up on first run (speeds up imports):
# export TORCH_CUDNN_V8_API_DISABLED=0

# cuDNN benchmark mode (picks fastest algorithm per input shape):
# enable inside Python via torch.backends.cudnn.benchmark = True
EOF
```

## 7.1.5 Monitoring utilities

```bash
# nvtop — beautiful interactive GPU monitoring:
sudo apt install -y nvtop
nvtop
# Press Q to quit.

# nvidia-ml-py (Python binding for low-level monitoring):
uv tool install nvitop
nvitop
# Alternative with more features.

# glances with GPU support:
uv tool install "glances[gpu]"
glances
```

For long-running jobs, `nvtop` in a Konsole tab is invaluable — you can see VRAM usage, GPU utilisation, and per-process allocation in real-time.

## 7.1.6 GPU-aware Ollama (optional preview, full setup in 7.3)

Quick sanity check that Ollama can use the GPU:

```bash
# One-line install:
curl -fsSL https://ollama.com/install.sh | sh

# Pull a small model:
ollama pull qwen2.5-coder:3b

# Run it:
ollama run qwen2.5-coder:3b 'write a haiku about CUDA'

# In a second terminal, while it's generating:
nvidia-smi
# Should show the ollama process on the GPU, ~2 GB VRAM used.
```

If Ollama is running on CPU only (no GPU in `nvidia-smi`'s process list), check:

```bash
ollama serve --verbose 2>&1 | grep -i "cuda\|gpu"
# Expect messages about detecting CUDA and loading the CUDA backend.
```

Full Ollama setup in [7.3](03-local-llms-ollama-openwebui.md).

## 7.1.7 NVIDIA persistence mode

Short-running CUDA apps have a driver warm-up cost (~1-2 s on first invocation after idle). Persistence mode keeps the driver initialised:

```bash
# Enable persistence (non-volatile until reboot):
sudo nvidia-smi -pm 1

# Verify:
sudo nvidia-smi -q | grep "Persistence Mode"
# Expect: Enabled.

# To make persistent across reboots, a systemd service already exists:
sudo systemctl enable --now nvidia-persistenced.service
```

This trades a few extra idle watts for faster `python train.py` / Ollama query response. Worth it on a dev laptop that's frequently plugged in.

## 7.1.8 Test CUDA compile end-to-end

A minimal program to prove the compile toolchain works:

```bash
cat > /tmp/vadd.cu <<'EOF'
#include <cstdio>
#include <cuda_runtime.h>

__global__ void vectorAdd(const float* a, const float* b, float* c, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n) c[i] = a[i] + b[i];
}

int main() {
    const int N = 1 << 20;
    size_t bytes = N * sizeof(float);
    float *a, *b, *c;
    cudaMallocManaged(&a, bytes);
    cudaMallocManaged(&b, bytes);
    cudaMallocManaged(&c, bytes);
    for (int i = 0; i < N; ++i) { a[i] = 1.0f; b[i] = 2.0f; }
    int threads = 256, blocks = (N + threads - 1) / threads;
    vectorAdd<<<blocks, threads>>>(a, b, c, N);
    cudaDeviceSynchronize();
    printf("c[0]=%f  c[N-1]=%f\n", c[0], c[N-1]);
    cudaFree(a); cudaFree(b); cudaFree(c);
    return 0;
}
EOF

cd /tmp
nvcc vadd.cu -o vadd
./vadd
# Expect: c[0]=3.000000  c[N-1]=3.000000
```

If that works, your CUDA toolchain is fully functional.

## 7.1.9 Snapshot

```bash
sudo timeshift --create --comments "post-cuda-verification $(date -I)" --tags D
```

Proceed to [7.2 Python via uv, PyTorch, JAX](02-python-uv-pytorch-jax.md).

---

[Home](../README.md) · [↑ 07 AI/ML](README.md) · [← Previous: 07 AI/ML (section)](README.md) · **7.1 CUDA & cuDNN** · [Next: 7.2 Python & PyTorch →](02-python-uv-pytorch-jax.md)
