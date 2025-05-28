# Ansible Semaphore Docker POC

This repository contains:

- **Playbooks** (`./playbooks/`)
- **Docker Compose** configuration (`docker-compose.yml`)
- **Inventory**, **Config**, **Reports**, and **Retry Files** folders

Before you `docker-compose up`, you need to set up a **semaphore** user on the host (so that the container can run as a non-root user) and grant group read/write permissions on the mounted directories.

---

## 1. Create a `semaphore` system user (UID/GID = 1001)

We will create a user named `semaphore` with UID `1001`, so that inside the container it matches the built-in `semaphore` user:

```bash
# Create a system user with UID 1001 and no login shell
sudo adduser \
  --system \
  --no-create-home \
  --group \
  --uid 1001 \
  --shell /usr/sbin/nologin \
  semaphore

