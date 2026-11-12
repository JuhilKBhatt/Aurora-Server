# OpenClaw Isolated VM & Cloudflare Tunnel Setup Guide

This guide documents the complete step-by-step setup for running **OpenClaw** inside an isolated Ubuntu Multipass VM on the `aurora` host server, air-gapped from the local LAN via UFW, and exposed securely via a Cloudflare Tunnel.

## 1. Prepare the Host Server (`aurora`)

### Check CPU Virtualization Support

Ensure hardware virtualization is enabled on the host:

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```

> **Verification:** Confirm the output number is `1` or higher.

### Install Multipass

```bash
sudo snap install multipass
```

> **Verification:** Run `multipass version` to confirm the daemon is running.

### Launch the Isolated Ubuntu VM

Create an Ubuntu 24.04 LTS virtual machine with 2 CPUs, 6GB RAM, and a 40GB disk:

```bash
multipass launch 24.04 --name openclaw-vm --cpus 2 --memory 6G --disk 40G
```

> **Verification:** Run `multipass list` and verify `openclaw-vm` shows a `Running` state with a private internal IPv4 address.

### Isolate the VM from the Local LAN (UFW)

Block the Multipass virtual bridge (`mpqemubr0`) from routing traffic to your local network subnets (`eno1` is the host's primary interface from `ip a`):

```bash
sudo ufw route deny in on mpqemubr0 out on eno1 to 192.168.0.0/16
sudo ufw route deny in on mpqemubr0 out on eno1 to 10.0.0.0/8
```

## 2. Install OpenClaw Inside the VM

### Enter the VM Shell

```bash
multipass shell openclaw-vm
```

> **Verification:** Your terminal prompt changes to `ubuntu@openclaw-vm:~$`.

### Run the Official OpenClaw Installer

Download and execute the installer script (installs the required Node.js runtime and OpenClaw CLI):

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

* Follow the initial terminal prompts.
* When it reaches **`Waiting for the dashboard to connect...`** (go to Web UI), press **`Ctrl + C`** to exit back to the VM terminal.

### Ensure the Background Daemon is Installed & Persistent

If exiting the wizard early left the background service disabled, enable user lingering and install the gateway service:

```bash
sudo loginctl enable-linger ubuntu
openclaw gateway install
openclaw gateway start
```

## 3. Configure Cloudflare Tunnel

1. Log in to the **Cloudflare Zero Trust** dashboard and navigate to **Networks > Tunnels**.
2. Click **Create a tunnel** -> Select **Cloudflared** -> Enter a tunnel name.
3. Follow the Debian/Ubuntu installation commands shown in Cloudflare inside your `openclaw-vm` shell to install `cloudflared` and register the service token.
4. Go to the **Public Hostname (Routes)** tab and click **Add a public hostname / route**:
   * **Subdomain:** `openclaw-aurora` *(Note: Use a hyphen instead of a dot so Cloudflare's free Universal SSL covers the certificate)*
   * **Domain:** `juhilkbhatt.com`
   * **Service Type:** `HTTP`
   * **URL:** `localhost:18789` *(or `127.0.0.1:18789`)*

> **Security Tip:** Add a **Cloudflare Access** application policy on `openclaw-aurora.juhilkbhatt.com` so visitors must authenticate with your email before reaching the OpenClaw gateway.

## 4. Configure OpenClaw Gateway for Cloudflare Proxy

Inside `ubuntu@openclaw-vm:~$`, configure OpenClaw to trust the local Cloudflare proxy headers, allow your domain origin, and set your gateway authentication token:

```bash
openclaw config set gateway.trustedProxies '["127.0.0.1", "::1", "127.0.0.1/8"]'
openclaw config set gateway.publicOrigin "https://openclaw-aurora.juhilkbhatt.com"
openclaw config set gateway.controlUi.allowedOrigins '["https://openclaw-aurora.juhilkbhatt.com"]'
openclaw config set gateway.auth.mode "token"
openclaw config set gateway.auth.token "MySecretPass2026"
```

*(Replace `MySecretPass2026` with your actual strong secret token).*

### Restart the Gateway Service

```bash
openclaw gateway restart
```

## 5. Access the Web UI & Pair Browser

### Open the Dashboard

Open your web browser and navigate to your tokenized URL:

```text
https://openclaw-aurora.juhilkbhatt.com/#token=MySecretPass2026
```

### Approve Browser Device Pairing (First-Time Login)

If the dashboard displays `Pairing required` or `Disconnected`, approve the pending browser connection inside `openclaw-vm`:

```bash
openclaw devices list
openclaw devices approve <request-id>
```

## 6. Agent Setup

### Model Provider Configuration (Gemini API)

Configure your Google AI Studio API key and default model:

```bash
# Add your Gemini API key or Web UI steps here
openclaw config set models.providers.google.apiKey "YOUR_GEMINI_API_KEY"
```

### Messaging Channel Setup (Telegram / Discord / WhatsApp)

Connect your messaging bot and approve pairing:

```bash
# Add your channel setup steps and pairing commands here
openclaw pairing list
openclaw pairing approve <pairing-id>
```

### Agent Identity, Memory & Workspace (`SOUL.md` / `HEARTBEAT.md`)

Document custom instructions, skills, and scheduled tasks:

* **Workspace Path:** `~/clawd` (or `~/.openclaw/workspace`)
* **Personality (`SOUL.md`):** *(Add notes on agent persona and rules)*
* **Scheduled Tasks (`HEARTBEAT.md`):** *(Add notes on cron intervals and background tasks)*

## 7. Useful Maintenance & Snapshot Commands

* **Check Gateway Status (inside VM):**

  ```bash
  openclaw gateway status
  openclaw doctor
  ```

* **Lock Down Config File Permissions (inside VM):**

  ```bash
  chmod 700 ~/.openclaw
  chmod -R 600 ~/.openclaw/*
  ```

* **Take a Clean VM Snapshot (on `aurora` host):**
  Stop the VM briefly to take a recovery snapshot, then start it back up:

  ```bash
  multipass stop openclaw-vm
  multipass snapshot openclaw-vm --name clean-install
  multipass start openclaw-vm
  ```

* **Restore VM from Snapshot if Something Breaks (on `aurora` host):**

  ```bash
  multipass restore openclaw-vm.clean-install
  ```