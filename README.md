# n8n – Local Podman Setup

## Overview

This repository contains the Docker Compose configuration for running **n8n locally on Windows using Podman Desktop**.

The environment is intended for development, testing, and evaluating n8n as a workflow automation platform.

### Environment

| Component | Configuration |
|---|---|
| Host OS | Windows |
| Container runtime | Podman Desktop |
| Podman backend | WSL2 |
| Container image | `docker.io/n8nio/n8n:latest` |
| Container name | `n8n` |
| Web interface | http://localhost:5678 |
| Timezone | `Europe/Copenhagen` |
| Database | SQLite (default) |
| Persistent storage | Podman named volume |
| Local file access | Read-only bind mount |

## 1. Prerequisites

Install:

- Podman Desktop and Podman CLI
- Windows Subsystem for Linux (WSL2)
- A Compose provider (Docker Compose or Podman Compose)

Verify the installation:

```powershell
podman --version
podman compose version
```

## 2. Initialize the Podman Machine

Create a Podman machine with 4 CPUs, 4 GB RAM, and 50 GB disk space:

```powershell
podman machine init --cpus 4 --memory 4096 --disk-size 50
```

Start the machine:

```powershell
podman machine start
```

Verify:

```powershell
podman machine list
podman info
```

If a machine already exists, skip initialization.

## 3. Project Directory

The project is stored in:

```text
C:\git\docker-compose-n8n
```

Directory structure:

```text
C:\git\docker-compose-n8n\
├── compose.yaml
├── README.md
├── files\
│   └── test.xlsx
└── backups\
    └── n8n-backup.tar
```

Create the required directories:

```powershell
New-Item -ItemType Directory -Force C:\git\docker-compose-n8n\files
New-Item -ItemType Directory -Force C:\git\docker-compose-n8n\backups
```

Do not commit Excel data, backup archives, credentials, or other sensitive files to Git.

Suggested `.gitignore`:

```gitignore
files/
backups/
.env
```

## 4. Compose Configuration

The `compose.yaml` file defines the n8n container.

```yaml
services:
  n8n:
    image: docker.io/n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped

    ports:
      - "127.0.0.1:5678:5678"

    environment:
      - GENERIC_TIMEZONE=Europe/Copenhagen
      - TZ=Europe/Copenhagen
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_RESTRICT_FILE_ACCESS_TO=/files

    volumes:
      - n8n_data:/home/node/.n8n
      - ./files:/files:ro

volumes:
  n8n_data:
```

### Configuration Explanation

**Port mapping**

```yaml
- "127.0.0.1:5678:5678"
```

Makes n8n accessible locally at http://localhost:5678.

**Timezone**

```yaml
- GENERIC_TIMEZONE=Europe/Copenhagen
- TZ=Europe/Copenhagen
```

Configures the timezone for n8n and the container.

**Settings file permissions**

```yaml
- N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
```

Enforces restrictive permissions on n8n's configuration file.

**File access restrictions**

```yaml
- N8N_RESTRICT_FILE_ACCESS_TO=/files
```

Restricts n8n file-access operations to `/files`.

**Persistent storage**

```yaml
- n8n_data:/home/node/.n8n
```

Stores the SQLite database, workflows, encrypted credentials, and encryption key in a persistent Podman volume.

**Local file access**

```yaml
- ./files:/files:ro
```

Mounts the Windows `files` directory into the container as `/files`, with read-only access.

Example mapping:

```text
Windows:
C:\git\docker-compose-n8n\files\test.xlsx

Container:
/files/test.xlsx
```

## 5. Start n8n

Open PowerShell:

```powershell
cd C:\git\docker-compose-n8n
podman compose up -d
```

Open the web interface:

http://localhost:5678

Complete the initial account setup if prompted.

Verify the container:

```powershell
podman ps
```

View logs:

```powershell
podman logs n8n
```

Follow logs:

```powershell
podman logs -f n8n
```

## 6. Stop and Restart n8n

Run commands from the project directory:

```powershell
cd C:\git\docker-compose-n8n
```

Stop n8n:

```powershell
podman compose stop
```

Start it again:

```powershell
podman compose start
```

Apply configuration changes:

```powershell
podman compose up -d
```

Remove the container while preserving persistent data:

