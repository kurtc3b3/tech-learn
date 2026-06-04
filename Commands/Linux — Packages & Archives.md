## What & When

**Package managers** (Debian/RHEL) and **archives**. Overview: [[Linux]].

---

## Debian / Ubuntu (apt)

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y pkg
sudo apt remove pkg
sudo apt search keyword
apt show pkg
```

```bash
# Install build tools
sudo apt install -y build-essential
```

---

## RHEL / CentOS / Fedora

```bash
sudo dnf install pkg      # modern
sudo yum install pkg      # older
sudo dnf remove pkg
```

---

## Snap / Flatpak (optional desktops)

```bash
snap install code --classic
flatpak install flathub org.example.App
```

---

## tar & gzip

| Command | Description |
| --- | --- |
| `tar -cvf a.tar dir` | Create tar |
| `tar -xvf a.tar` | Extract |
| `tar -czvf a.tar.gz dir` | gzip compress |
| `tar -xzvf a.tar.gz` | Extract gzip |
| `tar -cjvf a.tar.bz2 dir` | bzip2 |

```bash
tar -czvf backup-$(date +%F).tar.gz /etc/nginx/
```

---

## zip

```bash
zip -r archive.zip dir/
unzip archive.zip
```

---

## dpkg (low-level Debian)

```bash
dpkg -l | grep nginx
dpkg -i package.deb
```

---

## Related Notes

- [[Linux]]
- [[Commands/Linux — Disk & Storage]]
- [[Commands/CLI — Docker & Compose]]

---

## Tags

#linux #apt #dnf #tar #packages
