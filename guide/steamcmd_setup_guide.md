# SteamCMD Ubuntu Server Setup & Download Guide

This guide covers how to set up an Ubuntu server to download Windows Steam games in the background, directly to an existing mounted drive.

## 1. Initial Storage Setup
*Assuming your drive is already mounted at `/mnt/database`.*

List all the disks to verify:
```bash
lsblk
```

Create the directory for the games and take ownership (so you don't need root to download):
```bash
mkdir -p /mnt/database/steam_games
sudo chown -R $USER:$USER /mnt/database/steam_games
```

## 2. Install Required Packages
SteamCMD is a 32-bit application, so you must enable the 32-bit architecture and multiverse repository first.

```bash
sudo dpkg --add-architecture i386
sudo add-apt-repository multiverse
sudo apt update
sudo apt install steamcmd tmux samba
```
*Note: During the SteamCMD install, a purple screen will appear asking you to accept the license agreement. Use the **Tab** key to select "OK/I Agree" and press **Enter**.*

## 3. Start a Background Session
To ensure the download doesn't stop if your SSH connection drops, start a persistent terminal session using `tmux`.

```bash
tmux new -s steam
```

## 4. Launch and Configure SteamCMD
Launch the program:
```bash
steamcmd
```

Once you see the `Steam>` prompt, run these commands in order:

```text
# Limit download speed to 2 MB/s (2000 KB/s) to save network bandwidth
set_download_throttle 2000

# Force Steam to download Windows files instead of Linux files
@sSteamCmdForcePlatformType windows

# Tell Steam where to save the files
force_install_dir /mnt/database/steam_games/[GAME NAME AS IN STEAM]

# Log into your account (requires Steam Guard code on first login)
login your_steam_username

# Start the download (replace YOUR_APP_ID with the actual game ID from steamdb.info)
app_update YOUR_APP_ID validate
```

## 5. Managing the Background Session

**To Detach (Leave it running in the background):**
Once the download begins, press **Ctrl+B**, let go of both keys, and then press **D**. You can now safely close your SSH connection.

**To Reattach (Check progress later):**
Log back into your server and run:
```bash
tmux attach -t steam
```
*(When the download finishes, type `quit` to exit SteamCMD, and `exit` to close the tmux session).*

## 6. SAMBA Setup

Open the Samba configuration file to share the directory over your network:
```bash
sudo nano /etc/samba/smb.conf
```

Scroll to the bottom and paste this block:
```ini
[steam_games]
path = /mnt/database/steam_games
valid users = aurora
read only = no
```
*(Save and exit by pressing **Ctrl+O**, **Enter**, then **Ctrl+X**)*

Set a Samba password for your user:
```bash
sudo smbpasswd -a aurora
```

Restart the service to apply changes:
```bash
sudo systemctl restart smbd
```

## 7. Game Files Setup

1. Open Windows File Explorer and navigate to your main Steam library folder. By default, this is located at `C:\Program Files (x86)\Steam\steamapps\common`.
2. Paste the game folder you copied from your Ubuntu server directly into this `common` folder.
3. Open the Steam app on your PC, go to your **Library**, and select the game you just transferred. Click the blue **Install** button.
4. Ensure the "Install under" location exactly matches the drive where you just pasted the game files (e.g., your `C:` drive). 
5. Click **Install**. Steam will discover the existing files instead of downloading the game from scratch.