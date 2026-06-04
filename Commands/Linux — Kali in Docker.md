## What & When

Run **Kali Linux** in **Docker** for portable security labs. Concept: [[Linux — Kali & Security Labs]]. Overview: [[Linux]]. Base Docker: [[Commands/CLI — Docker & Compose]].

> Authorized testing only — isolated networks.

---

## Prerequisites

```bash
# Debian/Ubuntu host
sudo apt update
sudo apt install -y docker.io docker-compose-plugin
sudo systemctl enable --now docker

docker --version
docker run hello-world
```

```bash
# macOS
brew install --cask docker
# or: brew install docker docker-compose
```

Optional — run without sudo:

```bash
sudo usermod -aG docker "$USER"
newgrp docker
```

---

## Official images

| Image | Use |
| --- | --- |
| `kalilinux/kali-rolling` | Base |
| `kalilinux/kali-linux-headless` | No GUI |
| `kalilinux/kali-tools-top10` | Common tools |
| `kalilinux/kali-tools-web` | Web testing |
| `kalilinux/kali-tools-all` | Large |

---

## Basic container

```bash
docker pull kalilinux/kali-rolling

docker run -it kalilinux/kali-rolling /bin/bash

# inside:
apt update && apt install -y kali-linux-top10
exit
```

---

## Named persistent container

```bash
docker run -it --name kali kalilinux/kali-rolling /bin/bash
docker start -ai kali
```

Without volumes, installed tools are lost when the container is **removed** (not just stopped).

---

## Volume mount (reports & wordlists)

```bash
mkdir -p ~/kali-data
docker run -it \
  -v ~/kali-data:/root/data \
  --name kali \
  kalilinux/kali-rolling /bin/bash
```

---

## Headless / detached

```bash
docker run -d --name kali \
  -v ~/kali-data:/root/data \
  kalilinux/kali-rolling sleep infinity

docker exec -it kali bash
```

---

## Network modes

```bash
# Default bridge (safer)
docker run -it kalilinux/kali-rolling

# Host network — labs only, Linux host
docker run -it --network host kalilinux/kali-rolling
```

---

## Capabilities (isolated labs only)

```bash
docker run -it \
  --cap-add=NET_ADMIN \
  --cap-add=NET_RAW \
  kalilinux/kali-rolling
```

---

## Tool metapackages (inside container)

```bash
apt install -y kali-tools-top10
apt install -y kali-tools-web
apt install -y kali-tools-passwords
dpkg -l | grep kali
```

---

## Custom Dockerfile

```dockerfile
FROM kalilinux/kali-rolling

RUN apt update && apt install -y \
    kali-linux-top10 \
    wordlists \
    && apt clean

WORKDIR /root
CMD ["/bin/bash"]
```

```bash
docker build -t kali-custom .
docker run -it kali-custom
```

---

## Docker Compose

```yaml
services:
  kali:
    image: kalilinux/kali-rolling
    container_name: kali
    stdin_open: true
    tty: true
    volumes:
      - ./data:/root/data
```

```bash
docker compose up -d
docker exec -it kali bash
```

---

## GUI (Linux host + X11)

```bash
xhost +local:
docker run -it \
  -e DISPLAY="$DISPLAY" \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  kalilinux/kali-rolling
```

---

## Limitations

| Limitation | Detail |
| --- | --- |
| No full systemd | Service units behave differently |
| Wireless attacks | Usually **fail** in Docker |
| Kernel modules | Not available |
| **Good for** | Web recon, scripting, tool CLI |

Use a **VM** for OSCP-style wireless and low-level labs.

---

## Docker vs VM

| Docker | VM |
| --- | --- |
| Fast, light | Full hardware access |
| Easy reset | Wireless support |
| CI-friendly | Exam-realistic |

---

## Quick reference

```bash
docker pull kalilinux/kali-rolling
docker run -it --name kali -v ~/kali-data:/root/data kalilinux/kali-rolling
docker exec -it kali bash
docker ps -a
docker rm -f kali
```

---

## Related Notes

- [[Linux — Kali & Security Labs]]
- [[Linux]]
- [[Commands/Linux — Kali Tools]]
- [[Commands/CLI — Docker & Compose]]

---

## Tags

#linux #kali #docker #security #lab #headless
