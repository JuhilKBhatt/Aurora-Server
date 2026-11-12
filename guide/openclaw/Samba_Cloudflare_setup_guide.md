# OpenClaw VM File Access Setup Guide (Samba + Cloudflare Zero Trust)

This guide walks through setting up a secure Samba (SMB) network file share on an Ubuntu virtual machine running OpenClaw and routing it privately through a **Cloudflare Zero Trust Tunnel (WARP-to-Tunnel)**. This allows you to mount and edit your VM files natively across **Mac, Windows, and iPhone** from anywhere in the world without opening public firewall ports.

---

## Part 1: Gather Your VM Details

Connect to your Ubuntu VM via SSH and run these two commands to note down your exact system username and internal private IP address:

1. **Check your Linux username:**
   ```bash
   whoami
   ```

2. **Check your VM's internal private IP address:**
   ```bash
   hostname -I | awk '{print $1}'
   ```

---

## Part 2: Install and Configure Samba (SMB) on Ubuntu

1. **Install Samba:**
   ```bash
   sudo apt update
   sudo apt install samba -y
   ```

2. **Create a visible folder shortcut (symlink) for `.openclaw`:**
   Because macOS Finder and the iOS Files app strictly hide folders that start with a dot (`.`), Press Command + Shift + . (period) at the same time.

3. **Configure the Samba shares:**
   Open the Samba configuration file:
   ```bash
   sudo nano /etc/samba/smb.conf
   ```
   Scroll to the very bottom of the file and paste the following blocks (replace `<username>` with your actual Linux username):
   ```ini
   [OpenClawFiles]
      path = /home/ubuntu/.openclaw
      browsable = yes
      read only = no
      guest ok = no
      valid users = <username>
      hide dot files = no
   ```
   * **Why `path = /home/ubuntu/.openclaw`?** Mounting directly inside `.openclaw` bypasses the hidden root folder issue completely.
   * **Why `force user = ubuntu`?** Ensures any file you upload or edit from Mac, Windows, or iPhone retains proper Linux ownership.
   * **Why `hide dot files = no`?** Prevents Samba from tagging Linux dot-files with the Windows/SMB "Hidden" attribute.

   Save and exit (`Ctrl+O`, press `Enter`, then `Ctrl+X`).

4. **Set your Samba password:**
   Create the SMB login password for your username:
   ```bash
   sudo smbpasswd -a <username>
   ```

5. **Restart Samba to apply changes:**
   ```bash
   sudo systemctl restart smbd
   sudo systemctl status smbd
   ```

---

## Part 3: Create the Cloudflare Zero Trust Tunnel

1. Log in to the [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/).
2. Go to **Networks** -> **Tunnels** and click **Create a tunnel**.
3. Select **Cloudflared**, name your tunnel `openclaw-vm`, and click **Save tunnel**.
4. Under **Choose your environment**, select **Debian** (for Ubuntu), then copy and run the installation command shown in your VM's SSH terminal. This installs `cloudflared` as a persistent background `systemd` service.
5. Once the connector status shows **HEALTHY**, click **Next** and go to the **Private Network** (CIDR) tab inside your tunnel configuration.
6. Click **Add a private network** and enter your VM's private IP with a `/32` subnet mask in the **CIDR** box:
   ```text
   <VM_PRIVATE_IP>/32
   ```
7. Add a description (e.g., `OpenClaw VM Samba`) and click **Save**.

---

## Part 4: Configure Cloudflare Zero Trust Routing & Enrollment

### 1. Enable the TCP Proxy
1. In the Zero Trust left sidebar, go to **Settings** -> **Network**.
2. Scroll down to **Firewall**, toggle **Proxy** ON, and check the **TCP** box.

### 2. Configure Split Tunnels
By default, Cloudflare excludes private RFC-1918 IP ranges (like `10.0.0.0/8` and `192.168.0.0/16`) from entering the WARP tunnel.
1. Go to **Settings** -> **WARP Client**.
2. Under **Device settings**, click your **Default** profile -> **Configure** -> **Split Tunnels**.
3. Choose one of the two routing modes below:
   * **If using "Exclude IPs and domains" mode (Default):** Click **Manage**, find the subnet that covers your VM's IP (e.g., if your IP starts with `10.x.x.x`, find `10.0.0.0/8`), and click the **trash can icon** to delete it from the exclusion list. *(Note: Do NOT add `<VM_PRIVATE_IP>/32` to the Exclude list, or it will block the connection!)*
   * **If using "Include IPs and domains" mode:** Switch the radio button to **Include IPs and domains**, click **Manage**, and add `<VM_PRIVATE_IP>/32`. This routes *only* your OpenClaw VM traffic through Cloudflare WARP while leaving your regular internet traffic untouched.

