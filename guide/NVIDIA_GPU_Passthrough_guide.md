# Guide: Setting Up NVIDIA GPU Passthrough for Docker on Ubuntu

This guide covers the end-to-end process of configuring an NVIDIA GPU on an Ubuntu server to be used for video rendering and hardware acceleration inside Docker containers.

---

## Step 1: Install the NVIDIA Host Drivers

Before Docker can use the GPU, the host operating system needs the proprietary NVIDIA drivers installed.

```bash
sudo apt update
# Install the driver (version 535 or 580 are recommended for Maxwell/newer cards)
sudo apt install -y nvidia-driver-580 nvidia-smi
```

## Step 2: Reboot the Server

The new NVIDIA kernel modules will not actively load until the operating system restarts.

```bash
sudo reboot
```

## Step 3: Verify the Driver Installation

Once logged back in, verify that the GPU is communicating with the driver:

```bash
nvidia-smi
```
*You should see a table displaying your GPU details, driver version, and current memory usage.*

---

## Step 4: Add the NVIDIA Container Toolkit Repositories

To allow Docker to communicate with the physical GPU, you need the NVIDIA Container Toolkit. First, add the secure repositories:

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
```

## Step 5: Install the Toolkit

Install the actual software package:

```bash
sudo apt-get install -y nvidia-container-toolkit
```

## Step 6: Configure Docker Runtime

Tell Docker to register the NVIDIA runtime and restart the background service:

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

---

## Step 7: Test GPU Passthrough in Docker

Launch a temporary test container to ensure Docker can successfully execute commands on your GPU:

```bash
docker run --rm --gpus all ubuntu nvidia-smi
```

*If this outputs the exact same `nvidia-smi` table you saw in Step 3, your setup is complete!*

## Next Steps
You can now append `--gpus all` (or configure it in your `docker-compose.yml`) to deploy GPU-accelerated applications like Plex, Jellyfin, or custom FFmpeg rendering containers.
