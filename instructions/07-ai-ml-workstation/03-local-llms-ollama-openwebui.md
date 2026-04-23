[Home](../README.md) · [↑ 07 AI/ML](README.md) · [← Previous: 7.2 Python & PyTorch](02-python-uv-pytorch-jax.md) · **7.3 Local LLMs** · [Next: 08 Productivity →](../08-productivity-security/README.md)

---

# 7.3 Local LLMs — Ollama + Open WebUI + Continue.dev, 4 GB VRAM Picks

Running local LLM inference on the RTX 3050 Mobile's 4 GB VRAM, wired into your daily tools: a ChatGPT-like WebUI (Open WebUI), and VS Code / Cursor autocompletion (Continue.dev). Plus a recipe for LoRA fine-tuning with Unsloth when 4 GB is almost enough.

## The stack

| Layer            | Tool                 | Role                                                  |
| ---------------- | -------------------- | ----------------------------------------------------- |
| Inference engine | **Ollama**           | One-command pull + run quantized models               |
| Web UI           | **Open WebUI**       | ChatGPT-like interface, via Podman Quadlet            |
| IDE integration  | **Continue.dev**     | VS Code / Cursor autocomplete → local Ollama          |
| Fine-tuning      | **Unsloth + LoRA**   | 4-bit LoRA adapters, fits in 4 GB VRAM                |

## 7.3.1 Install Ollama

```bash
curl -fsSL https://ollama.com/install.sh | sh

# Enable the service:
sudo systemctl enable --now ollama

# Verify:
ollama --version
ollama list
# Initially empty; no models pulled.

# GPU detection:
journalctl -u ollama -n 20 | grep -i "cuda\|gpu"
# Expect log lines about detecting CUDA, RTX 3050, 4 GB VRAM.
```

If Ollama fails to detect the GPU, check:

- `nvidia-smi` works as your user.
- `ls -la /dev/nvidia*` shows `rw` for `video` group (Ollama runs as the `ollama` user, which must be in `video`).
- Restart: `sudo systemctl restart ollama`.

## 7.3.2 Pull the recommended model set for 4 GB VRAM

```bash
# Primary coding assistant (~2 GB, GPU-resident, 30-50 tok/s):
ollama pull qwen2.5-coder:3b

# General chat (~2 GB, GPU-resident):
ollama pull llama3.2:3b

# Newer Qwen 3 family, small but strong (~2.5 GB):
ollama pull qwen3:4b

# Reasoning model — tiny, runs CPU-friendly:
ollama pull deepseek-r1:1.5b

# Embeddings model — for RAG, CPU inference, small:
ollama pull nomic-embed-text

# Gemma 3 4B (if you want Google's small model):
ollama pull gemma3:4b

# List:
ollama list
```

Approximate disk footprint: ~15 GB total for all six models. Pull only what you'll use.

## 7.3.3 First interactive chat

```bash
ollama run qwen2.5-coder:3b

>>> write a Python function to reverse a linked list
```

First invocation may take 5-10 s (model loading into VRAM); subsequent responses stream at 30-50 tok/s.

Ctrl+D or `/bye` to exit.

## 7.3.4 Install Open WebUI via Podman Quadlet

Open WebUI is a ChatGPT-style web interface that talks to Ollama. Run as a user-level Quadlet so it starts with your session:

```bash
mkdir -p ~/.config/containers/systemd ~/Containers/openwebui

cat > ~/.config/containers/systemd/openwebui.container <<'EOF'
[Unit]
Description=Open WebUI (Ollama frontend)
After=network.target

[Container]
Image=ghcr.io/open-webui/open-webui:main
ContainerName=openwebui
PublishPort=3000:8080
Volume=%h/Containers/openwebui:/app/backend/data:Z
Environment=OLLAMA_BASE_URL=http://127.0.0.1:11434
Environment=WEBUI_AUTH=False
Network=host

[Service]
Restart=on-failure
TimeoutStartSec=120

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user start openwebui.service

# Takes ~30 s on first start to initialise SQLite + download assets.

# Verify:
curl -s http://localhost:3000 | head -5
# Expect: HTML (the WebUI landing page).
```

Open `http://localhost:3000` in Firefox. The Open WebUI interface:

- Left sidebar: your Ollama models.
- Main pane: chat interface.
- Settings (top right): change default model, system prompts, etc.

The `WEBUI_AUTH=False` disables the login page since this is a single-user local deployment. If you'll access it from another device on your LAN (phone, second laptop), change to `WEBUI_AUTH=True` and set up a user on first visit.