### 3. Set Up Device Enrollment Permissions
To prevent the `"Enrollment request is invalid"` error when logging into the WARP client:
1. Go to **Settings** -> **WARP Client**.
2. Under **Device enrollment permissions**, click **Manage**.
3. Under the **Policies** tab, click **Add a policy**:
   * **Policy name:** `Allow My Devices`
   * **Action:** `Allow`
   * **Include -> Selector:** `Emails`
   * **Value:** Enter your email address.
   * Click **Save**.
4. Click the **Authentication** tab next to Policies and verify **One-time PIN** (or your preferred identity provider) is enabled.
5. Check your exact **Team Name** under **Settings** -> **Custom Pages** (it is the prefix before `.cloudflareaccess.com`).

---

## Part 5: Connect from Mac, Windows, and iPhone

### Step 1: Sign In to the Cloudflare Client
* **Mac & Windows:** Install [Cloudflare WARP](https://one.one.one.one/). Click the menu bar/tray icon -> **Gear Icon** -> **Preferences** -> **Account** -> **Login with Cloudflare Zero Trust**, enter your exact **Team Name**, and complete the browser authentication. Turn the switch to **Connected**.
* **iPhone:** Install **Cloudflare One Agent** from the App Store. Open the app, log in with your **Team Name**, complete the email PIN verification, and toggle the VPN switch ON.

### Step 2: Mount the Samba Share
Once connected to Zero Trust, mount your share using your VM's private IP (`<VM_PRIVATE_IP>`), username (`ubuntu`), and your Samba password:

* **On Mac:**
  1. Open **Finder** and press `Cmd + K`.
  2. Enter `smb://<VM_PRIVATE_IP>/OpenClawFiles` (or `smb://<VM_PRIVATE_IP>/UbuntuHome`).
  3. Click **Connect**, select **Registered User**, and enter your `ubuntu` credentials.
  4. *Tip:* Press `Cmd + Shift + .` (period) inside Finder anytime you want to toggle hidden dot-files on or off.

* **On Windows:**
  1. Open **File Explorer**, right-click **This PC**, and select **Map network drive...**
  2. Choose a drive letter and enter the folder path: `\\<VM_PRIVATE_IP>\OpenClawFiles`
  3. Check **Connect using different credentials**, click **Finish**, and enter `ubuntu` and your Samba password.
  4. *Tip:* In File Explorer, click **View** -> **Show** -> **Hidden items** to view dot-files.

* **On iPhone:**
  1. Open the native **Files** app.
  2. Tap the **three-dot menu** (`...`) in the top-right corner and select **Connect to Server**.
  3. Enter `smb://<VM_PRIVATE_IP>/OpenClawFiles` and tap **Connect**.
  4. Choose **Registered User**, enter `ubuntu` and your Samba password, and tap **Next**.

---

## Part 6: Quick Troubleshooting Reference

| Issue | Cause | Fix |
| :--- | :--- | :--- |
| `Failed to add entry for user` when running `smbpasswd` | The username entered does not exist as a Linux system user. | Run `whoami` to find your actual system username (e.g., `ubuntu`) and run `sudo smbpasswd -a ubuntu`. |
| `Enrollment request is invalid` in the WARP app | No Device Enrollment rule matches your email, or the Team Name has a typo. | Add an `Allow` policy for your email under **Settings -> WARP Client -> Device enrollment permissions**. |
| Connection times out when connecting to `smb://<VM_PRIVATE_IP>` | The IP is still excluded in Split Tunnels, or TCP Proxy is off. | Ensure `10.0.0.0/8` and `<VM_PRIVATE_IP>/32` are deleted from the **Exclude** Split Tunnel list, and toggle WARP off and on. |
| Drive mounts, but the window appears empty | You mounted an empty folder, or your OS cached the old share before `smb.conf` was updated. | Ensure `path = /home/ubuntu/.openclaw` and `hide dot files = no` are in `/etc/samba/smb.conf`, run `sudo systemctl restart smbd`, **eject/disconnect** the drive on your device, and reconnect. |