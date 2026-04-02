# Luanti-to-Minecraft Bridge

> **Connect to a Minecraft Java Edition server using a legitimate Xbox account from Linux Mint 22.2 / Ubuntu, with GeyserMC as the translation layer.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Platform: Linux](https://img.shields.io/badge/Platform-Linux%20Mint%2022.2%20%7C%20Ubuntu-orange)](https://linuxmint.com)
[![Luanti: 5.6.1+](https://img.shields.io/badge/Luanti-5.6.1%2B-brightgreen)](https://www.luanti.org)
[![Powered by GeyserMC](https://img.shields.io/badge/Powered%20by-GeyserMC-green)](https://geysermc.org)

---

## What is this?

This project provides:

1. **A web-based configuration and script generator** — fill in your server details in a UI, then download ready-to-run bash scripts for your Linux machine.
2. **Installation scripts** — automatically install Java, download GeyserMC, write config, and create a systemd service.
3. **GeyserMC configuration** — pre-configured `config.yml` pointed at your Minecraft Java server.
4. **Firewall setup scripts** — open the required UDP/TCP ports with UFW.
5. **A complete setup guide** — explains the architecture, authentication flow, and troubleshooting.

---

## Architecture

```
Luanti Client (Bedrock UDP)
        │
        ▼ UDP:19132
  ┌─────────────┐
  │  GeyserMC   │ ◄──── Microsoft OAuth2 (Xbox Login)
  │  Standalone │
  └─────────────┘
        │
        ▼ TCP:25565
  Minecraft Java Edition Server
```

**GeyserMC** is an open-source bridge that translates the Minecraft Bedrock Edition protocol (UDP) into the Java Edition protocol (TCP). Luanti clients speak a Bedrock-compatible protocol and can connect to GeyserMC.

---

## System Requirements

| Requirement | Details |
|---|---|
| **OS** | Linux Mint 22.2 "Wilma" or Ubuntu 24.04 LTS |
| **Luanti** | **5.6.1 or newer** (client and/or server) |
| **Java** | OpenJDK 21 (installed automatically by the script) |
| **RAM** | 1 GB minimum (512 MB for GeyserMC) |
| **Microsoft account** | Valid Xbox/Microsoft account required |
| **Minecraft** | Java Edition license OR server with Floodgate enabled |
| **Network** | Open UDP port (default: 19132) for Bedrock clients |
| **Privileges** | `sudo` access on the machine |

---

## Quick Start

### 1. Use the Web Configurator

Open the web app, fill in your settings, and download the generated scripts.

### 2. Transfer Scripts to Your Linux Machine

```bash
scp install.sh config.yml quickstart.sh firewall.sh user@your-linux-host:~/geyser-setup/
```

### 3. Run the Firewall Script (optional but recommended)

```bash
chmod +x firewall.sh
./firewall.sh
sudo ufw reload
```

### 4. Run the Install Script

```bash
chmod +x install.sh
./install.sh
```

The script will:
- Update apt packages
- Install OpenJDK 21
- Download the latest GeyserMC Standalone JAR
- Write `config.yml` to your install directory
- Create and enable a systemd service: `geyser-bridge`

### 5. Start the Bridge

```bash
sudo systemctl start geyser-bridge
```

### 6. Authorize Your Xbox Account

On first start, GeyserMC will print:

```
To authenticate, please visit https://microsoft.com/link
and enter the code: XXXXX-XXXXX
```

Open that URL in your browser, log in with your Microsoft/Xbox account, and enter the code. This is Microsoft's **official Device Code OAuth2 flow** — fully legal and secure.

### 7. Connect via a Luanti / Bedrock Client

Point your client to your Linux machine's IP on the configured UDP port (default `19132`).

---

## Manual Installation (without the generator)

If you prefer to set up manually:

```bash
# 1. Install Java 21
sudo apt update
sudo apt install openjdk-21-jdk curl wget screen

# 2. Create install directory
sudo mkdir -p /opt/geyser-bridge
sudo chown "$USER:$USER" /opt/geyser-bridge
cd /opt/geyser-bridge

# 3. Download GeyserMC Standalone
wget -O Geyser-Standalone.jar \
  https://download.geysermc.org/v2/projects/geyser/versions/latest/builds/latest/downloads/standalone

# 4. Run once to generate default config.yml
java -jar Geyser-Standalone.jar
# (Ctrl+C after config is generated)

# 5. Edit config.yml
nano config.yml
```

Key settings in `config.yml`:

```yaml
bedrock:
  port: 19132            # UDP port Luanti/Bedrock clients connect to

remote:
  address: play.example.com  # Your Minecraft Java server
  port: 25565
  auth-type: microsoft   # Use Xbox/Microsoft authentication
```

---

## Configuration Reference

| Option | Description | Default |
|---|---|---|
| `bedrock.port` | UDP port GeyserMC listens on | `19132` |
| `remote.address` | Minecraft Java server hostname or IP | `play.example.com` |
| `remote.port` | Minecraft Java server port (TCP) | `25565` |
| `remote.auth-type` | Auth method: `microsoft`, `offline`, `floodgate` | `microsoft` |
| `max-players` | Max simultaneous connections | `100` |
| `show-coordinates` | Show F3 coordinates in HUD | `true` |
| `debug-mode` | Verbose logging | `false` |

---

## Systemd Service

The install script creates a systemd service for automatic start and management:

```bash
# Start the bridge
sudo systemctl start geyser-bridge

# Stop the bridge
sudo systemctl stop geyser-bridge

# Restart the bridge
sudo systemctl restart geyser-bridge

# Check status
sudo systemctl status geyser-bridge

# View live logs
journalctl -u geyser-bridge -f

# View last 50 log lines
journalctl -u geyser-bridge -n 50

# Enable/disable auto-start on boot
sudo systemctl enable geyser-bridge
sudo systemctl disable geyser-bridge
```

---

## Floodgate (Optional)

[Floodgate](https://github.com/GeyserMC/Floodgate) is a companion service that allows Bedrock players to join Java servers **without owning a Java Edition license**, using only their Xbox account.

**Requirements:**
- The target Minecraft server must have the Floodgate plugin installed
- Set `auth-type: floodgate` in `config.yml`

Download Floodgate:
```bash
wget -O floodgate.jar \
  https://download.geysermc.org/v2/projects/floodgate/versions/latest/builds/latest/downloads/spigot
```

Note: Not all public servers support Floodgate. Always check a server's rules before connecting.

---

## Troubleshooting

| Problem | Solution |
|---|---|
| **GeyserMC won't start** | Check `java -version` — needs Java 17+. Check logs: `journalctl -u geyser-bridge -n 50` |
| **Cannot connect to Minecraft server** | Verify `remote.address` and port in `config.yml`. Test: `nc -zv <server-ip> 25565` |
| **Xbox login link not appearing** | Watch logs on first start: `journalctl -u geyser-bridge -f`. Link appears at startup. |
| **Port already in use** | Check: `sudo ss -ulpn \| grep 19132`. Change `bedrock.port` in `config.yml`. |
| **Firewall blocking** | Run `firewall.sh` or: `sudo ufw allow 19132/udp && sudo ufw reload` |
| **Auth token expired** | Delete `saved-refresh-tokens.json` and restart — will re-prompt for login. |
| **Out of memory** | Increase JVM heap: change `-Xmx512M` to `-Xmx1G` in the systemd service or quickstart script. |

---

## About Luanti

[Luanti](https://www.luanti.org) (formerly Minetest) is a free, open-source voxel game engine and game. It uses its own protocol for multiplayer, but its network transport is compatible with Minecraft Bedrock's UDP-based approach, allowing GeyserMC to serve as a bridge.

Install Luanti on Ubuntu/Mint:
```bash
sudo add-apt-repository ppa:minetestdevs/stable
sudo apt update
sudo apt install minetest
```

---

## Legal & Compliance

- **GeyserMC** is open-source (MIT License) — free and legal to use.
- **Microsoft OAuth2 Device Code flow** is the official, documented authentication method — no credentials are intercepted or bypassed.
- You must comply with [Minecraft's EULA](https://minecraft.net/en-us/eula) and the Terms of Service of any server you connect to.
- This tool does **not** crack, bypass, or circumvent any copy protection or authentication system.
- Luanti is licensed under LGPL 2.1+ and is completely free software.

---

## Related Projects

| Project | Description | Link |
|---|---|---|
| **GeyserMC** | Bedrock ↔ Java Edition protocol bridge | [geysermc.org](https://geysermc.org) |
| **Floodgate** | Xbox auth for Java servers without Java license | [GitHub](https://github.com/GeyserMC/Floodgate) |
| **Luanti** | Free voxel game engine (formerly Minetest) | [luanti.org](https://www.luanti.org) |
| **ProtocolLib** | Java Edition protocol manipulation library | [GitHub](https://github.com/dmulloy2/ProtocolLib) |

---

## License

MIT License — see [LICENSE](LICENSE) for details.

This project is not affiliated with Microsoft, Xbox, Mojang, or the GeyserMC team. Minecraft is a trademark of Mojang Studios.
