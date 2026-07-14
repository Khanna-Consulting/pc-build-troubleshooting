# Firstmate + OpenCode + Ollama (Local Qwen) Setup

Run the full firstmate multi-agent orchestration on the kc-llm server using a local Qwen model via Ollama — no cloud API needed.

## Prerequisites

- SSH access to the server: `ssh kc-llm` (key-based auth configured)
- Ollama installed with `qwen3.5:35b` pulled
- tmux installed (`sudo apt install tmux -y`)

## 1. SSH Key Auth (one-time, from Mac)

```bash
# Add to ~/.ssh/config
Host kc-llm
  HostName 100.112.124.101
  User kc-llm
  IdentityFile ~/.ssh/id_personal

# Copy key to server (password required once)
ssh-copy-id -i ~/.ssh/id_personal.pub kc-llm
```

## 2. Copy Tools to Server (from Mac)

> Already done as of 2026-07-14. Only re-run if tools are updated locally.

```bash
rsync -av --progress --exclude='.git' ~/tools/ kc-llm:~/tools/
```

This copies: `firstmate`, `gnhf`, `lavish-axi`, `no-mistakes`, `treehouse`.

## 3. Install OpenCode on Server

> Already installed (v1.18.0) as of 2026-07-14.

```bash
ssh kc-llm
curl -fsSL https://opencode.ai/install | bash
```

## 4. Configure OpenCode to Use Ollama/Qwen

Write the config to `~/.config/opencode/opencode.jsonc`:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "model": "ollama/qwen3.5:35b",
  "provider": {
    "ollama": {
      "api": "openai",
      "options": {
        "baseURL": "http://localhost:11434/v1",
        "apiKey": "ollama"
      },
      "models": {
        "qwen3.5:35b": {
          "id": "qwen3.5:35b",
          "name": "Qwen 3.5 35B",
          "tool_call": true,
          "temperature": true,
          "limit": {
            "context": 131072,
            "output": 8192
          },
          "cost": {
            "input": 0,
            "output": 0
          }
        }
      }
    }
  }
}
```

## 5. Set Environment Variables

Add to `~/.bashrc`:

```bash
export LOCAL_ENDPOINT=http://localhost:11434/v1
export OPENAI_API_KEY=dummy
```

Then `source ~/.bashrc`.

## 6. Set Firstmate Crew Harness to OpenCode

```bash
mkdir -p ~/tools/firstmate/config
echo "opencode" > ~/tools/firstmate/config/crew-harness
```

This tells firstmate to spawn crewmates via opencode (which routes to local Qwen) instead of Claude.

## 7. Run

```bash
ssh kc-llm
tmux
cd ~/tools/firstmate
opencode
```

Firstmate spawns crewmates in tmux windows via opencode, all powered by the local Qwen 3.5 35B model through Ollama.

## Architecture

```
Mac (Claude + firstmate)     Server (OpenCode + firstmate)
        |                              |
   Claude API                   Ollama (localhost:11434)
        |                              |
   firstmate                    firstmate
        |                              |
   crew: claude              crew: opencode -> Qwen 3.5 35B
```

## Troubleshooting

- **OpenCode still shows GPT-5.3**: Make sure `~/.config/opencode/opencode.jsonc` has the correct config and restart opencode
- **Ollama not responding**: Run `ollama serve` in a separate tmux pane, then `ollama run qwen3.5:35b` to verify the model works
- **scp/rsync slow**: The Tailscale connection goes through a relay (`relay "nyc"`). Use `--exclude='.git'` to skip ~400MB of git history
- **SSH still asks for password**: Check permissions on the server: `chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys`
