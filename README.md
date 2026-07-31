# Real-Time Chat App — Dockerized Deployment with NGINX & CI/CD

## 1. Project Overview

This project is a real-time WebSocket chat application, containerized with Docker and deployed behind an NGINX reverse proxy on an AWS EC2 instance. The application was provided with a deliberately broken deployment configuration; this repository contains the debugged, working setup along with an automated CI/CD pipeline using GitHub Actions.

**Stack:**
- **Backend:** Python FastAPI, WebSocket support on `/ws`
- **Frontend:** Static single-page HTML client
- **Reverse proxy:** NGINX (serves frontend + proxies WebSocket traffic)
- **Orchestration:** Docker Compose
- **Deployment:** AWS EC2 (free tier)
- **CI/CD:** GitHub Actions (auto-deploy on push to `main`)

**Live URL:** `http://13.126.4.97`

---

## 2. Architecture Diagram

```
                     ┌─────────────────────────────────────────┐
                     │              AWS EC2 Instance             │
                     │         (Public IP, Port 80 open)          │
                     │                                             │
   User Browser      │   ┌───────────────────────────────────┐    │
  ─────────────────► │   │      Docker Compose Network        │    │
   http://<EC2-IP>    │   │                                     │    │
        │            │   │  ┌───────────────┐                  │    │
        └────────────┼──►│  │ nginx container│  host 80 → 80    │    │
          port 80    │   │  │  (public-facing)│                 │    │
                     │   │  └───────┬────────┘                 │    │
                     │   │          │ proxy_pass                │    │
                     │   │          │ http://backend:8000/ws    │    │
                     │   │          ▼                            │    │
                     │   │  ┌────────────────┐                  │    │
                     │   │  │ backend container│ (FastAPI)       │    │
                     │   │  │  0.0.0.0:8000    │ (internal only) │    │
                     │   │  └────────────────┘                  │    │
                     │   └───────────────────────────────────┘    │
                     └─────────────────────────────────────────┘
```

- Only the `nginx` container is exposed to the internet (`80:80`).
- The `backend` container is **not** published to the host — it's reachable only inside the Docker Compose network, by its service name (`backend`).
- Two containers total, always: `chat-nginx` and `chat-backend`.

---

## 3. How Docker Containers Are Set Up

The app runs as two services defined in `docker-compose.yml`:

| Container | Image/Build | Port Mapping | Role |
|---|---|---|---|
| `chat-backend` | Built from local `Dockerfile` | `expose: 8000` (internal only) | Runs the FastAPI app via `uvicorn` |
| `chat-nginx` | `nginx:alpine` | `80:80` (published to host) | Serves frontend static files, reverse-proxies WebSocket traffic |

Both containers are set to `restart: always`, so they automatically restart if they crash or if the EC2 instance reboots.

The backend's `Dockerfile`:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY app/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app/main.py .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## 4. How Docker Networking Works

Docker Compose automatically creates a private bridge network shared by all services defined in the same `docker-compose.yml`. Within this network:

- Every container can reach every other container using the **service name** as a hostname (Docker's built-in DNS resolves `backend` to the backend container's internal IP).
- `localhost` inside a container refers only to **that container itself** — not the host machine, and not sibling containers. This was the root cause of two of the three bugs in this assignment (see Section 7).
- Only ports explicitly listed under `ports:` (not `expose:`) are reachable from outside Docker — this is why `nginx` is reachable from the internet but `backend` is not.

---

## 5. How NGINX Reverse Proxy Works

NGINX serves two distinct roles in `nginx.conf`:

**a) Static file server** — the `location /` block serves the built frontend files, mounted into the container via a volume (`./frontend:/usr/share/nginx/html:ro`):
```nginx
location / {
    root /usr/share/nginx/html;
    index index.html;
    try_files $uri $uri/ /index.html;
}
```

**b) Reverse proxy** — the `location /ws` block forwards WebSocket requests to the backend container over the internal Docker network:
```nginx
location /ws {
    proxy_pass http://backend:8000/ws;
    ...
}
```