```powershell
podman compose down
```

**Warning:** Avoid `podman compose down -v` unless intentionally deleting persistent data.

## 7. Access Local Excel Files

Place Excel files in:

```text
C:\git\docker-compose-n8n\files\
```

Verify access:

```powershell
podman exec n8n ls -l /files
```

### Example n8n Workflow

Create the following workflow:

1. **Manual Trigger** — starts the workflow.
2. **Read/Write Files from Disk** — reads `/files/test.xlsx`.
3. **Extract From File** — extracts XLSX data.
4. **Filter** — optionally filters rows based on column values.
5. **Code** — processes the resulting data.

File-reading node:

```text
Operation: Read File(s) From Disk
File Selector: /files/test.xlsx
```

Extraction node:

```text
Operation: Extract From XLSX
Input Binary Field: data
```

The extracted Excel rows are available as JSON items.

## 8. Persistent Data

Podman Compose automatically creates a named volume.

List volumes:

```powershell
podman volume ls
```

The volume name will typically be:

```text
docker-compose-n8n_n8n_data
```

The actual name depends on the Compose project name.

**Important:** Changing the project directory can change the Compose project name and therefore the volume name. An existing volume is not automatically migrated when the project is moved.

The volume persists when the container is stopped, removed, or recreated, provided it is not explicitly deleted.

## 9. Backup

Stop n8n before backing up the SQLite database:

```powershell
cd C:\git\docker-compose-n8n
podman compose stop
```

Create the backup directory if necessary:

```powershell
New-Item -ItemType Directory -Force C:\git\docker-compose-n8n\backups
```

Identify the actual volume:

```powershell
podman volume ls
```

Export the volume, replacing the example volume name if necessary:

```powershell
podman volume export docker-compose-n8n_n8n_data --output C:\git\docker-compose-n8n\backups\n8n-backup.tar
```

Restart n8n:

```powershell
podman compose start
```

The backup includes the persistent application data, including the SQLite database and encryption key.

**Security:** Treat backup archives as sensitive and store them securely.

## 10. Restore

Restore only while n8n is stopped and into a new or empty volume.

Create the volume if necessary:

```powershell
podman volume create docker-compose-n8n_n8n_data
```

Import the backup:

```powershell
podman volume import docker-compose-n8n_n8n_data C:\git\docker-compose-n8n\backups\n8n-backup.tar
```

Start n8n:

```powershell
cd C:\git\docker-compose-n8n
podman compose up -d
```

Verify that workflows and credentials are available.

Restore using a compatible n8n version. For reliable recovery, record the image version used when creating each backup.

## 11. Security Considerations

This deployment is intended for local evaluation, not production use.

Current security measures:

- Web interface bound to `127.0.0.1`
- Restricted access to local files
- Read-only Excel file mount
- Enforced configuration file permissions
- Encrypted n8n credentials

Before production deployment, evaluate:

- Dedicated Linux VM or server
- HTTPS and access controls
- External PostgreSQL database
- Encryption key management
- Backup encryption and recovery testing
- Execution-data retention
- Microsoft Entra ID permissions
- Container image version pinning
- Network restrictions and monitoring

## 12. Useful Commands

| Command | Purpose |
|---|---|
| `podman machine start` | Start Podman machine |
| `podman machine stop` | Stop Podman machine |
| `podman compose up -d` | Start n8n |
| `podman compose stop` | Stop n8n |
| `podman compose down` | Remove containers and Compose network |
| `podman ps` | List running containers |
| `podman logs n8n` | View n8n logs |
| `podman volume ls` | List persistent volumes |
| `podman exec n8n ls -l /files` | Verify local file access |
| `podman exec n8n n8n audit` | Run n8n security audit |

## 13. References

- [n8n Documentation](https://docs.n8n.io/)
- [n8n Docker Installation](https://docs.n8n.io/hosting/installation/docker/)
- [n8n Security](https://docs.n8n.io/hosting/securing/)
- [Podman Documentation](https://docs.podman.io/)
- [Podman Desktop](https://podman-desktop.io/)

---

**Status:** Local proof-of-concept environment for evaluating n8n workflow automation capabilities.
# docker-compose-n8n
Files needed for creating a n8n instance in docker or podman
