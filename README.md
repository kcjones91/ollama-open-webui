# OpenWebUI + Ollama Deployment with Podman Compose

## Overview
This setup deploys **OpenWebUI** and **Ollama** using **Podman Compose**, ensuring GPU acceleration for AI model execution. The configuration includes resource limits, security best practices, and an optional **Nginx reverse proxy**.

## Prerequisites
Ensure the following are installed on your system:
- **Podman**
- **Podman Compose**
- **NVIDIA Drivers** (for GPU acceleration)
- **NVIDIA Container Toolkit** (if applicable)
- **SELinux Enabled** (for security)

## Installation

### 1. Install Podman Compose
```bash
pip install podman-compose
```

### 2. Clone Repository and Set Up Directories
```bash
mkdir -p /opt/openwebui/{ollama,data}
```

### 3. Set Permissions
```bash
sudo useradd -r -s /sbin/nologin podmanuser
sudo chown -R podmanuser:podmanuser /opt/openwebui
sudo chmod -R 750 /opt/openwebui
sudo restorecon -Rv /opt/openwebui
```

### 4. Create `podman-compose.yml`
```yaml
version: "3"

services:
  ollama:
    image: docker.io/ollama/ollama:latest
    container_name: ollama
    restart: always
    ports:
      - "11434:11434"
    deploy:
      resources:
        limits:
          memory: "8g"
          cpus: "4"
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    volumes:
      - "/opt/openwebui/ollama:/root/.ollama:Z"
    user: "1001:1001"

  openwebui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: openwebui
    restart: always
    ports:
      - "4000:4000"
    depends_on:
      - ollama
    deploy:
      resources:
        limits:
          memory: "4g"
          cpus: "2"
    volumes:
      - "/opt/openwebui/data:/app/backend/data:Z"
    environment:
      - PORT=4000
      - WEBUI_URL=http://localhost:4000
      - OLLAMA_API_BASE_URL=http://ollama:11434
    user: "1001:1001"
```

### 5. Deploy Containers
```bash
podman-compose up -d
```

## Verification

### Check Running Containers
```bash
podman ps -a
```
Expected Output:
```
CONTAINER ID  IMAGE                               STATUS         PORTS               NAMES
xxxxxx        docker.io/ollama/ollama:latest      Up            11434->11434/tcp    ollama
yyyyyy        ghcr.io/open-webui/open-webui:main  Up            3000->3000/tcp      openwebui
```

### Verify GPU Acceleration
```bash
podman exec -it ollama ollama ps
```
Expected Output:
```
NAME                  ID              SIZE      PROCESSOR    UNTIL              
deepseek-r1:latest    6.0 GB          100% GPU  4 minutes from now    
```

If GPU is not detected, check inside the container:
```bash
podman exec -it ollama bash
ls /dev/dri/
```

### Test OpenWebUI Access
Open a browser and navigate to:
```
http://localhost:3000
```
Select a model (e.g., `deepseek-r1`) and run a test query.

## Optional: Nginx Reverse Proxy
If running OpenWebUI remotely, configure an **Nginx reverse proxy**:

1. Install Nginx:
```bash
sudo dnf install nginx -y
```

2. Create `/etc/nginx/conf.d/openwebui.conf`:
```nginx
server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

3. Restart Nginx:
```bash
sudo systemctl restart nginx
```

Now OpenWebUI is accessible at:
```
http://localhost
```

## Conclusion
This guide sets up **OpenWebUI and Ollama using Podman Compose** with **GPU acceleration, resource limits, and SELinux security**. For remote access, **Nginx can be configured as a reverse proxy**.

For further debugging, use:
```bash
sudo journalctl -t setroubleshoot | tail -20
```