This means the browser only ever talks to nginx on port 80 — it never talks to the backend directly. NGINX decides, based on the request path, whether to return a static file or tunnel the connection through to FastAPI.

---

## 6. How WebSocket Works Through NGINX

WebSocket connections start as a normal HTTP request that asks to be "upgraded" to a persistent, full-duplex connection. By default, NGINX proxies requests as plain HTTP/1.0 and does not forward upgrade headers — so without extra configuration, the WebSocket handshake silently fails.

To support it, the proxy block includes:
```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```
- `proxy_http_version 1.1` — WebSocket upgrades require HTTP/1.1; the proxy default (1.0) doesn't support them.
- `Upgrade $http_upgrade` — forwards the client's upgrade request through to the backend.
- `Connection "upgrade"` — tells NGINX to keep the connection open and treat it as a protocol switch rather than a single request/response.

Long timeouts (`proxy_read_timeout`, `proxy_send_timeout`, set to 86400s) keep the tunnel alive for the duration of a chat session instead of closing it after NGINX's default timeout.

---

## 7. Issues Found and How They Were Fixed

| # | Issue | Where | Fix |
|---|---|---|---|
| 1 | Backend bound to `127.0.0.1`, unreachable from other containers | `Dockerfile` | Changed to `--host 0.0.0.0` so it listens on all interfaces inside the container |
| 2 | Frontend volume mount was commented out, so NGINX served its default welcome page | `docker-compose.yml` | Uncommented `./frontend:/usr/share/nginx/html:ro` |
| 3 | `proxy_pass` pointed to `localhost:8000`, and WebSocket upgrade headers were commented out | `nginx.conf` | Changed to `proxy_pass http://backend:8000/ws;` (using the Docker Compose service name) and uncommented `Upgrade`/`Connection` headers |

---

## 8. How the CI/CD Pipeline Works

Defined in `.github/workflows/deploy.yml`. On every push to `main`:

1. GitHub spins up a temporary Ubuntu runner.
2. The runner uses the `appleboy/ssh-action` to SSH into the EC2 instance, authenticating with a dedicated deploy keypair (private key stored as a GitHub Actions secret, public key added to the server's `authorized_keys`).
3. On the server, it runs:
   ```bash
   cd ~/devops
   git pull origin main
   docker-compose down
   docker-compose up -d --build
   ```
4. This pulls the latest code, rebuilds the Docker images, and restarts both containers — fully automated, no manual SSH needed after the initial setup.

Secrets used (configured under repo **Settings → Secrets and variables → Actions**):
- `SERVER_IP` — EC2 public IP
- `SERVER_USER` — SSH username (`ubuntu`)
- `SSH_PRIVATE_KEY` — private half of the dedicated deploy keypair

---

## 9. Steps to Deploy the Project

**Local test:**
```bash
git clone https://github.com/<your-username>/devops.git
cd devops
docker-compose up -d --build
```
Visit `http://localhost` — open two tabs to confirm real-time sync.

**Cloud deployment (EC2):**
1. Launch an Ubuntu EC2 instance (free tier), open inbound ports 22 and 80 in the security group.
2. SSH in and install Docker:
   ```bash
   sudo apt update && sudo apt install -y docker.io docker-compose git
   sudo usermod -aG docker $USER
   ```
3. Clone the repo and run:
   ```bash
   git clone https://github.com/<your-username>/devops.git
   cd devops
   docker-compose up -d --build
   ```
4. Visit `http://<EC2_PUBLIC_IP>`.

**Enable CI/CD:**
1. Generate a dedicated SSH keypair, add the public key to the EC2 instance's `~/.ssh/authorized_keys`.
2. Add `SERVER_IP`, `SERVER_USER`, `SSH_PRIVATE_KEY` as GitHub Actions secrets.
3. Push to `main` — the workflow automatically redeploys the latest code.

---

## Live Deployment

- **Repository:** `https://github.com/<your-username>/devops`
- **Live Public IP:** `http://<YOUR_EC2_PUBLIC_IP>`
