# Using a Remote Ollama Server Locally

Guide for accessing an Ollama instance running on a remote AI server (via SSH) from your local machine, and hooking it up to tools like Cline and Continue in VSCode.

## Background

- Ollama runs a local API server, listening on port `11434` by default.
- If Ollama is only bound to `localhost` on the remote server, you can't reach it directly from your Mac — you need to either tunnel into it or expose it on the network.

## Option A: SSH Tunnel (quick, no server config changes)

From a terminal on your **local machine**, run:

```bash
ssh -L 11434:localhost:11434 kc-llm@100.112.124.101
```

### What this actually does
This command does **two things at once**:
1. Opens an interactive SSH session on the remote server (you'll see the remote prompt, e.g. `kc-llm@kc-llm-MS-7E85:~$`).
2. In the background, forwards port `11434` on your Mac to port `11434` on the server.

Both are true at the same time — logging into the remote shell is expected, not an error.

### Keeping the tunnel alive
- **Do not close this window** — closing it kills the tunnel.
- To actually use the tunnel from your Mac, open a **separate, new local terminal window** (do *not* SSH anywhere in this one).

### Testing the tunnel
In the new local (non-SSH) window:

```bash
curl http://localhost:11434/api/tags
```

You should get back JSON listing your models (e.g. `qwen3.5:35b`).

### Running a model locally through the tunnel
If you have the `ollama` CLI installed on your Mac (`brew install ollama`):

```bash
ollama run qwen3.5:35b
```

This routes through the tunnel to the remote server's model.

## Option B: Expose Ollama on the network (persistent, no tunnel needed each time)

On the **server**, set the `OLLAMA_HOST` environment variable so Ollama listens on all interfaces instead of just localhost:

```bash
OLLAMA_HOST=0.0.0.0:11434 ollama serve
```

If Ollama runs as a systemd service, edit the service file to add that env var, then:

```bash
sudo systemctl daemon-reload && sudo systemctl restart ollama
```

Once exposed, you can hit it directly from your Mac without a tunnel:

```bash
curl http://100.112.124.101:11434/api/tags
```

### Note on Tailscale
The IP `100.112.124.101` is in the `100.x.x.x` range, which usually indicates a **Tailscale** address. If both your Mac and the server are on the same tailnet, you get a private, direct connection — no need to open the port to the public internet. Still requires Ollama to be bound to `0.0.0.0` (or the Tailscale interface) rather than only `127.0.0.1`.

### Security note
Ollama's API has **no authentication by default**. If you expose it beyond localhost:
- Restrict access via firewall rules to your own IP, or
- Keep it on a private network (e.g. Tailscale/VPN) rather than opening it to the public internet.

## Using it in Cline (VSCode)

1. Open Cline settings.
2. Select **Ollama** as the API provider.
3. Set the base URL:
   - `http://localhost:11434` if using the SSH tunnel
   - `http://100.112.124.101:11434` if using Tailscale / exposed network access
4. Select your model from the dropdown.

## Using it in Continue (VSCode)

In `config.json`:

```json
{
  "models": [
    {
      "title": "Remote Ollama",
      "provider": "ollama",
      "model": "llama3",
      "apiBase": "http://<server-ip-or-localhost>:11434"
    }
  ]
}
```

## OpenAI-Compatible Endpoint

Ollama also exposes an OpenAI-compatible API at `/v1`. Any tool that expects an "OpenAI API base URL" can be pointed to:

```
http://<server-ip-or-localhost>:11434/v1
```

This is useful for tools that don't have a native Ollama integration option.
