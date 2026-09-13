# SnapOtter Docker Setup Guide

This guide provides the necessary commands to set up and run SnapOtter with GPU support and expose it via a Cloudflare Tunnel.

## 1. Verify GPU Access in Docker
Run the following command to ensure Docker can successfully communicate with your NVIDIA GPU:
```bash
docker run --rm --gpus all ubuntu nvidia-smi
```

## 2. Create a Persistent Volume
Create a Docker volume to store your SnapOtter data persistently across container restarts:
```bash
docker volume create SnapOtter-data
```

Verify that the volume was created successfully by listing your volumes and looking for `SnapOtter-data`:
```bash
docker volume ls
```

## 3. Run the SnapOtter Container
Start the SnapOtter container in detached mode, exposing port 1349, allocating all GPUs, and mounting the persistent data volume:
```bash
docker run -d --name SnapOtter --gpus all -p 1349:1349 -v SnapOtter-data:/data snapotter/snapotter:latest
```

Run the following command in your terminal to see a list of your currently active containers and confirm `SnapOtter` is running:
```bash
docker ps
```

## 4. Expose via Cloudflare Tunnel
To make the application accessible externally:
1. Go to your Cloudflare Zero Trust dashboard (or your cloudflared tunnel configuration).
2. Add a new Public Hostname mapping your desired domain name to the local service URL:
   - **URL:** `http://localhost:1349/`