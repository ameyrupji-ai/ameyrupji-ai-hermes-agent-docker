# Hermes Agent — Docker Setup

Runs [Hermes Agent](https://github.com/NousResearch/hermes-agent) in Docker, talking to a local model served by [LM Studio](https://lmstudio.ai) on your Mac.

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) running on macOS
- [LM Studio](https://lmstudio.ai) installed and running with a model loaded
- LM Studio's local server enabled (default port `1234`)

### Tested configuration

| | |
|---|---|
| **Machine** | MacBook Pro (Mac14,9) |
| **Chip** | Apple M2 Pro — 10 cores (6P + 4E) |
| **Memory** | 16 GB |
| **macOS** | macOS 26.1 (Build 25B78) |
| **Model** | `qwen/qwen3-vl-4b` via LM Studio |
| **Model context** | 32,768 tokens (set in LM Studio when loading the model) |

---

## Project structure

```
hermes-agent-docker/
├── docker-compose.yml
├── data/
│   └── config.yaml   ← model endpoint config
└── README.md
```

All agent state (sessions, memories, skills, API keys) is persisted in `data/`, which maps to `/opt/data` inside the container.

---

## Setup

### 1. Start LM Studio

Open LM Studio, load a model (e.g. `qwen3-14b`), and start the local server:

- Go to **Local Server** tab
- Click **Start Server** (default: `http://localhost:1234`)

### 2. Configure the model name

Edit `data/config.yaml` and set `name` to match the model identifier shown in LM Studio:

```yaml
model:

```

### 3. Run first-time setup

This initialises the `data/` directory and walks you through the Hermes configuration wizard:

```bash
docker compose run --rm hermes setup
```

> You can skip or accept defaults for anything that does not apply (e.g. API keys for cloud providers).

---

## Running

### Interactive CLI

```bash
docker compose up
```

The Hermes terminal opens in your current shell. Type your prompt and press **Enter**.

Press `Ctrl+C` to exit.

### Switch model interactively

If you want to pick a different model (Hermes will auto-discover what LM Studio has loaded):

```bash
docker compose run --rm hermes model
```

---

## Dashboard

The web dashboard starts automatically alongside the CLI and is accessible at:

```
http://localhost:9119
```

It provides a browser-based interface to view sessions, memories, and agent configuration.

---

## Gateway mode (background / persistent)

To run Hermes as a persistent background service instead of an interactive CLI, edit `docker-compose.yml`:

```yaml
command: gateway run   # replace: command: hermes
```

Then start in detached mode:

```bash
docker compose up -d
```

Stop it with:

```bash
docker compose down
```

---

## Talking to Hermes via curl (API server)

The agent exposes an OpenAI-compatible API on port **8642**. The API key is set by `API_SERVER_KEY` in `docker-compose.yml` (default: `change-me-local-dev` — change this before exposing to a network).

### Health check

```bash
curl http://localhost:8642/health
```

### Send a message (single turn)

```bash
curl http://localhost:8642/v1/chat/completions \
  -H "Authorization: Bearer change-me-local-dev" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "hermes-agent",
    "messages": [
      {"role": "user", "content": "Hello, what can you do?"}
    ]
  }'
```

### Send a message (streaming)

```bash
curl http://localhost:8642/v1/chat/completions \
  -H "Authorization: Bearer change-me-local-dev" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "hermes-agent",
    "messages": [
      {"role": "user", "content": "Hello, what can you do?"}
    ],
    "stream": true
  }'
```

### Multi-turn conversation (server-side state)

```bash
# First message — returns a response ID
curl http://localhost:8642/v1/responses \
  -H "Authorization: Bearer change-me-local-dev" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "hermes-agent",
    "input": "What files are in the current directory?",
    "store": true
  }'

# Follow-up — chain using the previous response ID
curl http://localhost:8642/v1/responses \
  -H "Authorization: Bearer change-me-local-dev" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "hermes-agent",
    "input": "Now show me the contents of the README.",
    "previous_response_id": "<id from previous response>"
  }'
```

### List available models

```bash
curl http://localhost:8642/v1/models \
  -H "Authorization: Bearer change-me-local-dev"
```

> **Note:** The API server requires the gateway to be running. It is active in both interactive CLI mode (`command: hermes`) and gateway mode (`command: gateway run`). To change the API key, update `API_SERVER_KEY` in `docker-compose.yml` and restart with `docker compose up`.

---

## Networking notes

| Context | Address to use |
|---|---|
| Docker container → LM Studio on Mac | `host.docker.internal:1234` (already set in `config.yaml`) |
| Browser → Hermes dashboard | `localhost:9119` |
| curl / API clients → Hermes agent | `localhost:8642` |

`host.docker.internal` is provided automatically by Docker Desktop on macOS. The `extra_hosts` entry in `docker-compose.yml` ensures it also resolves on Linux hosts.

---

## Troubleshooting

**Container cannot reach LM Studio**
Verify LM Studio's server is running and bound to `0.0.0.0` (not just `127.0.0.1`). In LM Studio go to **Local Server → Server Settings** and enable "Allow connections from network".

**Dashboard not loading**
The dashboard starts a few seconds after the container boots. Wait a moment and refresh `http://localhost:9119`.

**Wrong model name**
If Hermes cannot find the model, run `docker compose run --rm hermes model` to pick from the list of models LM Studio is currently serving.