### Useful Open WebUI features

- **Knowledge bases**: upload PDFs / text files, WebUI chunks and embeds them via `nomic-embed-text`, you chat grounded in your docs.
- **Prompts**: save reusable prompt templates.
- **Chat history**: SQLite-backed, survives container restart.
- **Workspaces**: separate chat histories for different contexts.
- **Voice**: TTS + STT, CPU-inference models bundled.

## 7.3.5 Continue.dev — VS Code integration

Install the extension (done in [5.4.1](../05-web-development/04-editors-and-ides.md#core-extensions)):

```bash
code --install-extension continue.continue
```

Configure `~/.continue/config.json`:

```json
{
  "models": [
    {
      "title": "Qwen 2.5 Coder 3B (local)",
      "provider": "ollama",
      "model": "qwen2.5-coder:3b",
      "apiBase": "http://localhost:11434"
    },
    {
      "title": "Llama 3.2 3B (local)",
      "provider": "ollama",
      "model": "llama3.2:3b"
    },
    {
      "title": "DeepSeek R1 1.5B (local, reasoning)",
      "provider": "ollama",
      "model": "deepseek-r1:1.5b"
    }
  ],
  "tabAutocompleteModel": {
    "title": "Qwen 2.5 Coder 3B",
    "provider": "ollama",
    "model": "qwen2.5-coder:3b",
    "apiBase": "http://localhost:11434"
  },
  "embeddingsProvider": {
    "provider": "ollama",
    "model": "nomic-embed-text"
  },
  "slashCommands": [
    { "name": "edit", "description": "Edit selected code" },
    { "name": "comment", "description": "Write comments for selected code" },
    { "name": "share", "description": "Share this session" },
    { "name": "cmd", "description": "Generate a shell command" }
  ],
  "contextProviders": [
    { "name": "code" },
    { "name": "docs" },
    { "name": "diff" },
    { "name": "open" },
    { "name": "terminal" },
    { "name": "problems" },
    { "name": "folder" },
    { "name": "codebase" }
  ]
}
```

In VS Code, `Ctrl+L` opens the Continue chat sidebar. Tab autocomplete is enabled by default (there's no keybinding; it suggests as you type).

### Latency budget

Tab autocompletion on Qwen 2.5 Coder 3B takes ~500 ms to first token, then ~30 tok/s. Noticeably slower than GitHub Copilot (which is cloud, ~100 ms). Trade-offs:

- **Local**: private, free, offline-capable, ~500 ms.
- **Cloud (Copilot, Cursor)**: private-to-vendor, $10-20/mo, online, ~100 ms.

I use both. Continue.dev for most files, Copilot/Cursor when I need faster-than-thought autocompletion.

## 7.3.6 Cursor + Claude integration

Covered briefly in [5.4.2](../05-web-development/04-editors-and-ides.md#542-cursor-ai-first). Cursor's AI works best with subscribed-tier cloud models (Claude, GPT). For local models, Cursor can also point at Ollama via Settings → Models → Ollama.

## 7.3.7 VRAM-efficient model loading

Ollama auto-unloads models after 5 minutes of inactivity. Tune via environment variable:

```bash
# Keep models loaded for 1 hour (more responsive, more idle VRAM use):
sudo systemctl edit ollama
```

Paste:

```ini
[Service]
Environment="OLLAMA_KEEP_ALIVE=1h"
Environment="OLLAMA_HOST=127.0.0.1:11434"
Environment="OLLAMA_NUM_PARALLEL=2"
Environment="OLLAMA_MAX_LOADED_MODELS=2"
```

- `OLLAMA_KEEP_ALIVE=1h` — models stay resident for 1 hour idle.
- `OLLAMA_NUM_PARALLEL=2` — up to 2 concurrent requests per model; good for "background coder model + foreground chat model" scenario.
- `OLLAMA_MAX_LOADED_MODELS=2` — max 2 models in VRAM at once.

Restart:

```bash
sudo systemctl restart ollama
```

With these settings, on 4 GB VRAM, you can run one 3B + one 1.5B concurrently. Or one 4B alone.

## 7.3.8 LoRA fine-tuning on 4 GB VRAM with Unsloth

The only realistic way to fine-tune on a 3050 Mobile. Unsloth is a library that accelerates Hugging Face training by ~2x and halves memory usage via custom CUDA kernels.

```bash
cd ~/work/ai/lora-experiments
uv init --python 3.12 --name lora-experiments
source .venv/bin/activate

# Unsloth for your CUDA version:
uv pip install "unsloth[cu130] @ git+https://github.com/unslothai/unsloth.git"

# Plus the usual:
uv pip install peft trl bitsandbytes accelerate wandb
```

### Training recipe — LoRA on Qwen 3B base

```python
# /// script
# requires-python = ">=3.12"
# dependencies = [
#   "unsloth",
#   "peft",
#   "trl",
#   "datasets",
#   "bitsandbytes",
# ]
# ///

from unsloth import FastLanguageModel
from trl import SFTTrainer
from transformers import TrainingArguments
from datasets import load_dataset

max_seq_length = 2048

# Load Qwen 2.5 3B with 4-bit quantization (fits in 4 GB VRAM):
model, tokenizer = FastLanguageModel.from_pretrained(
    model_name="unsloth/Qwen2.5-3B-bnb-4bit",
    max_seq_length=max_seq_length,
    load_in_4bit=True,
)

# Add LoRA adapters:
model = FastLanguageModel.get_peft_model(
    model,
    r=16,                           # LoRA rank
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj",
                    "gate_proj", "up_proj", "down_proj"],
    lora_alpha=16,
    lora_dropout=0,
    bias="none",
    use_gradient_checkpointing="unsloth",
    random_state=42,
)

# Your dataset:
dataset = load_dataset("your-username/your-dataset", split="train")

# Train:
trainer = SFTTrainer(
    model=model,
    tokenizer=tokenizer,
    train_dataset=dataset,
    dataset_text_field="text",
    max_seq_length=max_seq_length,
    args=TrainingArguments(
        per_device_train_batch_size=1,
        gradient_accumulation_steps=16,
        warmup_steps=5,
        num_train_epochs=1,
        learning_rate=2e-4,
        fp16=True,
        logging_steps=1,
        optim="adamw_8bit",
        weight_decay=0.01,
        lr_scheduler_type="cosine",
        seed=42,
        output_dir="outputs",
    ),
)

trainer.train()

# Save:
model.save_pretrained("lora_model")
tokenizer.save_pretrained("lora_model")

# Export to GGUF for Ollama:
model.save_pretrained_gguf("lora_model_gguf", tokenizer, quantization_method="q4_k_m")
```

Training time for ~10k samples: 4-8 hours. Keep the laptop plugged in, set `asusctl profile -P Performance`, watch `nvtop` — GPU should be at 99 % utilisation throughout.

## 7.3.9 Renting cloud GPUs when 4 GB is not enough

When the model you want to fine-tune is >7 B or the training set is too large for an 8-hour local run:

| Provider        | GPU option    | Price (2026)   | Good for                                      |
| --------------- | ------------- | -------------- | --------------------------------------------- |
| **Runpod**      | A100 80 GB    | ~$1.50/hr      | Hourly, quick experiments                     |
| **Paperspace**  | H100 80 GB    | ~$3.50/hr      | Notebook-style, integrated Jupyter            |
| **Lambda Labs** | 8x H100       | ~$30/hr        | Serious training runs                         |
| **vast.ai**     | Various       | $0.3-1/hr      | Cheapest, peer-to-peer, variable reliability  |

For a single fine-tune of a 7B model on a dataset too large for local, $10-30 total cloud spend is often cheaper than 3 days of laptop time. Scale up cloud, keep laptop for inference.

## 7.3.10 The "local copilot" everyday workflow

Your daily pattern with all of the above:

- VS Code tab autocomplete → Continue.dev → Qwen 2.5 Coder 3B on Ollama. Local, private, free.
- Firefox → `http://localhost:3000` → Open WebUI → general chat / RAG over your notes. Local.
- Cursor, for "big" refactors where you're willing to send code to the cloud → Claude via Cursor subscription.
- Fine-tuning local → Unsloth on laptop, or rent H100 for $5.

## 7.3.11 Snapshot

```bash
sudo timeshift --create --comments "post-local-llms $(date -I)" --tags D
```

Section 07 complete. Proceed to [08 Productivity and Security](../08-productivity-security/README.md).

---

[Home](../README.md) · [↑ 07 AI/ML](README.md) · [← Previous: 7.2 Python & PyTorch](02-python-uv-pytorch-jax.md) · **7.3 Local LLMs** · [Next: 08 Productivity →](../08-productivity-security/README.md)
