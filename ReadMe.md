# Minecraft Java Server on Azure (Docker + Fabric)

This repository contains the configuration and existing world data for hosting a Minecraft Java Edition server on an Azure Ubuntu Virtual Machine.

## Server configuration

| Setting | Value |
|---|---|
| Hosting | Azure Ubuntu VM |
| Minecraft Edition | Java |
| Minecraft Version | `26.3` |
| Server Software | Fabric |
| Docker Image | `itzg/minecraft-server:latest` |
| RAM Allocation | 1.5 GB Java heap |
| Expected Players | 2–5 |
| Authentication | Offline mode |
| Server Port | `25565` (TCP) |
| World | Existing Aternos world |

### Installed mods

The following Fabric mods are already included in `data/mods/`:

- Fabric API `0.160.5+26.3`
- FerriteCore `9.0.0-fabric`
- Krypton `0.3.2`
- Lithium `0.26.1+mc26.3`

Make sure every player uses a compatible Minecraft client and the required client-side mods.

---

## 1. Azure VM prerequisites

Create an Azure Virtual Machine with:

- **OS:** Ubuntu 22.04 or 24.04 LTS
- **Architecture:** x86-64
- **RAM:** Preferably 2 GB or more, since the JVM heap is configured to use 3 GB and the operating system and Docker also need memory.
- **Storage:** Enough disk space for the world and future backups.
- **Public IP:** Required for direct internet connections.

Connect to your VM using SSH:

```bash
ssh <username>@<VM_PUBLIC_IP>
```

Replace the placeholders with your Azure VM username and public IP.

## 2. Install Docker and Git

Update Ubuntu:

```bash
sudo apt update
sudo apt upgrade -y
```

Install Docker, Docker Compose and Git:

```bash
sudo apt install -y docker.io docker-compose-v2 git
```

Enable Docker:

```bash
sudo systemctl enable --now docker
```

Verify the installations:

```bash
docker --version
docker compose version
git --version
```

## 3. Clone the repository

Clone the repository:

```bash
git clone https://github.com/topCodegeek/mconazure.git
```

Enter the project directory:

```bash
cd mconazure
```

The repository should have this structure:

```text
mconazure/
├── compose.yaml
└── data/
    ├── banned-ips.json
    ├── banned-players.json
    ├── ops.json
    ├── server.properties
    ├── whitelist.json
    ├── config/
    ├── defaultconfigs/
    ├── mods/
    │   ├── fabric-api-0.160.5+26.3.jar
    │   ├── ferritecore-9.0.0-fabric.jar
    │   ├── krypton-0.3.2.jar
    │   └── lithium-fabric-0.26.1+mc26.3.jar
    └── world/
        ├── level.dat
        ├── data/
        ├── datapacks/
        ├── dimensions/
        ├── players/
        └── serverconfig/
```

**Important:** Keep the existing `data/world/` directory. It contains the world you are migrating from Aternos.

## 4. Verify Docker Compose configuration

The root-level `compose.yaml` should contain:

```yaml
services:
  minecraft:
    image: itzg/minecraft-server:latest
    container_name: minecraft
    restart: unless-stopped

    ports:
      - "25565:25565"

    environment:
      EULA: "TRUE"
      TYPE: "FABRIC"
      VERSION: "26.3"
      MEMORY: "1.5G"
      OVERRIDE_SERVER_PROPERTIES: "false"

    volumes:
      - ./data:/data

    stdin_open: true
    tty: true
```

### Important configuration notes

- `TYPE: FABRIC` tells the container to install and run Fabric.
- `VERSION: "26.3"` selects the Minecraft version.
- `MEMORY: "1.5G"` configures the Java heap.
- `./data:/data` mounts