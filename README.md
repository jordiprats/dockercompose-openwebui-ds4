# ds4 ROCm with Open WebUI

This repository provides a Docker Compose stack to run **[ds4](https://github.com/antirez/ds4)** — a high-performance local LLM inference server by Salvatore Sanfilippo (antirez) — alongside **[Open WebUI](https://github.com/open-webui/open-webui)** and **[Ollama](https://ollama.com/)** , all optimised for **AMD ROCm GPUs** (RDNA 3.5 / gfx1151).

## Architecture

```
┌──────────────┐     http://ds4:8000/v1     ┌──────────────┐
│  Open WebUI  │ ──────────────────────────→│  ds4-server  │
│  (frontend)  │                            │  (LLM)       │
└──────────────┘                            └──────────────┘
       │                                           │
       │ http://ollama:11434                       │ ROCm devices
       ▼                                           ▼
┌──────────────┐                           ┌──────────────┐
│   Ollama     │                           │  AMD GPU     │
│  (container) │                           │  (ROCm)      │
└──────────────┘                           └──────────────┘
```

- **Open WebUI** serves the chat interface on port `3000`.
- **Ollama** provides a separate model runner (optional).
- **ds4** runs the custom LLM server compiled with ROCm, exposed on port `8000`.
- Both Ollama and ds4 have direct access to AMD GPU devices (`/dev/kfd`, `/dev/dri`).

## Services

### 1. Ollama (`ollama`) — optional

> **Disabled by default.** Enable with `docker compose --profile ollama up -d`.

- Image: `ollama/ollama:rocm`
- Optimised for ROCm with `HSA_OVERRIDE_GFX_VERSION=11.0.0`
- Concurrent prompt handling (`OLLAMA_NUM_PARALLEL=4`)
- Flash attention enabled (`OLLAMA_FLASH_ATTENTION=1`)
- Models stay loaded in RAM (`OLLAMA_KEEP_ALIVE=-1`)
- OpenAI API passthrough enabled
- Volumes: `/mnt/llm/ollama` for model storage

### 2. Open WebUI (`open-webui`)

- Image: `ghcr.io/open-webui/open-webui:main`
- Port: `3000:8080`
- Connects to Ollama via `OLLAMA_BASE_URL`
- Connects to ds4 via a custom OpenAI API endpoint (see configuration below)
- Persistent backend storage at `/mnt/llm/openwebui/backend`

### 3. ds4 Server (`ds4`)

- **Build from source** using the Dockerfile in `./ds4`
- Clones [antirez/ds4](https://github.com/antirez/ds4) at the specified branch (`DS4_REF`)
- Compiled with ROCm (`make rocm`)
- Uses ROCm libraries: `hipblas`, `rocblas`, `rocwmma`, `hipcub`
- Patches `rocWMMA` headers for ROCm 7.1.0 compatibility
- Environment: `HSA_OVERRIDE_GFX_VERSION=11.5.1`
- Exposes port `8000`
- Key startup arguments:
  - `-m /models/<gguf-file>` — path to your GGUF model
  - `--ctx 100000` — context length
  - `--prefill-chunk 4096` — prefill chunk size
  - `--kv-disk-dir /kv/ds4-kv` — KV cache on disk (persistent across restarts)
  - `--kv-disk-space-mb 75000` — disk KV cache limit (~75 GB)
- Volumes:
  - `/home/jprats/ds4/gguf:/models` — GGUF model files (host path may need adjustment)
  - `/mnt/kvcache:/kv` — KV cache persistence

## Prerequisites

- **Hardware**: AMD GPU with ROCm support (tested on RDNA 3.5 / gfx1151)
- **Software**:
  - Docker with `compose` plugin
  - ROCm drivers installed on the host
  - Adequate disk space for GGUF models and KV cache

## Usage

### 1. Clone and configure

```bash
git clone <this-repo>
cd dockercompose-openwebui-ds4
```

### 2. Configure via `.env`

Copy the `.env` file and edit the values to match your host paths and preferences:

```bash
cp .env .env.local   # or edit .env directly
```

Key variables to customise:

| Variable | Default | Description |
|---|---|---|
| `OLLAMA_VOLUME` | `/mnt/llm/ollama` | Ollama model storage on host |
| `OPENWEBUI_VOLUME` | `/mnt/llm/openwebui/backend` | Open WebUI backend persistence |
| `DS4_MODEL_DIR` | `/home/jprats/ds4/gguf` | Directory containing your GGUF model |
| `DS4_MODEL` | `DeepSeek-V4-Flash-...-0731.gguf` | GGUF model filename (path inside container) |
| `KV_CACHE_DIR` | `/mnt/kvcache` | Directory for disk-based KV cache |
| `WEBUI_SECRET_KEY` | (set to a fixed key) | Open WebUI secret — change to a random string |
| `DS4_CTX` | `100000` | Context window size (tokens) |
| `DS4_KV_DISK_SPACE_MB` | `75000` | Max disk space for KV cache (MB) |
| `HSA_OVERRIDE_GFX_VERSION_OLLAMA` | `11.0.0` | ROCm target for Ollama |
| `HSA_OVERRIDE_GFX_VERSION_DS4` | `11.5.1` | ROCm target for ds4 |

### 3. Download the GGUF model

The model is publicly hosted on Hugging Face at [`antirez/deepseek-v4-gguf`](https://huggingface.co/antirez/deepseek-v4-gguf) — **no token required**.

The ds4 repository provides a [`download_model.sh`](https://github.com/antirez/ds4/blob/main/download_model.sh) script you can run inside the container:

```bash
# Run the download script inside the ds4 container
# (the script is included in the cloned repo at /build/ds4/download_model.sh)
docker exec ds4 /build/ds4/download_model.sh ds4f-q2
```

Or manually download the file and place it in the directory mapped to `/models` (currently `/home/jprats/ds4/gguf` on the host). The 0731 model filename is:
`DeepSeek-V4-Flash-IQ2XXS-w2Q2K-AProjQ8-SExpQ8-OutQ8-chat-v2-imatrix-0731.gguf`

### 4. Start the stack

```bash
# Start ds4 + Open WebUI (ollama disabled by default)
docker compose up -d

# If you also want Ollama:
docker compose --profile ollama up -d
```

The first time ds4 is started, it will **build from source** (this can take a while). Subsequent starts are instant.

### 5. Connect Open WebUI to ds4

1. Log into Open WebUI at `http://localhost:3000` as an administrator.
2. Navigate to **Settings → Connections**.
3. Under **OpenAI API**, click the **+** icon or edit an existing entry.
4. Set the **API Base URL** to `http://ds4:8000/v1`.
5. Set the **API Key** to any placeholder (e.g. `ds4-secret`) — ds4 does not require authentication.
6. Click **Save**.

Now you can chat using the ds4 model via Open WebUI.

### 6. (Optional) Use Ollama models

If you enabled Ollama (via `--profile ollama`), Open WebUI connects to it via `OLLAMA_BASE_URL`. Pull models as usual:

```bash
docker exec -it ollama ollama pull <model-name>
```

## Configuration Reference

| Variable / Argument | Description |
|---|---|
| `HSA_OVERRIDE_GFX_VERSION` | Forces ROCm compilation target for your GPU architecture |
| `OLLAMA_NUM_PARALLEL` | Max concurrent prompt threads |
| `OLLAMA_FLASH_ATTENTION` | Enables flash attention (reduces memory) |
| `OLLAMA_KEEP_ALIVE` | Keeps model loaded indefinitely (`-1`) |
| `--ctx` | Context window size (tokens) |
| `--prefill-chunk` | Tokens processed per prefill chunk |
| `--kv-disk-dir` | Directory for disk-backed KV cache |
| `--kv-disk-space-mb` | Max disk space for KV cache (MB) |

## Troubleshooting

- **Build failures**: Ensure your host has ROCm installed. The Dockerfile installs `hipcc`, `rocm-smi`, `rocminfo`, and ROCm libraries.
- **GPU not detected**: Verify `/dev/kfd` and `/dev/dri` exist on the host. Run `rocminfo` inside the container to confirm GPU visibility.
- **Out of memory**: Reduce `--ctx` or increase `--kv-disk-space-mb`.
- **Open WebUI can't reach ds4**: The `extra_hosts` entry `host.docker.internal` may help if ds4 is on a different network. The service name `ds4` should be resolvable within the Compose network.

## License

This repository is a configuration wrapper. Refer to the respective licenses of [ds4](https://github.com/antirez/ds4), [Open WebUI](https://github.com/open-webui/open-webui), and [Ollama](https://github.com/ollama/ollama).
