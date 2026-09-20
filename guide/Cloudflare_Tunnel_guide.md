# Cloudflare Tunnel Setup Guide

This guide walks through creating a secure Cloudflare Tunnel to expose a local service (like `ttyd` or a web server) to the internet without opening any inbound firewall ports.

## 1. Create a New Tunnel

First, initiate the tunnel creation process from your Cloudflare account.

1. Log in to the **Cloudflare Zero Trust Dashboard**.
2. Navigate to **Networks** -> **Tunnels** on the left-hand sidebar.
3. Click the **+ Create a tunnel** button.
4. Choose the standard Cloudflared setup and click **Next**.
5. **Enter a Tunnel Name** (e.g., "ubuntu-server-terminal") and click **Save tunnel**.

## 2. Install and Authenticate `cloudflared`

Next, you need to install the Cloudflare daemon on your server so it can establish an outbound connection to Cloudflare's edge network.

1. On the "Install and run a connector" page, **select your setup environment** (e.g., Linux, then choose Debian/Ubuntu and your architecture, typically 64-bit).
2. Cloudflare will generate a specific installation command for your tunnel. It will look something like this:
   ```bash
   curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb && sudo dpkg -i cloudflared.deb && sudo cloudflared service install <YOUR_UNIQUE_TOKEN>
   ```
3. Copy the exact command provided in the dashboard.
4. **Run the command** in your server's terminal to download the packages, install the service, and automatically start the tunnel.
5. Once the server connects, you will see a success message in the Cloudflare dashboard. Click **Next**.

## 3. Route the Tunnel to Your App

Finally, map a public domain name to the local port running on your server.

1. Go back to the **Tunnels** page and click on your newly created tunnel.
2. Click on the **Public Hostname** (or Routes) tab at the top of the tunnel configuration page.
3. Click **+ Add a public hostname** (or "+ Add Route" -> "Published Application").
4. **Configure the Public Hostname:**
   * **Subdomain:** Enter the prefix you want to use (e.g., `terminal`).
   * **Domain:** Select your registered domain from the dropdown (e.g., `yourdomain.com`).
5. **Configure the Service:**
   * **Type:** Select `HTTP`.
   * **URL:** Enter `localhost:<port number>` (e.g., `localhost:3889` for your ttyd setup).
6. Click **Save hostname**.

Your local service is now securely published to the web. Visiting `https://terminal.yourdomain.com` will route directly to the local port on your server.