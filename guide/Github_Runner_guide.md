# GitHub Actions Self-Hosted Runner & Docker Deployment Guide

This guide covers how to set up an automated deployment pipeline using a GitHub self-hosted runner and a restricted user account. This method securely pulls code and restarts Docker containers without requiring a public static IP, SSH keys, or giving root access to GitHub.

## Phase 1: Server Preparation (Permissions & User)

First, create a restricted user on your server specifically for running deployments. Run these commands as your main administrative user.

### 1. Create a Dedicated Deploy User
Create the `deployer` user without a password (since it will run as a background service):
```bash
sudo useradd -m -s /bin/bash deployer
```

### 2. Grant Project Ownership
Give the `deployer` user ownership of your project directory so it has permission to run `git pull`. 
*(Note: Replace the path below with your actual project directory if it is different).*
```bash
sudo chown -R deployer:deployer <PATH/TO/PROJECT>
```

### 3. Whitelist Docker Commands
Create a security rule so the `deployer` user can restart Docker containers without needing to type a `sudo` password. Using `$(which docker)` ensures the correct absolute path is used.
```bash
echo "deployer ALL=(ALL) NOPASSWD: $(which docker) compose *" | sudo tee /etc/sudoers.d/deployer
```

Verify the rule saved correctly:
```bash
cat /etc/sudoers.d/deployer
```
*(You should see an output like: `deployer ALL=(ALL) NOPASSWD: /usr/bin/docker compose *`)*

---

## Phase 2: Generate the Runner Token (GitHub)

Leave your terminal open and head over to your GitHub repository in a web browser.

1. Go to **Settings** > **Actions** > **Runners**.
2. Click the green **New self-hosted runner** button.
3. Select **Linux** as the Runner image and **x64** as the Architecture.
4. Switch to the `deployer` user and set up the runner directory:
```bash
sudo -u deployer bash
mkdir ~/actions-runner-<REPO-NAME> && cd ~/actions-runner-<REPO-NAME>
```
*Next, copy and paste the commands from the GitHub **Download** and **Configure** sections into this terminal.* 
When the configuration script asks for the runner name and labels, just press **Enter** to accept all the defaults.

5. You will need to copy the exact commands it provides under the "Download" (except mkdir and cd commands) and "Configure" sections, as they contain a temporary security token unique to your repo.

---

## Phase 3: Install & Configure the Runner (Server)

Head back to your server terminal to download and start the runner agent.

### 1. Install the Runner as a Background Service
To ensure the runner stays active and automatically restarts if the server reboots, you must install it as a system service. This step requires root privileges. 

Exit the `deployer` shell to return to your main user (`aurora`):
```bash
exit
```

Install the service for the `deployer` user:
```bash
sudo bash -c "cd /home/deployer/actions-runner-<REPO-NAME> && ./svc.sh install deployer"
```

Start the service:
```bash
sudo bash -c "cd /home/deployer/actions-runner-<REPO-NAME> && ./svc.sh start"
```

### 2. Verify the Runner is Active
Go back to your GitHub repository in your web browser. Navigate to **Settings** > **Actions** > **Runners**. You should now see your self-hosted runner listed with a green **Idle** status.

---

## Phase 4: Create the Deployment Pipeline

Now that the server is listening for jobs, add the workflow file to your codebase.

1. On your local computer, open your project.
2. Create or update the file at `.github/workflows/deploy.yml`.
3. Paste the following configuration:

```yaml
name: Auto Deploy
on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - name: Pull and Restart Docker
        run: |
          cd <PATH_TO_REPO>/<REPO_NAME>
          git pull origin main
          sudo docker compose down
          sudo docker compose up -d --build
```

**To test the pipeline:** Commit this file and push it to your `main` branch. Check the **Actions** tab in your GitHub repository to watch it automatically deploy your updated code!

---

## To Setup Another Github Runner


### 1. Grant Project Ownership
Give the `deployer` user ownership of your project directory so it has permission to run `git pull`. 
*(Note: Replace the path below with your actual project directory if it is different).*
```bash
sudo chown -R deployer:deployer <PATH/TO/PROJECT>
```

### 2. Set Up the New Runner (Second Repo)

Server side
For your new repository, switch to the deployer user and create the correctly named folder from the start (replace <REPO-NAME> with the actual name).

```bash
sudo -u deployer bash
mkdir ~/actions-runner-<REPO-NAME> && cd ~/actions-runner-<REPO-NAME>
```

Once inside this folder, paste the Download and Configure commands from your new GitHub repository's settings page.

### 3. Install the New Runner Service
Server side
Once the new runner is configured, type exit to return to your aurora user, then install and start it as a service using the new folder path.

```bash
exit
sudo bash -c "cd /home/deployer/actions-runner-<REPO-NAME> && ./svc.sh install deployer"
sudo bash -c "cd /home/deployer/actions-runner-<REPO-NAME> && ./svc.sh start"
```

**To test the pipeline:** Commit this file and push it to your `main` branch. Check the **Actions** tab in your GitHub repository to watch it automatically deploy your updated code!