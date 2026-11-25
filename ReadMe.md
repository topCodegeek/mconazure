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
| Java Heap | 1536 M |
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

## 1. Create an Azure VM

Create an Azure Virtual Machine with:

- **OS:** Ubuntu 22.04 or 24.04 LTS
- **Architecture:** x86-64
- **RAM:** Preferably 4 GB or more
- **Storage:** Enough SSD space for the world and backups
- **Public IP:** Required for direct internet connections

Connect using SSH:

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

Enable Docker to start automatically whenever Ubuntu boots:

```bash
sudo systemctl enable --now docker
```

Verify:

```bash
docker --version
docker compose version
git --version
```

## 3. Configure SSD-backed swap memory

Swap allows Ubuntu to use a portion of your SSD as virtual memory when physical RAM is under pressure.

This can help prevent sudden out-of-memory failures, although SSD swap is significantly slower than physical RAM. It should be treated as emergency headroom, not additional usable Minecraft RAM.

The following creates a 2 GB swap file.

Check whether swap already exists:

```bash
sudo swapon --show
free -h
```

If you already have sufficient swap configured, you can skip the following creation steps.

Create a 2 GB swap file:

```bash
sudo fallocate -l 2G /swapfile
```

Set secure permissions:

```bash
sudo chmod 600 /swapfile
```

Format it as swap:

```bash
sudo mkswap /swapfile
```

Enable it:

```bash
sudo swapon /swapfile
```

Make swap persistent across reboots:

```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Configure Ubuntu to use swap conservatively:

```bash
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf
sudo sysctl -p /etc/sysctl.d/99-swappiness.conf
```

Verify:

```bash
sudo swapon --show
free -h
```

You should now see approximately 2 GB of additional swap space.

**Important:** Do not increase Minecraft's Java heap just because swap is available. Keep the heap at 3 GB. Excessive swapping can make the server extremely slow and cause freezes.

## 4. Clone the repository

```bash
git clone https://github.com/topCodegeek/mconazure.git
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

**Important:** Keep the existing `data/world/` directory. It contains the world migrated from Aternos.

## 5. Configure Docker Compose

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
      MEMORY: "1536M" # 1.5 gb (decimal not allowed)
      OVERRIDE_SERVER_PROPERTIES: "false"

    volumes:
      - ./data:/data

    # Prevent Docker logs from growing indefinitely.
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

    stdin_open: true
    tty: true
```

### Understanding the important settings

| Setting | Purpose |
|---|---|
| `MEMORY: "1536M"` | Sets the Java heap allocation |
| `restart: unless-stopped` | Automatically restarts the server after VM/Docker restarts |
| `./data:/data` | Mounts the existing world and server files |
| `max-size: "10m"` | Rotates a log file when it reaches 10 MB |
| `max-file: "3"` | Retains at most three rotated log files |

Docker will retain approximately 30 MB of container log files, excluding small metadata overhead.

This prevents persistent Docker logs from consuming unlimited disk space. It does not limit the Minecraft server's own memory usage.

**Note:** `docker compose logs -f` streams logs to your terminal; it does not itself store an unlimited copy. Docker's log rotation settings control the persistent container logs.

## 6. Configure server.properties

Open the existing file:

```bash
nano data/server.properties
```

Verify:

```properties
level-name=world
online-mode=false
server-port=25565
```

`level-name` must match your existing world directory.

`online-mode=false` enables offline/cracked access. Minecraft accounts are not authenticated by Microsoft in this mode.

Save using `Ctrl + O`, press Enter, and exit using `Ctrl + X`.

## 7. Set file permissions

The Minecraft Docker image normally runs using UID/GID 1000.

Run:

```bash
sudo chown -R 1000:1000 data
```

## 8. Configure Azure networking

Minecraft Java Edition uses TCP port `25565` by default.

In the Azure Portal:

1. Open your Virtual Machine.
2. Go to **Networking**.
3. Open the associated Network Security Group.
4. Add an inbound security rule:

| Setting | Value |
|---|---|
| Source | Any, or your players' public IPs |
| Source port ranges | `*` |
| Destination | Any |
| Service | Custom |
| Destination port | `25565` |
| Protocol | TCP |
| Action | Allow |
| Priority | An unused priority |
| Name | `Minecraft-Server` |

If UFW is enabled:

```bash
sudo ufw allow 25565/tcp
```

Do not expose the Docker RCON port (`25575`) publicly.

## 9. Start the Minecraft server

From the directory containing `compose.yaml`:

Pull the Docker image:

```bash
sudo docker compose pull
```

Start the server:

```bash
sudo docker compose up -d
```

View the logs:

```bash
sudo docker compose logs -f minecraft
```

Wait until the logs indicate that the server has finished starting and is ready to accept connections.

Press `Ctrl + C` to exit the log viewer. This does not stop the server.

---

## 10. Configure automatic server startup

The server is configured with:

```yaml
restart: unless-stopped
```

Docker uses this restart policy to automatically start the Minecraft container when its daemon starts, including after an Ubuntu VM reboot.

Docker itself must also be enabled at boot:

```bash
sudo systemctl enable docker
```

Since you already enabled Docker in Step 2, this is just a verification step.

Check:

```bash
sudo systemctl is-enabled docker
```

Expected output:

```text
enabled
```

Check the container restart policy:

```bash
sudo docker inspect minecraft --format '{{.HostConfig.RestartPolicy.Name}}'
```

Expected output:

```text
unless-stopped
```

### Test automatic startup

Once the Minecraft server has started successfully, reboot the VM:

```bash
sudo reboot
```

Reconnect through SSH after Ubuntu comes back online:

```bash
ssh <username>@<VM_PUBLIC_IP>
```

Check the server:

```bash
sudo docker compose -f ~/mconazure/compose.yaml ps
```

If your repository is in a different directory, adjust the path.

The Minecraft container should start automatically without manually executing `docker compose up` again.

**Important:** This starts Minecraft when the VM boots. It does not automatically start an Azure VM that has been deallocated or stopped from the Azure Portal. You must start the VM itself through Azure first.

---

## 11. Connect to the server

In Minecraft Java Edition:

1. Open Multiplayer.
2. Select Add Server.
3. Enter:

```text
<VM_PUBLIC_IP>:25565
```

Connect using Minecraft Java Edition `26.3`.

---

## Server management

Run these commands from the directory containing `compose.yaml`.

### View logs

```bash
sudo docker compose logs -f minecraft
```

### Check status

```bash
sudo docker compose ps
```

### Stop the server

```bash
sudo docker compose stop
```

### Start the server

```bash
sudo docker compose start
```

### Restart the server

```bash
sudo docker compose restart
```

### Apply Compose configuration changes

```bash
sudo docker compose up -d
```

### Stop and remove the container

```bash
sudo docker compose down
```

This removes the container and network but does not delete the bind-mounted `data/` directory.

---

## Backups

Your world and server configuration are stored in the local `data/` directory.

Create a backup:

```bash
sudo docker compose stop

