# Bifrost + Ollama HTTPS Proxy

This project runs Bifrost behind HTTPS and points it at an Ollama instance outside the container. The Ollama server can run on the same host or on another trusted machine on your LAN.

All stacks use PostgreSQL-backed Bifrost persistence, while Bifrost also keeps its local app data under `./bifrost` for the web UI and on-disk state.

## Compose Layout

Root compose files are the nginx + self-signed certificate variants:

- `docker-compose-win.yml`
- `docker-compose-mac.yml`
- `docker-compose-linux.yml`

Caddy variants live under `caddy/`:

- `caddy/docker-compose-win.yml`
- `caddy/docker-compose-mac.yml`
- `caddy/docker-compose-linux.yml`

All variants publish only `443`.

## Shared Environment

Create `.env` from `.env.example` and set the values you need before first start:

```dotenv
APP_HOST=0.0.0.0
APP_PORT=8080
LOG_LEVEL=info
LOG_STYLE=json
POSTGRES_DB=bifrost
POSTGRES_USER=bifrost
POSTGRES_PASSWORD=replace-with-a-random-postgres-password
OLLAMA_API_BASE=http://host.docker.internal:11434
SELF_SIGNED_CERT_CN=host.example.com
SELF_SIGNED_CERT_SAN=DNS:host.example.com,DNS:localhost,IP:127.0.0.1
SELF_SIGNED_CERT_DAYS=825
```

If Ollama runs on another machine, set `OLLAMA_API_BASE` to that host instead, for example `http://192.168.1.50:11434`.

## Windows

Use `docker-compose-win.yml` for the nginx + self-signed path.

1. Let Ollama listen for Docker Desktop:

```powershell
setx OLLAMA_HOST "0.0.0.0:11434"
```

Then fully restart the Ollama app.

2. Restrict direct inbound Ollama access to Docker Desktop or WSL:

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\configure-ollama-firewall.ps1
```

3. Start the self-signed nginx stack:

```powershell
docker compose -f docker-compose-win.yml up -d
```

4. Trust the generated self-signed certificate:

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\trust-selfsigned-cert.ps1
```

5. Or start the Caddy variant:

```powershell
docker compose -f caddy/docker-compose-win.yml up -d
powershell -ExecutionPolicy Bypass -File .\scripts\trust-caddy-root.ps1
```

## macOS

Use `docker-compose-mac.yml` for the nginx + self-signed path.

1. Let Ollama listen for Docker Desktop:

```bash
launchctl setenv OLLAMA_HOST "0.0.0.0:11434"
osascript -e 'quit app "Ollama"'
open -a Ollama
```

2. Restrict direct inbound Ollama access to Docker Desktop:

```bash
sudo sh ./scripts/configure-ollama-pf.sh
```

3. Start the self-signed nginx stack:

```bash
docker compose -f docker-compose-mac.yml up -d
```

4. Or start the Caddy variant:

```bash
docker compose -f caddy/docker-compose-mac.yml up -d
```

For macOS Caddy trust, export and trust the local Caddy root CA manually after the stack starts.

## Linux

Use `docker-compose-linux.yml` for the nginx + self-signed path.

The Linux compose files add `host.docker.internal:host-gateway` so Bifrost can reach a host-installed Ollama with the same default `OLLAMA_API_BASE` used on macOS and Windows.

1. Let host Ollama listen on all interfaces:

```bash
export OLLAMA_HOST="0.0.0.0:11434"
ollama serve
```

2. Start the self-signed nginx stack:

```bash
docker compose -f docker-compose-linux.yml up -d
```

3. Or start the Caddy variant:

```bash
docker compose -f caddy/docker-compose-linux.yml up -d
```

4. If Linux host firewalling is enabled, allow only the Docker host or the specific trusted subnet to reach Ollama on `11434`.

## Proxy Differences

nginx self-signed stacks:

- terminate HTTPS with `certs/server.crt` and `certs/server.key`
- auto-generate the cert pair on first start if missing
- use `scripts/trust-selfsigned-cert.ps1` on Windows to trust the cert

Caddy stacks:

- terminate HTTPS with `tls internal`
- generate and manage a local Caddy root CA automatically
- use `scripts/trust-caddy-root.ps1` on Windows to trust the Caddy CA

## First Bifrost Setup

After the stack starts, open `https://localhost/`.

In the Bifrost UI:

1. Add a provider named `ollama`.
2. Set its Base URL to the same value as `OLLAMA_API_BASE`.
3. Save the provider configuration.

Bifrost uses an OpenAI-compatible API. Once Ollama is configured, call it with model names prefixed by `ollama/`, for example `ollama/llama3.2:latest`.

Example:

```bash
curl -sk https://localhost/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "ollama/llama3.2:latest",
    "messages": [{"role": "user", "content": "Hello from Bifrost"}]
  }'
```

The `-k` flag is acceptable only for local ad hoc testing. For normal use, trust the local certificate authority or self-signed certificate and allow clients to validate HTTPS normally.

## Ollama Reachability

To verify reachability from inside the Bifrost container, test the actual Ollama endpoint:

```bash
docker exec -it <bifrost-container-name> sh
wget -qO- http://host.docker.internal:11434/api/tags
```

If Ollama runs on another machine, replace `host.docker.internal` with that machine's LAN IP.

## Common Commands

Windows self-signed:

```powershell
docker compose -f docker-compose-win.yml up -d
docker compose -f docker-compose-win.yml down
docker compose -f docker-compose-win.yml logs -f
```

macOS self-signed:

```bash
docker compose -f docker-compose-mac.yml up -d
docker compose -f docker-compose-mac.yml down
docker compose -f docker-compose-mac.yml logs -f
```

Linux self-signed:

```bash
docker compose -f docker-compose-linux.yml up -d
docker compose -f docker-compose-linux.yml down
docker compose -f docker-compose-linux.yml logs -f
```

Any Caddy variant:

```bash
docker compose -f caddy/docker-compose-<os>.yml up -d
docker compose -f caddy/docker-compose-<os>.yml down
docker compose -f caddy/docker-compose-<os>.yml logs -f
```

## Public Exposure

These stacks are intended for local or private-network use first.

If you want public internet exposure:

1. Do not expose Ollama directly.
2. Keep Bifrost behind the HTTPS proxy only.
3. Replace the local certificate strategy with a certificate issued for your real hostname.
4. Lock down the Bifrost UI and provider configuration path before exposing it publicly.
