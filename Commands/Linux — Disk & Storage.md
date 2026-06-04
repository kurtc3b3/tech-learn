## What & When

**Disk usage**, block devices, and **mount**. Concept: [[Linux — Architecture]]. Overview: [[Linux]].

---

## Usage

```bash
df -h
df -i          # inodes
du -sh /var/log
du -h --max-depth=1 /home
ncdu /var      # interactive (apt install ncdu)
```

```bash
brew install ncdu   # macOS
```

---

## Block devices

| Command | Description |
| --- | --- |
| `lsblk` | Tree of disks/partitions |
| `blkid` | UUID / filesystem type |
| `fdisk -l` | Partitions (root) |

---

## Mount

```bash
mount
findmnt
sudo mount /dev/sdb1 /mnt/data
sudo umount /mnt/data
```

`/etc/fstab` — persistent mounts (edit carefully).

---

## sync

```bash
sync   # flush buffers before unplugging disk
```

---

## LVM (concept)

```bash
pvs
vgs
lvs
```

Used on many servers; detailed admin is outside this vault — use distro docs.

---

## Related Notes

- [[Linux]]
- [[Commands/Linux — Files Advanced]]
- [[Commands/Linux — Packages & Archives]]
- [[K8S]] — PVCs

---

## Tags

#linux #disk #mount #df #du #lsblk
