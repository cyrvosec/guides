---
title: "Hermes Agent: step-by-step install (cloud or 100% local)"
description: "Install Hermes Agent, connect it to a model (cloud or local with Ollama), test it, and keep it fast and safe."
---

# 🤖 Hermes Agent: step-by-step install

**Hermes Agent** is an open-source AI agent that runs in your terminal. It can run commands, read and write files, remember things between sessions, schedule jobs and keep running 24/7. It's built by [Nous Research](https://nousresearch.com) and released under the MIT license.

You choose the "brain" (the AI model):

| Path | Good for | You need |
|---|---|---|
| **A. Cloud model** | Fastest start, smartest answers | An account and API key from a provider (e.g. Nous Portal, OpenRouter) |
| **B. 100% local with Ollama** | Privacy: nothing leaves your machine | 16 GB RAM or more; a GPU makes it much faster |

> Tested by us in October 2026 (Hermes Agent v0.22, Ollama 0.40) on Ubuntu Server, CPU only.

---

## ⚠️ Read this first: an agent is not a chatbot

Hermes can **run commands on your computer**. Treat it like a new employee with a terminal:

- Install it under a **separate user** or in a **VM**, not on the machine with your bank and personal files.
- Turn on only the tools you need (`hermes tools`). Start without web browsing.
- **Prompt injection:** if the agent reads emails, web pages or files from strangers, hidden text in them can tell it to do things. Don't connect it to untrusted input while it has terminal access.
- Keep API keys only in `~/.hermes/.env`, never in a chat or a public file.

---

## Step 1: Install Hermes

**Linux, macOS or WSL2:**

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh -o install-hermes.sh
less install-hermes.sh        # read what it does before running it
bash install-hermes.sh
source ~/.bashrc              # or: source ~/.zshrc
```

**Windows (PowerShell):**

```powershell
iex (irm https://hermes-agent.nousresearch.com/install.ps1)
```

The installer brings its own Python, Node.js, ripgrep and FFmpeg. Check it worked:

```bash
hermes --version
```

> **Antivirus warning on Windows?** Some antivirus tools flag `uv.exe` in the Hermes folder. The official README explains how to verify the file's checksum against the official release. Verify it first; don't just turn the antivirus off.

---

## Step 2A: Connect a cloud model (easiest)

```bash
hermes setup
```

The wizard asks for the provider and the API key. Nous Portal has its own shortcut: `hermes setup --portal`. Skip to **Step 4**.

---

## Step 2B: Local model with Ollama (private)

1. Install Ollama from [ollama.com/download](https://ollama.com/download).
2. Download a model that supports **tool calling** (the agent needs it to run commands). A 7B–9B model is the minimum that works; bigger is smarter but slower. Browse [ollama.com/search](https://ollama.com/search) and filter by **tools**:

```bash
ollama pull <model-name>
ollama show <model-name>      # look for "tools" under Capabilities
```

---

## Step 3: Give the model a big enough memory (local only)

**This is the step most guides miss.** Hermes **refuses to start below a 64,000-token context window**. Ollama's default is much smaller, and the error you see is just *"The model request failed"* (the real reason is in `~/.hermes/logs/`).

**Linux (Ollama as a system service):**

```bash
sudo systemctl edit ollama
```

Add these lines, then save:

```ini
[Service]
Environment="OLLAMA_CONTEXT_LENGTH=65536"
Environment="OLLAMA_FLASH_ATTENTION=1"
Environment="OLLAMA_KV_CACHE_TYPE=q8_0"
Environment="OLLAMA_KEEP_ALIVE=-1"
```

```bash
sudo systemctl restart ollama
```

**Windows / macOS:** set the same four variables as user environment variables, then quit and reopen Ollama.

What they do: 64K context for Hermes; flash attention and a compressed (q8_0) cache so 64K fits in less RAM; keep-alive so the model stays loaded instead of reloading every time.

---

## Step 4: Point Hermes at the model

```bash
hermes model
```

- **Cloud:** pick your provider and model.
- **Local:** choose the **custom / OpenAI-compatible** option and enter:
  - URL: `http://127.0.0.1:11434/v1`
  - Model: the exact name from `ollama list`

Then tell Hermes the context size:

```bash
hermes config set model.context_length 65536
```

---

## Step 5: Test it

```bash
hermes doctor
```

Fix anything marked as an error. Warnings about features you don't use (Discord, image generation…) are fine.

Now a real test, where the agent has to use a tool on its own:

```bash
hermes chat -q "Use the terminal tool to run: uptime -p. Then reply with only its output." --oneshot
```

If it answers with your uptime, the agent works end to end. 🎉 For a normal conversation, just type `hermes`.

---

## Step 6 (optional): Keep it running 24/7

The **gateway** runs Hermes in the background (cron jobs, messaging apps):

```bash
hermes gateway setup
hermes gateway start
hermes gateway status
```

**Linux:** so it survives logout and reboots, enable "linger" for your user:

```bash
loginctl enable-linger $USER
```

---

## ⚡ Why is it slow? (local models)

These are our real numbers on a mini PC with a 6-core mobile CPU, no GPU, 9B model:

| What | Speed |
|---|---|
| Model reading the prompt | ~37 tokens/s |
| Model writing the answer | ~8 tokens/s |
| **First** message after a restart | ~4–6 min |
| New session once warm | **~45 s** |
| Each follow-up step | 3–15 s |

Why: every request sends Hermes' instructions and tool list (about 7,500 tokens). The **first** time, the model has to read all of it. After that, Ollama **caches** it, so it's fast. The cache is lost when you restart, change the Hermes config, or send a different prompt through the same Ollama.

How to speed it up:

1. **GPU:** the biggest win. Reading the prompt goes from minutes to seconds.
2. **Fewer skills and tools:** `hermes skills config` and `hermes tools`. Turn off what you don't use (music, Apple apps, video…). Fewer tools also confuse a small model less.
3. **Warm-up:** send one tiny prompt after boot so the cache is ready when you need it.

---

## 🛠️ Troubleshooting

| Problem | Fix |
|---|---|
| "The model request failed" | Context under 64K. Redo Step 3 and check `~/.hermes/logs/` |
| `hermes: command not found` | Open a new terminal or run `source ~/.bashrc` |
| A script or SSH command with `hermes` / `ollama run` hangs | They wait for keyboard input. Add `</dev/null` at the end of the command |
| `ollama pull` fails on a server | The `ollama` service user may have no internet. Download the model file another way and import it with `ollama create` |
| Very slow | See "Why is it slow?" above |

**Useful commands:** `hermes doctor` · `hermes tools list` · `hermes skills list` · `hermes insights` · `hermes update`

---

*Guide by [@cyrvosec](https://instagram.com/cyrvosec). Cyrvo is not affiliated with Nous Research. Hermes Agent is MIT-licensed; official docs: [hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/).*
