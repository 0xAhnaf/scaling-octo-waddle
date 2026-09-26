# Deployment Runbook: Sticky Notes API

> **Purpose:** Step-by-step deployment guide for the CSE3100 Sticky Notes application on a Linux server with PM2 + Nginx + MySQL (Docker).
>
> **Target:** `s202302040**.austattendance.online` → `187.52.122.100`
>
> **Last Updated:** 2026-09-27

---

## Table of Contents

1. [Configuration Reference](#configuration-reference)
2. [Prerequisites](#prerequisites)
3. [SSH Connection](#1-ssh-connection)
4. [Clone Repository](#2-clone-repository)
5. [MySQL Container](#3-mysql-container)
6. [Database User Setup](#4-database-user-setup)
7. [Environment Configuration](#5-environment-configuration)
8. [PM2 Process Manager](#6-pm2-process-manager)
9. [Local Health Check](#7-local-health-check)
10. [Nginx Reverse Proxy](#8-nginx-reverse-proxy)
11. [Final Verification](#9-final-verification)
12. [Important Notes](#important-notes)
13. [Troubleshooting](#troubleshooting)

---

## Configuration Reference

| Variable | Value | Description |
|----------|-------|-------------|
| `SSH_KEY` | `~/Downloads/s202302040**` | Path to private SSH key |
| `SERVER_USER` | `s202302040**` | Remote server username |
| `SERVER_IP` | `187.52.122.100` | Server public IP address |
| `APP_PORT` | `30**` | Application listening port |
| `DB_NAME` | `bookdb` | MySQL database name |
| `DB_PORT` | `3306` | MySQL port |
| `DB_USER` | `s202302040**` | Database username |
| `DB_PASSWORD` | `PASSWORD` | Database password *(change in production)* |
| `DOMAIN` | `s202302040**.austattendance.online` | Public domain name |
| `REPO_URL` | `https://github.com/S-Arshad032/cse3100-sticky-note-previous-year.git` | GitHub repository |
| `PROJECT_DIR` | `/home/s202302040**/cse3100-sticky-note-previous-year` | Project path on server |
| `PM2_NAME` | `bookapi-0**` | PM2 process name |
| `CONTAINER_NAME` | `bookdb` | MySQL Docker container name |

> ⚠️ **Security:** Never commit actual passwords. Replace `PASSWORD` with a strong secret. Use `.env.example` as a template and fill in secrets locally.

---

## Prerequisites

- SSH key configured at `SSH_KEY` with access to `SERVER_USER@SERVER_IP`
- Docker installed and running on the server
- PM2 installed globally (`npm install -g pm2`)
- Nginx installed and running
- Domain `DOMAIN` pointed to `SERVER_IP` (A record)

---

Link to ssh key: **[https://drive.google.com/drive/folders/1h-mK3DO2VyKcZn-3-io85SvV-9i38n6G?usp=drive_link](https://drive.google.com/drive/folders/1h-mK3DO2VyKcZn-3-io85SvV-9i38n6G?usp=drive_link)**

---

## 0. Restrict permission for ssh key

```
icacls <path_to_your_privatekey> /inheritance:r
icacls <path_to_your_privatekey> /grant:r ""$($env:USERNAME):(R)"""
```


---

## 1. SSH Connection

```bash
ssh -i "$SSH_KEY" "$SERVER_USER@$SERVER_IP"
```

---

## 2. Clone Repository

```bash
cd /home/$SERVER_USER
git clone "$REPO_URL"
cd cse3100-sticky-note-previous-year
cat .env.example
```

---

## 3. MySQL Container

### Start Container
```bash
sudo docker start "$CONTAINER_NAME"
```

### Get Container IP
```bash
sudo docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' "$CONTAINER_NAME"
```

> **Action:** Copy the printed IP address — you'll need it for `DB_HOST` in the `.env` file.

---

## 4. Database User Setup

```bash
sudo docker exec -it "$CONTAINER_NAME" mysql -uroot -prootpass
```

```sql
CREATE USER IF NOT EXISTS 's202302040**'@'%' IDENTIFIED BY 'PASSWORD';
ALTER USER 's202302040**'@'%' IDENTIFIED BY 'PASSWORD';
GRANT SELECT ON bookdb.* TO 's202302040**'@'%';
FLUSH PRIVILEGES;
EXIT;
```

---

## 5. Environment Configuration

```bash
cp .env.example /home/$SERVER_USER/.env
nano /home/$SERVER_USER/.env
```

Paste the following values into `.env`:

```ini
PORT=30**
HOST=127.0.0.1
DB_HOST=<PASTE_CONTAINER_IP_HERE>
DB_PORT=3306
DB_NAME=bookdb
DB_USER=s202302040**
DB_PASSWORD=PASSWORD
```

> **Important:** Replace `<PASTE_CONTAINER_IP_HERE>` with the actual IP from **Step 3**.

**Save & Exit nano:** `Ctrl+O`, `Enter`, then `Ctrl+X`

---

## 6. PM2 Process Manager

```bash
cd "$PROJECT_DIR"
pm2 start server.js --name "$PM2_NAME"
pm2 save
pm2 startup
```

> **Action:** Copy and run the exact `sudo env PATH=...` command printed by `pm2 startup`.

```bash
pm2 save
```

---

## 7. Local Health Check

```bash
curl http://127.0.0.1:30**/api/health
```

**Expected Response:**
```json
{"status":"ok","database":"up"}
```

---

## 8. Nginx Reverse Proxy

### Create Site Configuration
```bash
sudo nano /etc/nginx/sites-available/s202302040**.conf
```

Paste **one** of the following configurations:

**Option A — With Full Proxy Headers (Recommended)**
```nginx
server {
    listen 80;
    server_name s202302040**.austattendance.online;

    location / {
        proxy_pass http://127.0.0.1:30**;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Option B — Minimal**
```nginx
server {
    listen 80;
    server_name s202302040**.austattendance.online;

    location / {
        proxy_pass http://127.0.0.1:30**;
    }
}
```

**Save & Exit nano:** `Ctrl+O`, `Enter`, then `Ctrl+X`

### Enable Site & Reload Nginx
```bash
sudo ln -sf /etc/nginx/sites-available/s202302040**.conf /etc/nginx/sites-enabled/s202302040**.conf
sudo nginx -t && sudo systemctl reload nginx
```

---

## 9. Final Verification

### Public Health Check
```bash
curl http://s202302040**.austattendance.online/api/health
```

**Expected Response:**
```json
{"status":"ok","database":"up"}
```

### Open in Browser
- http://s202302040**.austattendance.online
- http://s202302040**.austattendance.online/api/health

---

## Important Notes

| ❌ Do Not | ✅ Do Instead |
|-----------|---------------|
| Run `npm install` | Dependencies should already be in repo |
| Run `npm run build` | No build step required |
| Run PM2 with `sudo` | Run PM2 as the deploy user |
| Drop or delete `bookdb` container | Keep container running |
| Stop or remove `bookdb` container | Container must persist for database |

---

## Troubleshooting

| Symptom | Diagnosis Command | Likely Fix |
|---------|-------------------|------------|
| `curl` fails locally | `pm2 logs $PM2_NAME` | Check app errors, verify `.env` values |
| Nginx returns 502 | `sudo nginx -t` | Fix config, ensure `proxy_pass` port matches `APP_PORT` |
| Database connection refused | `docker inspect $CONTAINER_NAME` | Verify `DB_HOST` matches container IP |
| PM2 process not persisting | `pm2 list` after reboot | Re-run `pm2 startup` command exactly as printed |
| Domain not resolving | `dig $DOMAIN` | Verify A record points to `SERVER_IP` |
| Permission denied (SSH) | `ssh -v -i $SSH_KEY ...` | Check key permissions (`chmod 600`), correct user/IP |

---

## Quick Checklist

- [ ] SSH connected to server
- [ ] Repository cloned to `$PROJECT_DIR`
- [ ] MySQL container (`$CONTAINER_NAME`) running
- [ ] Database user created with `SELECT` on `bookdb`
- [ ] `.env` configured with correct `DB_HOST` (container IP)
- [ ] PM2 process `$PM2_NAME` started and saved
- [ ] `pm2 startup` command executed
- [ ] Local health check returns `{"status":"ok","database":"up"}`
- [ ] Nginx site enabled and reloaded
- [ ] Public health check returns `{"status":"ok","database":"up"}`
- [ ] Application accessible via browser at `http://$DOMAIN`

---

*Generated for CSE3100 Sticky Notes deployment. Keep this runbook updated as infrastructure changes.*