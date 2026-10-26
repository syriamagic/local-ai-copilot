# Building a Local AI Co-Pilot for IT System & Network Administrators

**Target Audience:** Senior IT System and Network Administrators  
**Hardware Baseline:** HP ZBook Fury G9 (12GB VRAM GPU, 32GB RAM) or equivalent  
**Goal:** Deploy a private, local AI assistant (Ollama + Open WebUI) capable of generating scripts, proposing automation, and using a custom knowledge base for reverse engineering and sysadmin tasks.

---

## Table of Contents

1. [Prerequisites & GPU Driver Installation (OS-Specific)](#1-prerequisites--gpu-driver-installation)
2. [Installing Ollama & Selecting Models](#2-installing-ollama--selecting-models)
3. [Deploying Open WebUI (Podman)](#3-deploying-open-webui-podman)
4. [Troubleshooting Networking & SELinux](#4-troubleshooting-networking--selinux)
5. [Configuring the AI Persona (System Prompt)](#5-configuring-the-ai-persona-system-prompt)
6. [Creating Custom Python Tools (Function Calling)](#6-creating-custom-python-tools-function-calling)
7. [Adding Custom Knowledge (RAG)](#7-adding-custom-knowledge-rag)
8. [Maintenance & Updates](#8-maintenance--updates)

---

## 1. Prerequisites & GPU Driver Installation

To leverage GPU acceleration for AI inference, you must install the proprietary NVIDIA drivers. **Do not use Nouveau** (open-source).

### 🔴 Fedora (Tested on Fedora 44)

```bash
# 1. Enable RPM Fusion
sudo dnf install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
                 https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

# 2. Install drivers
sudo dnf update -y
sudo dnf install -y akmod-nvidia xorg-x11-drv-nvidia-cuda

# 3. Reboot
sudo reboot
```

### 🟠 Ubuntu (22.04 / 24.04 LTS)

```bash
sudo ubuntu-drivers autoinstall
# OR explicitly:
sudo apt install nvidia-driver-535 nvidia-utils-535
sudo reboot
```

### 🟣 CentOS / RHEL (8 / 9)

```bash
# 1. Enable EPEL and install required tools
sudo dnf install -y epel-release
sudo dnf install -y kernel-devel kernel-headers gcc make dkms

# 2. Add NVIDIA Repository (RHEL 9 example)
sudo dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo

# 3. Install drivers
sudo dnf module install -y nvidia-driver:latest-dkms
sudo reboot
```

### Verification (All OS)

Run `nvidia-smi`. You should see your GPU (e.g., RTX A3000) and the CUDA version.

---

## 2. Installing Ollama & Selecting Models

### Installation (All OS)

```bash
curl -fsSL https://ollama.com/install.sh | sh
sudo systemctl enable --now ollama
```

> On Fedora, you can alternatively use `sudo dnf install ollama`.

### Required System Tweak (Listen on all interfaces)

To allow the WebUI container to talk to Ollama, modify the systemd service:

```bash
sudo systemctl edit ollama.service
```

Add:

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
```

Reload and restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

### Model Selection (Optimized for 12GB VRAM)

Pull the models using `ollama pull <model>`:

| Model | Purpose | VRAM |
|-------|---------|------|
| `qwen2.5-coder:14b` (or `7b`) | Best for scripting, automation, and tool calling. | ~9GB |
| `gemma4:12b` | Excellent for general reasoning, analyzing logs, and multimodal tasks. | ~8GB |
| `nomic-embed-text` | Required for RAG (Knowledge Base). | ~270MB |

---

## 3. Deploying Open WebUI (Podman)

> **⚠️ CRITICAL NOTICE:** Modern Linux distributions (especially Fedora 44+) use Podman 6+ with Pasta networking. `slirp4netns` is deprecated. Bridging a rootless container to a host service (Ollama) often fails due to SELinux and network namespaces. **The only reliable method is using `--network=host`.**

### Installation Steps

Install Podman:

- **Fedora/RHEL/CentOS:** `sudo dnf install -y podman`
- **Ubuntu:** `sudo apt install -y podman`

Start Open WebUI (run as your normal user, **not** root):

```bash
podman run -d \
  --name open-webui \
  --network=host \
  -v open-webui:/app/backend/data:Z \
  -e OLLAMA_BASE_URL=http://127.0.0.1:11434 \
  --restart=always \
  ghcr.io/open-webui/open-webui:main
```

> **Note:** The `:Z` at the end of the volume mount is strictly required for SELinux (Fedora/RHEL/CentOS).

### Access the UI

Because we used `--network=host`, you must access the WebUI on port **8080**:

👉 **http://localhost:8080**

---

## 4. Troubleshooting Networking & SELinux

**If the WebUI shows "No models available":**

### Verify Container-to-Host Connectivity

```bash
podman exec -it open-webui curl -s http://127.0.0.1:11434/api/tags
```

If this returns JSON, the connection is perfect.

### Reset Stale Configuration

Open WebUI caches connection settings. If you previously tried a different IP/port, it will ignore your new variables. Force a reset:

```bash
podman rm -f open-webui
# Re-run the podman command, but add: -e RESET_CONFIG_ON_START=true
# After models appear, restart without that flag.
```

### Firewall (if accessing from another machine)

- **Fedora/RHEL/CentOS:**
  ```bash
  sudo firewall-cmd --permanent --add-port=8080/tcp && sudo firewall-cmd --reload
  ```
- **Ubuntu:**
  ```bash
  sudo ufw allow 8080/tcp
  ```

---

## 5. Configuring the AI Persona (System Prompt)

Do not prompt the AI from scratch every time. Create a **Custom Model**:

1. Go to **Workspace > Models > + Create Model**.
2. **Base Model:** Select `qwen2.5-coder:14b`.
3. **Name:** `SysAdmin Copilot`.
4. **System Prompt:** Paste the following:

```text
You are an expert Senior IT System and Network Administrator with 15+ years of
experience in Linux (Fedora, RHEL, Ubuntu), Windows Server, networking (Cisco,
pfSense), and cloud infrastructure (AWS/Azure).

Your goal is to help the user by:
1. Providing robust, secure, and production-ready scripts (Bash, PowerShell, Python, Ansible).
2. Troubleshooting complex network and system issues with step-by-step diagnostics.
3. Suggesting modern automation workflows and best practices.
4. Always explaining your reasoning before providing code.
5. Highlighting security implications and potential risks in your solutions.

When writing code, use clean formatting, add comments, and handle errors
properly. Do not provide overly simplistic answers; treat the user as a peer.
```

5. Click **Save**.

---

## 6. Creating Custom Python Tools (Function Calling)

You can give your AI the ability to generate and format scripts explicitly by creating a **Tool**.

1. Go to **Workspace > Tools > + Create Tool**.
2. **Name:** `Shell Command Proposer`.
3. Paste this exact code:

```python
"""
title: Shell Command Proposer
author: IT Admin
version: 1.0.0
required_open_webui_version: 0.4.0
"""

import json
from pydantic import BaseModel, Field


class Tools:
    def __init__(self):
        self.valves = self.Valves()

    class Valves(BaseModel):
        safety_note: str = Field(
            "⚠️ ATTENTION : Vérifiez toujours cette commande avant de l'exécuter en production.",
            description="Avertissement de sécurité.",
        )

    async def propose_shell_command(self, task: str) -> str:
        """
        Propose une commande shell sécurisée.
        """
        return json.dumps(
            {
                "task": task,
                "instruction": "Génère la commande shell exacte pour accomplir cette tâche.",
                "warning": self.valves.safety_note,
            },
            indent=2,
        )
```

4. **Save.** Then go to your chat, select your **SysAdmin Copilot** model, open the settings (sliders icon), and **enable the tool**.

---

## 7. Adding Custom Knowledge (RAG)

To add a GitHub repository (e.g., `awesome-reverse-engineer-anything`) to your AI's permanent memory:

1. Clone the repo locally and locate the `.md` files:

   ```bash
   git clone https://github.com/AIshihzong/awesome-reverse-engineer-anything.git
   ```

2. In Open WebUI, go to **Workspace > Knowledge > + Create Knowledge**.
3. **Name:** `Reverse Engineering Repo`.
4. Drag and drop the `.md` files (and any documentation).
5. Open WebUI will use `nomic-embed-text` to vectorize the text.
6. Go to **Workspace > Models > Edit your SysAdmin Copilot**.
7. Scroll down to **Knowledge** and add `Reverse Engineering Repo`.
8. **Save.** The AI will now autonomously query this repository when asked relevant questions.

---

## 8. Maintenance & Updates

To safely update Open WebUI **without losing** your chats, knowledge base, or personas:

```bash
# 1. Pull the latest image
podman pull ghcr.io/open-webui/open-webui:main

# 2. Remove the old container (data is safe in the volume)
podman rm -f open-webui

# 3. Start the new container using the exact same command
podman run -d \
  --name open-webui \
  --network=host \
  -v open-webui:/app/backend/data:Z \
  -e OLLAMA_BASE_URL=http://127.0.0.1:11434 \
  --restart=always \
  ghcr.io/open-webui/open-webui:main
```

### Monitoring Performance

Keep a terminal open and run:

```bash
watch -n 1 nvidia-smi
```

When chatting with the AI, you should see VRAM usage spike. If VRAM exceeds 11GB, consider downgrading to a smaller quantization (e.g., `qwen2.5-coder:7b`) to prevent system RAM offloading (which drastically reduces speed).
