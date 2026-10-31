# 🎯 Vibe Kanban Stack

Self-hosted deployment of the Vibe Kanban Cloud (remote) variant — multi-user
Kanban for AI coding agents with GitHub/Google OAuth or bootstrap local
authentication, PostgreSQL persistence, and ElectricSQL real-time sync.

## 🧩 Components

### 🖥️ [Vibe Kanban Remote Server](app/)

The remote-server deployment: source-built `vk-remote-server` image, PostgreSQL
database, and ElectricSQL sync layer, with forwarding / letsencrypt / step-ca
environments behind the collection's nginx-proxy stacks.

### 🌐 [Let's Encrypt Manager](modules/ssl-automation/letsencrypt-manager/)

Automatic SSL certificate generation and renewal using Let's Encrypt for
production deployments with internet access.

### 🔒 [Step CA Manager](modules/ssl-automation/step-ca-manager/)

Self-signed trusted certificate authority for virtual Docker networks
without internet access. Automatically manages and distributes CA
certificates within isolated environments.

## 🚀 Quick Start

```bash
cd app
docker build -t localhost/vk-remote-server:0.1.45 images/vk-remote-server
sb build
cd build/forwarding   # or letsencrypt / step-ca
cp .env.example .env  # set secrets + auth config
docker compose up -d
```

See [`app/README.md`](app/README.md) for the full runbook.

## 📋 Requirements

- Docker & Docker Compose (BuildKit for the source image build)
- 4+ GB free RAM for the image build (10–15 min)
- Domain name (for production deployments)
- Email address (for Let's Encrypt)
- One auth method: GitHub or Google OAuth credentials, or bootstrap local auth

## 📄 License

This project is dual-licensed under:

- [Apache License 2.0](LICENSE-APACHE)
- [MIT License](LICENSE-MIT)
