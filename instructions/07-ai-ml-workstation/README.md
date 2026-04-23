[Home](../README.md) · [← Previous: 6.4 Native Lyrical](../06-ros2-robotics/04-native-lyrical-after-may-22.md) · **07 — AI / ML Workstation** · [Next: 7.1 CUDA & cuDNN →](01-cuda-13-cudnn-native.md)

---

# 07 — AI / ML Workstation

NVIDIA 580 + native CUDA 13 + cuDNN 9 + PyTorch/JAX + Ollama + Open WebUI + Continue.dev, tuned for the RTX 3050 Mobile's 4 GB VRAM budget.

## Steps

| #   | Step                                                                           | Time    | Core?                                     |
| ---:| ------------------------------------------------------------------------------ | ------- | ----------------------------------------- |
| 7.1 | [CUDA 13 + cuDNN native install on 26.04](01-cuda-13-cudnn-native.md)          | 10 min  | Yes                                       |
| 7.2 | [Python via `uv`, pinned 3.12 venv, PyTorch + JAX](02-python-uv-pytorch-jax.md)| 15 min  | Yes                                       |
| 7.3 | [Local LLMs: Ollama + Open WebUI Quadlet + Continue.dev, 4 GB VRAM model picks](03-local-llms-ollama-openwebui.md) | 25 min  | Yes                                       |

Total: ~50 minutes.

## Prerequisites you must have done

- [2.2 NVIDIA driver 580 + `nvidia-cuda-toolkit`](../02-post-install-foundations/02-nvidia-driver-580-and-cuda.md) — driver and CUDA toolkit installed, `nvidia-smi` works.
- [2.3 ASUS TUF `supergfxctl`](../02-post-install-foundations/03-asus-tuf-hybrid-gpu.md) — you are in `Hybrid` mode, not `Integrated`.
- [5.1 Zsh + Starship](../05-web-development/01-shell-and-terminal.md) — `.zshrc` is where CUDA paths are persisted.

## What the 4 GB VRAM budget means in practice

| Model class                   | Fits on GPU?                | Typical speed                | Usefulness                                |
| ----------------------------- | --------------------------- | ---------------------------- | ----------------------------------------- |
| 1.5 B (e.g. `deepseek-r1:1.5b`)| Easily, Q4_K_M ~1 GB        | 40–70 tok/s                  | Tiny classification, reasoning demos      |
| 3 B (e.g. `qwen2.5-coder:3b`) | Easily, Q4_K_M ~2 GB        | 30–50 tok/s                  | **Primary local coding assistant**         |
| 4 B (e.g. `qwen3:4b`, `gemma3:4b`) | Yes, Q4_K_M ~2.5 GB    | 25–40 tok/s                  | General chat, writing, light RAG          |
| 7 B                           | Tight, Q4_K_M ~4.2 GB — spills to CPU | 5–10 tok/s        | Overnight jobs; interactively painful     |
| 8 B (e.g. `llama3.1:8b`)      | No, spills heavily          | 3–6 tok/s                    | CPU-only, leave for batch                 |
| 13 B+                         | No                          | 1–3 tok/s                    | Forget it                                 |

**Practical rule:** for an on-laptop "local Copilot" experience, use 3B–4B models. Anything 7B+ goes to API or to a bigger GPU.

## Fine-tuning / LoRA on 4 GB VRAM

It is possible but constrained. [7.3](03-local-llms-ollama-openwebui.md) covers:

- **Unsloth** + **LoRA** + **4-bit quantization** is the only viable path. A 3 B base + LoRA adapter at rank 16 fits with `bitsandbytes` 4-bit loading.
- Gradient checkpointing + `fp16` mixed precision mandatory.
- Batch size 1, gradient accumulation 16+.
- Expect a LoRA epoch on ~10k samples of 3B base to take 4–8 hours.
- For anything bigger, [rent an H100 on Paperspace / Lambda / Runpod for $2/hr](03-local-llms-ollama-openwebui.md#renting-cloud-gpus-when-4-gb-is-not-enough) — cheaper than fighting the 4 GB limit.

---

[Home](../README.md) · [← Previous: 6.4 Native Lyrical](../06-ros2-robotics/04-native-lyrical-after-may-22.md) · **07 — AI / ML Workstation** · [Next: 7.1 CUDA & cuDNN →](01-cuda-13-cudnn-native.md)
