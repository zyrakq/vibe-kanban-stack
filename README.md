# 🎯 Vibe Kanban Stack

Self-hosted deployment of the Vibe Kanban Cloud (remote) variant — multi-user
Kanban for AI coding agents with GitHub/Google OAuth or bootstrap local
authentication, PostgreSQL persistence, and ElectricSQL real-time sync.

## 🧩 Components

### 🖥️ [Vibe Kanban Remote Server](app/)

The app deployment: source-built `vibe-kanban-remote-server` image, the
`vibe-kanban-app` and `vibe-kanban-electric` services, a swappable
postgresql/pgrust database extension, and forwarding / letsencrypt / step-ca
environments behind the collection's nginx-proxy stacks.

### 🌐 [Let's Encrypt Manager](modules/ssl-automation/letsencrypt-manager/)

Automatic SSL certificate generation and renewal using Let's Encrypt for
production deployments with internet access.

### 🔒 [Step CA Manager](modules/ssl-automation/step-ca-manager/)

Self-signed trusted certificate authority for virtual Docker networks
without internet access. Automatically manages and distributes CA
certificates within isolated environments.

## 🚀 Quick Start

Each component has its own README with detailed setup instructions. Choose
the certificate management solution that fits your deployment scenario.

Example deployment flow (forwarding, postgresql combo):

```bash
cd app
docker build -t localhost/vibe-kanban-remote-server:0.1.45 images/vibe-kanban-remote-server
sb build
cd build/forwarding/postgresql
cp .env.example .env   # set secrets + auth config
docker compose up -d
```

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
