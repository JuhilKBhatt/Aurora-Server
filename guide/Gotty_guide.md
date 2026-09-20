# GoTTY with Two Factor Authentication

This guide walks you through securing a web-based terminal (GoTTY) using Google Authenticator for Two-Factor Authentication (2FA) on an Ubuntu server.

## Step 1: System Updates and Prerequisites

First, execute updates on the new instance and install essential networking tools.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install net-tools -y
```

## Step 2: Create a Dedicated User

Add a user account for testing and grant it `sudo` privileges:

```bash
sudo adduser <user>
sudo usermod -aG sudo <user>
```

## Step 3: Install and Configure Google Authenticator for 2FA

Install the Google pluggable authentication module (PAM) for 2FA:

```bash
sudo apt install libpam-google-authenticator -y
```

Switch over to your new user account, as this is where we want to add 2FA, and invoke the 2FA key generator:

```bash
su - <user>
google-authenticator
```
*(Follow the on-screen prompts, scan the QR code with your authenticator app, and securely save your backup codes.)*

## Step 4: Configure PAM to Enforce 2FA

Edit the common authentication file to require 2FA:

```bash
sudo nano /etc/pam.d/common-auth
```

Insert the following two lines at the end of the file, then save and exit:

```text
auth required pam_google_authenticator.so nullok
auth required pam_permit.so
```

## Step 5: Configure SSH Server

Edit the SSH server configuration file to ensure it prompts for the 2FA challenge:

```bash
sudo nano /etc/ssh/sshd_config
```

Depending on your Ubuntu version, find and change the appropriate key to `yes`:

*   **For Ubuntu 22.04:**
    ```text
    KbdInteractiveAuthentication yes
    ```
*   **For Ubuntu 20.04:**
    ```text
    ChallengeResponseAuthentication yes
    ```

Restart the SSH service for the changes to take effect:

```bash
sudo systemctl restart sshd.service
```

## Step 6: Synchronize System Time

2FA authentication relies heavily on an accurate system clock. Set your system to get the time from the internet and set your specific timezone:

```bash
sudo timedatectl set-ntp yes
sudo timedatectl list-timezones
sudo timedatectl set-timezone <your-timezone>
```

*Note: At this point, you should be able to SSH to your server and be prompted for a username, password, and 2FA key. The `sudo` command will now also prompt for a 2FA key.*

---

## Step 7: Install GoTTY Web Terminal

Now, install GoTTY to serve the terminal over the web.

Download the latest release, extract it, and move it to your binaries folder:

```bash
wget https://github.com/yudai/gotty/releases/download/v2.0.0/gotty_linux_amd64.tar.gz
tar -xzf gotty_linux_amd64.tar.gz
sudo mv gotty /usr/local/bin/
rm gotty_linux_amd64.tar.gz
```

*(Optional)* To test GoTTY immediately on port 3002, you can run:
```bash
sudo gotty -w -p 3002 /bin/login
```
*Press `Ctrl+C` to stop the test.*

## Step 8: Create a Systemd Service for GoTTY

To ensure GoTTY runs continuously in the background and starts on boot, create a systemd service file:

```bash
sudo nano /etc/systemd/system/gotty.service
```

Paste the following configuration into the file:

```ini
[Unit]
Description=GoTTY Terminal Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/gotty -w -p 3889 /bin/login
Restart=always
User=root
Group=root

[Install]
WantedBy=multi-user.target
```

Save and exit. Then, reload systemd, enable, and start the GoTTY service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable gotty
sudo systemctl start gotty
```

## Step 9: Secure with Cloudflare Zero Trust

Finally, navigate to **Cloudflare Zero Trust** to set up a secure tunnel to your local port `3889` (or whichever port you configured in the service file) to expose the terminal securely without opening inbound firewall ports.