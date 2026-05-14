# Distributed Routing Across Two Ollama Nodes

This guide shows how to run Ollama on two machines and route requests between them:

- Raspberry Pi (8 GB RAM): smaller quantized models and light requests
- Windows PC (16 GB RAM): larger models and heavier requests

> [!IMPORTANT]
> Do not try to split one single inference across both machines. Run each machine as its own Ollama server and route requests between servers.

## 1) Set up both Ollama nodes

Install Ollama on both hosts:

- Raspberry Pi/Linux: https://docs.ollama.com/linux
- Windows: https://docs.ollama.com/windows

Bind each server to a LAN-reachable address using `OLLAMA_HOST`.

Examples:

```shell
# Linux / Raspberry Pi
OLLAMA_HOST=0.0.0.0:11434 ollama serve
```

```powershell
# Windows PowerShell
$env:OLLAMA_HOST="0.0.0.0:11434"
ollama serve
```

Allow inbound TCP `11434` on each host firewall.

Verify connectivity from a client on the same LAN:

```shell
curl http://<pi-ip>:11434/api/tags
curl http://<windows-ip>:11434/api/tags
```

## 2) Place models by memory budget

Keep model names/tags consistent where possible so clients and routers can switch nodes without model-name rewrites.

Suggested placement:

- Pi: small models and smaller quantizations (for example 1B to 3B classes, Q4/Q5 style quantization)
- Windows: medium/larger models and heavier prompts

Use `/api/tags` to confirm inventory on each node.

## 3) Add a routing layer

Use a single endpoint for clients (reverse proxy or lightweight API router), then route to node backends.

Routing policy example:

- Heavy/long prompts -> Windows
- Light/short prompts -> Pi
- If preferred node is down -> fail over to the other node

A practical approach is to route by model name (for example, `llama3.2:3b` to Pi and `qwen3:8b` to Windows), then add prompt-size rules if needed.

## 4) Health checks and failover

Health-check both backends with `/api/tags`:

- Mark backend unhealthy on timeout or repeated failures
- Stop routing traffic to unhealthy backend
- Re-enable after successful checks

Keep checks lightweight and frequent enough to detect failure quickly.

## 5) Tune node performance

Use environment variables to protect each node from overload:

- `OLLAMA_NUM_PARALLEL`: cap concurrent requests
- `OLLAMA_MAX_QUEUE`: cap queued requests
- `OLLAMA_KEEP_ALIVE`: control model residency in memory
- `OLLAMA_CONTEXT_LENGTH`: set default context limit if needed

On the Pi, use stricter limits and smaller contexts to avoid swap pressure.

Per-request controls in the API (`num_ctx`, `num_predict`, `keep_alive`) can further limit memory and latency for weaker nodes.

## 6) Observe and adjust

Track at minimum:

- Request latency per node
- Error rate/timeouts per node
- Queue pressure and saturation events
- Host memory usage

Then adjust routing weights and limits (typically favoring the Windows node for heavier traffic).

## 7) Secure LAN exposure

- Restrict access to trusted subnets/VLANs only
- Prefer exposing a single router endpoint instead of both raw node ports to all clients
- For browser clients, explicitly set `OLLAMA_ORIGINS` to trusted origins
- If multiple users share the service, put authentication and TLS at the router/proxy layer

## 8) Validate in stages

1. Baseline each node independently with the same client.
2. Send mixed traffic through the router and verify policy behavior.
3. Stop one node and confirm failover works.
4. Bring the node back and confirm automatic recovery.
5. Repeat under sustained load and tune queue/concurrency limits.

## Quick operational checklist

- [ ] Both nodes reachable over LAN (`/api/tags` works)
- [ ] Models are intentionally split by memory budget
- [ ] Router has clear rules and backend failover
- [ ] Health checks remove/re-add backends automatically
- [ ] Pi has stricter queue/concurrency/context limits
- [ ] Access is restricted and browser origins are locked down
- [ ] Recovery behavior is tested and documented
