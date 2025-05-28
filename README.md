# Ansible Semaphore Docker POC

This repository contains:

* **Playbooks** (`./playbooks/`)
* **Docker Compose** configuration (`docker-compose.yml`)
* **Reports** (`./reports`)
* **Retry Files** (`./playbooks/retry_files`)

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
```

> * `--system` creates a system account (UID < 1000 normally, but we force 1001 here)
> * `--no-create-home` skips creating `/home/semaphore`
> * `--group` creates a matching group `semaphore`
> * `--shell /usr/sbin/nologin` prevents interactive login

Verify:

```bash
getent passwd 1001
# should output something like:
# semaphore:x:1001:1001::/home/semaphore:/usr/sbin/nologin
enument group semaphore
# should show GID 1001
```

---

## 2. Grant group read/write on mounted folders

We bind-mount the following host directories into the container:

* `./playbooks` → `/playbooks`
* `./reports`   → `/reports`

*To allow your host user (typically UID/GID=1000) to read/write these files*, make sure the directories on the host are owned by group `1000` (or your actual host primary group) and are group-writable:

```bash
# From your repo root:
# Replace 1000 with your host user's actual GID if different
HOST_GID=1000

# Change group ownership to your host group and grant group rwX
sudo chown -R :${HOST_GID} playbooks reports
sudo chmod -R g+rwX playbooks reports

# (Optional) Set the setgid bit so new files (e.g. retry_files under playbooks/) inherit the group:
sudo find playbooks reports -type d -exec chmod g+s {} +
```

* `chmod g+rwX`:

  * `g+r`: group-read
  * `g+w`: group-write
  * `X`: execute only on directories (keeps them traversable)

* `chmod g+s` on directories ensures any new file/dir created inside inherits the parent group.

---

## 3. Start the POC stack

Once the host user and permissions are in place, you can spin up the Docker services:

```bash
docker-compose up -d
```

* Semaphore will run as UID 1001\:GID 1001 (`semaphore:semaphore`) inside the container.
* All your playbooks, reports, and retry files will be created with group `semaphore` so you can modify them on the host.

---

## 4. Usage Overview

1. **Full run** template → runs all hosts → creates `retry_files/<playbook>.retry` for failures.
2. **Retry run** template → uses `--limit @retry_files/...` → re-runs only the failed hosts.
3. **Reports** are CSV files in `./reports/` for auditing run status.


