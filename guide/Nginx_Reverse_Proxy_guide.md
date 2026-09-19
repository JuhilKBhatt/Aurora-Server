# Nginx Reverse Proxy Guide

A step-by-step reference for configuring Nginx reverse proxying, port assignments, and Docker deployments on your Ubuntu server.

## Mental Model: How Nginx & Docker Work Together

Your Ubuntu server has a single public-facing **Host Nginx** listening on ports **80** (HTTP) and **443** (HTTPS). 

Instead of opening container ports directly to the internet, Host Nginx inspects the incoming domain name (e.g. `abc.domain.com` vs `zxy.domain.com`) and routes the traffic internally to the correct local port on `127.0.0.1`.

---

## Setting Up `<project-name>.domain.com`

### Step 1: Start the Docker Stack on Port X

Verify that the container is up and listening on port X:
```bash
docker ps
```
You should see:
```text
0.0.0.0:X->80/tcp   <container_name>
```

Test it locally from the server terminal:
```bash
curl -I http://127.0.0.1:X
```
*(Should return `HTTP/1.1 200 OK`)*

---

### Step 2: Create Host Nginx Configuration

Create the site configuration in `/etc/nginx/sites-available/`:
```bash
sudo nano /etc/nginx/sites-available/<project-name>
```

Paste the following:
```nginx
server {
    listen 80;
    server_name <project-name>.domain.com;

    location / {
        proxy_pass http://127.0.0.1:X;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Save and exit (`Ctrl+O` → `Enter` → `Ctrl+X`).

---

### Step 3: Enable the Site & Reload Nginx

In Nginx on Debian/Ubuntu, configs in `sites-available` are enabled by creating a symbolic link in `sites-enabled`:

```bash
# 1. Create the symlink
sudo ln -s /etc/nginx/sites-available/<project-name> /etc/nginx/sites-enabled/

# 2. Test syntax (MUST see "test is successful")
sudo nginx -t

# 3. Reload Nginx without downtime
sudo systemctl reload nginx
```
