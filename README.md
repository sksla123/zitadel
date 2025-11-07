# 🚀 ZITADEL Installation Guide (with Nginx Reverse Proxy & Docker Compose)

## Overview

This repository provides a production-ready setup guide for deploying **[ZITADEL](https://zitadel.com)** — a modern, Go-based IAM (Identity and Access Management) solution — behind an **Nginx reverse proxy** using **Docker Compose**.  
The configuration prioritizes **security**, **maintainability**, and **resource efficiency** while adhering closely to ZITADEL’s official documentation.

---

## 🔍 What is ZITADEL?

ZITADEL is a **cloud-native IAM platform** written in Go, offering a lightweight yet powerful alternative to traditional Java-based IAM solutions such as Keycloak.

### Key Features
- **All-in-One Deployment:** Runs with only CockroachDB or PostgreSQL (no external dependencies)
- **Standards-Compliant:** Supports OIDC, OAuth 2.0, and SAML 2.0
- **Modern Authentication:** Built-in support for passwordless login (Passkeys, FIDO2)
- **API-First Design:** All management operations are exposed via API
- **Lightweight Performance:** Minimal memory (100–150 MB) and CPU usage

---

## 🧩 Architecture

```
Client (HTTPS)
   │
   ▼
[Nginx Reverse Proxy]  ← Handles SSL/TLS termination
   │
   ▼
[ZITADEL Backend] — http://127.0.0.1:3002 (gRPC/API)
[ZITADEL Frontend] — http://127.0.0.1:3001 (Login UI)
[PostgreSQL]       — Database
```

- **SSL termination** is handled by Nginx.
- ZITADEL communicates internally over HTTP (not HTTPS).
- External access is restricted to Nginx.

---

## ⚙️ Prerequisites

| Component | Description |
|------------|--------------|
| Docker & Docker Compose | Required to run ZITADEL, PostgreSQL, and Login UI containers |
| Nginx | Serves as a reverse proxy in front of the ZITADEL containers |
| Domain | A registered domain (e.g., `iam.example.com`) pointing to the Nginx server |
| SSL Certificate | Valid certificate for the domain (e.g., from Let’s Encrypt) |

---

## 🧱 Directory Structure

```
project-root/
├── docker-compose.yaml
└── config/
    ├── zitadel-config.yaml
    ├── zitadel-init-steps.yaml
    ├── zitadel-secrets.yaml
    ├── zitadel_masterkey.txt
    └── db_password.txt
```

---

## 🐳 Docker Compose Configuration

```yaml
services:
  zitadel:
    image: ghcr.io/zitadel/zitadel:v4.6.4
    container_name: zitadel_be
    restart: always
    networks:
      - zitadel
    command: >
      start-from-init
      --config /etc/zitadel/config.yaml
      --config /run/secrets/zitadel_secrets_file
      --steps /etc/zitadel/init-steps.yaml
      --masterkeyFile /run/secrets/zitadel_masterkey_file
    environment:
      ZITADEL_TLS_ENABLED: false
    ports:
      - "127.0.0.1:3002:8080"
      - "127.0.0.1:3001:3000"
    user: "0"
    configs:
      - source: zitadel_config_file
        target: /etc/zitadel/config.yaml
      - source: zitadel_init_steps
        target: /etc/zitadel/init-steps.yaml
    secrets:
      - zitadel_secrets_file
      - zitadel_masterkey_file
    volumes:
      - "ztd-shared-data:/pat:delegated"
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "/app/zitadel", "ready"]
      interval: 10s
      timeout: 60s
      retries: 5
      start_period: 10s

  login:
    image: ghcr.io/zitadel/zitadel-login:v4.6.4
    container_name: zitadel_fe
    restart: unless-stopped
    environment:
      - ZITADEL_API_URL=http://localhost:8080
      - CUSTOM_REQUEST_HEADERS=Host:iam.example.com
      - NEXT_PUBLIC_BASE_PATH=/ui/v2/login
      - ZITADEL_SERVICE_USER_TOKEN_FILE=/pat/login-client.pat
    network_mode: service:zitadel
    user: "1000"
    volumes:
      - "ztd-shared-data:/pat:ro"
    depends_on:
      zitadel:
        condition: service_healthy

  db:
    image: postgres:17-alpine
    restart: always
    environment:
      - POSTGRES_USER=psg_admin
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password_file
    networks:
      - zitadel
    volumes:
      - 'ztd-db-data:/var/lib/postgresql/data:rw'
    secrets:
      - db_password_file
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "psg_admin", "-d", "postgres"]
      interval: 10s
      timeout: 60s
      retries: 5
      start_period: 10s

networks:
  zitadel:

volumes:
  ztd-db-data:
  ztd-shared-data:

configs:
  zitadel_config_file:
    file: ./config/zitadel-config.yaml
  zitadel_init_steps:
    file: ./config/zitadel-init-steps.yaml

secrets:
  zitadel_secrets_file:
    file: ./config/zitadel-secrets.yaml
  zitadel_masterkey_file:
    file: ./config/zitadel_masterkey.txt
  db_password_file:
    file: ./config/db_password.txt
```

### ⚠️ Important
Keep the following environment variable:
```yaml
ZITADEL_TLS_ENABLED: false
```
Even if TLS is disabled in the configuration file or command flags, **this variable must remain set** — otherwise, the healthcheck will fail due to a known issue (see [ZITADEL Issue #9495](https://github.com/zitadel/zitadel/issues/9495)).

---

## 🧾 Configuration Files

### 1️⃣ zitadel-config.yaml

```yaml
Log:
  Level: 'info'

ExternalPort: 443
ExternalDomain: iam.example.com
ExternalSecure: true

TLS:
  Enabled: false

Database:
  postgres:
    Host: 'db'
    Port: 5432
    Database: zitadel
    User:
      SSL:
        Mode: 'disable'
    Admin:
      SSL:
        Mode: 'disable'

WebAuthNName: your-root-org-name
```

---

### 2️⃣ zitadel-init-steps.yaml

```yaml
FirstInstance:
  LoginClientPatPath: /pat/login-client.pat
  Org:
    Human:
      Username: 'admin'
      Password: 'Admin1234!'
      PasswordChangeRequired: true
    LoginClient:
      Machine:
        Username: 'login-client'
        Name: 'Automatically Initialized IAM_LOGIN_CLIENT'
      pat:
        ExpirationDate: '2999-01-01T00:00:00Z'
```

---

## 🔐 Secrets

### 1. Master Key
```bash
tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 32 > ./config/zitadel_masterkey.txt
chmod 600 ./config/zitadel_masterkey.txt
```

### 2. Database Password
```bash
tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 32 > ./config/db_password.txt
chmod 600 ./config/db_password.txt
```

### 3. zitadel-secrets.yaml
```yaml
Database:
  postgres:
    User:
      Username: 'ztd_user'
      Password: 'STRONG_PASSWORD_FOR_ZITADEL_USER'
    Admin:
      Username: 'psg_admin'
      Password: 'VALUE_FROM_db_password.txt'
```

---

## 🌐 Nginx Reverse Proxy Configuration

### /etc/nginx/conf.d/upstreams.conf
```nginx
upstream zitadel_frontend {
    server 127.0.0.1:3001;
}

upstream zitadel_backend {
    server 127.0.0.1:3002;
}
```

### /etc/nginx/sites-enabled/auth
```nginx
server {
    listen 443 ssl http2;
    server_name iam.example.com;

    ssl_certificate /etc/letsencrypt/live/iam.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/iam.example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;

    location /ui {
        proxy_pass http://zitadel_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;
    }

    location /ui/v2/login {
        proxy_pass http://zitadel_frontend;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;
    }

    location /debug {
        proxy_pass http://zitadel_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;
    }

    location / {
        grpc_pass grpc://zitadel_backend;
        grpc_set_header Host $host;
        grpc_set_header X-Forwarded-Proto https;
    }
}
```

---

## 🧠 Nginx Performance Tuning (Optional)

In `/etc/nginx/nginx.conf`, increase worker connections:

```nginx
events {
    worker_connections 1024;
}
```

This improves concurrency when handling long-lived HTTP/2 or gRPC connections.

---

## 🚀 Deployment

```bash
# Validate Nginx configuration
sudo nginx -t && sudo systemctl restart nginx

# Launch ZITADEL stack
docker compose up -d
```

Access the console at:

```
https://iam.example.com/ui/console
```

Default credentials:
```
Username: admin@zitadel.iam.example.com
Password: Admin1234!
```

---

## ✅ Post-Setup Notes

- The first login will require a password change (`PasswordChangeRequired: true`).
- Regularly back up `zitadel_masterkey.txt` and your database.
- Ensure secrets are **never committed** to Git or exposed publicly.

---

## 📚 References
- [ZITADEL Official Documentation](https://zitadel.com/docs)
- [ZITADEL GitHub Repository](https://github.com/zitadel/zitadel)
- [Known TLS Healthcheck Issue #9495](https://github.com/zitadel/zitadel/issues/9495)

---
### 📘 Korean Installation Guide

If you prefer to read this guide in Korean, check out the full article here:  
[View Korean Guide (한국어 가이드 보기)](https://nyong2e.tistory.com/21)