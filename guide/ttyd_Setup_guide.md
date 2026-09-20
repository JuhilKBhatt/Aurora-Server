# ttyd Installation and Setup Guide

This guide covers the installation, systemd configuration, and troubleshooting steps for setting up `ttyd` as a web-based terminal, specifically configured to work behind a Cloudflare tunnel on port 3889.

## 1. Download and Install the Binary

Just like GoTTY, `ttyd` is distributed as a single executable file. We will download the latest 64-bit static build, make it executable, and move it to the system binaries.

```bash
wget https://github.com/tsl0922/ttyd/releases/latest/download/ttyd.x86_64
chmod +x ttyd.x86_64
sudo mv ttyd.x86_64 /usr/local/bin/ttyd
```

*To verify it installed correctly, run `ttyd --version`. It should output the current version number.*

## 2. Create the Systemd Service

We will create a background service file to connect `ttyd` to `/bin/login` on port 3889. 

Open the service file:
```bash
sudo nano /etc/systemd/system/ttyd.service
```

Paste the following configuration. 
> **Note:** `ttyd` uses a capital `-W` flag to allow writable client connections. The `-P 15` flag sends a ping every 15 seconds to prevent Cloudflare from dropping the connection due to inactivity.

```ini
[Unit]
Description=ttyd Terminal Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/ttyd -W -P 15 -p 3889 /bin/login
Restart=always
User=root
Group=root

[Install]
WantedBy=multi-user.target
```
*Save the file and exit your text editor.*

## 3. Reload, Enable, and Start

Tell the system to recognize the new service, set it to start automatically when the server boots, and start it immediately:

```bash
sudo systemctl daemon-reload
sudo systemctl enable ttyd
sudo systemctl start ttyd
```

*To verify it is running perfectly, run `systemctl status ttyd`. You should see "active (running)" in green text.*

Because we used port `3889`, your existing Cloudflare tunnel should automatically pick up the new service. When you visit your web terminal URL, you will be greeted by the xterm.js interface, followed immediately by your standard system username, password, and 2FA prompts.

---

## 4. Troubleshooting & Network Rules

If you are having trouble connecting locally, use these steps to verify port bindings and firewall rules.

### Check Network Bindings
If the service is running, use `netstat` to see exactly which port `ttyd` is actively listening on.

```bash
sudo netstat -tulnp | grep ttyd
```
*   **Expected output:** `tcp 0 0 0.0.0.0:3889 0.0.0.0:* LISTEN 12345/ttyd`
*   If the command returns nothing, `ttyd` is not listening on any port.
*   If you see `127.0.0.1:3889` instead of `0.0.0.0:3889` or `:::3889`, it means `ttyd` is only bound to localhost, which is why a local IP connection will fail.

### Check Local Firewall Rules
If `netstat` confirms it is listening on `0.0.0.0:3889` (all interfaces), your Ubuntu server's firewall might be blocking the connection from your local IP.

Check the Uncomplicated Firewall (UFW) status:
```bash
sudo ufw status
```

If UFW is active and you do not see port `3889` listed in the allowed rules, allow it temporarily to test your local connection:
```bash
sudo ufw allow 3889/tcp
```

---

## 5. Prevent Inactivity Disconnects

In addition to the `-P 15` ping parameter in the systemd service (which handles Cloudflare timeouts), you should ensure Linux itself isn't killing your idle sessions.

### Disable the Linux Shell Timeout
Linux distributions often use the `TMOUT` environment variable to automatically close idle shells. First, check if it is active:

```bash
echo $TMOUT
```

If it returns a number (like `300` or `900`), your shell is killing the session after that many seconds. Disable it permanently for your user by running:

```bash
echo "unset TMOUT" >> ~/.bashrc
```
*(Log out and log back in, or run `source ~/.bashrc`, for this change to take effect).*