sudo tar -czf ~/minecraft-backup-$(date +%F-%H%M).tar.gz data/

sudo docker compose start
```

List backups:

```bash
ls -lh ~/minecraft-backup-*.tar.gz
```

Keep backups outside the Git repository and ideally copy important backups to separate storage.

Restore a backup only after stopping the server and preserving the current data directory.

---

## Updating the server

Before updating:

1. Stop the server.
2. Create a backup.
3. Update the desired Minecraft version or mod files.
4. Start the server and inspect the logs.

Pull the latest Docker image:

```bash
sudo docker compose pull
sudo docker compose up -d
```

**Warning:** Do not blindly run `git pull` on a running server. The `data/` directory contains live world files, which change as players play. Git updates can conflict with these files or overwrite important data.

---

## Troubleshooting

### Cannot connect

Check:

```bash
sudo docker compose ps
sudo docker compose logs --tail=100 minecraft
sudo ufw status
```

Verify the Azure NSG and Ubuntu firewall both allow TCP `25565`.

### Server starts with a new world

Check:

```properties
level-name=world
```

Verify:

```bash
ls -la data/world/level.dat
```

Also confirm the volume mapping:

```yaml
volumes:
  - ./data:/data
```

### Permission errors

```bash
sudo chown -R 1000:1000 data
sudo docker compose restart
```

### Mod compatibility errors

Verify that every installed mod supports Minecraft `26.3` and the installed Fabric loader version. Inspect logs for missing dependencies or incompatible versions.

### VPS runs out of memory

Check memory and swap usage:

```bash
free -h
sudo swapon --show
```

Check container resource usage:

```bash
sudo docker stats
```

If the server is repeatedly using swap heavily, reduce the Java heap or upgrade the VM's physical RAM. Swap is not a substitute for sufficient physical memory.

### Docker logs are consuming too much space

Inspect Docker disk usage:

```bash
sudo docker system df
```

Inspect the Minecraft container's log configuration:

```bash
sudo docker inspect minecraft --format '{{json .HostConfig.LogConfig}}'
```

The configured `max-size` and `max-file` values should be present.

---

## Security considerations

Because this server uses `online-mode=false`:

- Players can join without Microsoft account authentication.
- A player may impersonate another player's username.
- Whitelisting alone does not securely verify player identity.
- Be careful when assigning operator (`OP`) privileges.

For a private server, consider restricting access through Azure NSG source-IP rules or a private VPN such as Tailscale.

Keep the GitHub repository private if it contains player data, world files, IP bans or other private information. Never commit passwords or secrets.

---

## Useful references

- [Minecraft Docker Server – itzg](https://github.com/itzg/docker-minecraft-server)
- [Fabric Documentation](https://docs.fabricmc.net/)
- [Azure Virtual Machines Documentation](https://learn.microsoft.com/en-us/azure/virtual-machines/)
- [Azure Network Security Groups](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview)

**Repository:** https://github.com/topCodegeek/mconazure

**Server platform:** Azure Ubuntu VM + Docker Compose + Fabric