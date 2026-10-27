# Hermes Agent Docker Compose

Run [Hermes Agent](https://github.com/NousResearch/hermes-agent) with Docker or Podman. Setup wizard walks you through everything.

## Quick Start

```bash
git clone https://github.com/DeployFaith/hermes-docker-compose.git
cd hermes-docker-compose

docker compose up -d
docker compose exec ollama ollama pull gpt-oss:20b
docker compose exec gateway hermes setup
```

The setup wizard configures your LLM provider, messaging platforms, tools, and personality. To use the local Ollama service, register it once with the `providers.ollama` block in [Local Models](#local-models-ollama), then run `docker compose exec gateway hermes model` and select any downloaded model.

## Configuration

Copy `.env.example` to `.env` and edit as needed:

```bash
cp .env.example .env
```

| Variable                | Default     | Description                                  |
| ----------------------- | ----------- | -------------------------------------------- |
| `HERMES_DATA_DIR`       | `~/.hermes` | Host path for persistent data                |
| `GATEWAY_PORT`          | `8642`      | Host port for the gateway API                |
| `DASHBOARD_PORT`        | `9120`      | Host port for the dashboard UI               |
| `OLLAMA_DATA_DIR`       | `~/.ollama` | Host path for downloaded models              |
| `OLLAMA_PORT`           | `11434`     | Host port for the Ollama API (loopback only) |
| `OLLAMA_CONTEXT_LENGTH` | `65536`     | Default context window                       |
| `OLLAMA_KEEP_ALIVE`     | `30m`       | How long an idle model stays loaded          |

## Usage

```bash
docker compose up -d                            # start

# If using Ollama, first pull a model:
docker compose exec ollama ollama pull <model>  # download a local model
docker compose exec ollama ollama list          # list downloaded models

# Then run the setup wizard and start the interactive CLI:
docker compose exec gateway hermes setup        # first-time setup wizard
docker compose exec gateway hermes              # interactive CLI
docker compose logs -f gateway                  # logs
docker compose down                             # stop
docker compose pull && docker compose up -d     # upgrade
```

Podman users can run the same commands with `podman compose` (or an installed Docker Compose provider).

## Local Models (Ollama)

The `ollama` service runs on the internal network as `http://ollama:11434`. Hermes reaches it by container name, so the published port only matters for host-side access.

Pull a model:

```bash
docker compose exec ollama ollama pull gpt-oss:20b
```

### Automatic model discovery

Register Ollama once in `$HERMES_DATA_DIR/config.yaml`:

```yaml
providers:
  ollama:
    api: http://ollama:11434/v1
    discover_models: true
```

`model.default` is just the initial model, not a hardcoded catalog. Switch models later with `docker compose exec gateway hermes model`, or `/model` inside a chat. After pulling a new model, discovery may take up to five minutes to refresh; force it with:

```bash
docker compose exec gateway hermes model --refresh
```

Restart the containers after changing `config.yaml`:

```bash
docker compose restart gateway dashboard
```

Hermes detects each model's declared context window and sends it to Ollama automatically. `OLLAMA_CONTEXT_LENGTH` is only the server-side default for models that do not declare one.

Model suggestions for a 16 GB NVIDIA GPU:

| Model               |    Size | Context | Notes                                        |
| ------------------- | ------: | ------: | -------------------------------------------- |
| `gpt-oss:20b`       |  ~14 GB |    128K | Best all-round agent/tool default; tight fit |
| `devstral:24b`      |  ~14 GB |    128K | Agentic coding; tight fit, shorter context   |
| `qwen3.5:9b`        | ~6.6 GB |    256K | Fast fallback with room for long context     |
| `qwen2.5-coder:14b` |   ~9 GB |     32K | Reliable coding fallback                     |

Models above ~16 GB, such as `qwen3-coder:30b` and `qwen3.6:27b`, spill to system RAM and run much slower. Check placement with `docker compose exec ollama ollama ps`; `100% GPU` is ideal.

GPU access uses CDI (`devices: ["nvidia.com/gpu=all"]`) and requires the NVIDIA Container Toolkit plus a CDI spec. Docker Engine 28.3+ supports this by default; on Podman, generate the spec once with:

```bash
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
```

Remove the `devices` line in `docker-compose.yml` to run on CPU. On Docker older than 28.3, replace it with the legacy reservation:

```yaml
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: all
          capabilities: [gpu]
```

## Data

Everything persists in `$HERMES_DATA_DIR` on your host. Ollama models persist in `~/.ollama` (or `OLLAMA_DATA_DIR`). Back both up.

A systemd-managed Ollama service often stores models under `/var/lib/ollama/.ollama` instead of `~/.ollama`. To reuse those models, set `OLLAMA_DATA_DIR=/var/lib/ollama/.ollama` and make sure the container can read and write that directory.

With rootful Docker, files written by the Ollama container are owned by root. Use a separate `OLLAMA_DATA_DIR` for the container if that causes permission issues. Rootless Podman does not have this problem.

## Auto-Updates

Uncomment the `watchtower` block in `docker-compose.yml` to enable automatic image updates.

## Resources

- Docs: [hermes-agent.nousresearch.com](https://hermes-agent.nousresearch.com)
- GitHub: [github.com/NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)
- Discord: [discord.gg/NousResearch](https://discord.gg/NousResearch)

## License

MIT
