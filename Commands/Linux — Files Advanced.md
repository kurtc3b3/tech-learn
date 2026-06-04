## What & When

**Advanced file operations** — links, rsync, ACLs, integrity, temp files. Overview: [[Linux]].

---

## Links

| Command | Description |
| --- | --- |
| `ln file link` | Hard link |
| `ln -s target link` | Symbolic link |
| `readlink link` | Symlink target |
| `realpath file` | Canonical path |

---

## rsync (Sync & Backup)

| Command | Description |
| --- | --- |
| `rsync -av src/ dest/` | Archive + verbose |
| `rsync -av --delete src/ dest/` | Mirror (destructive on dest) |
| `rsync -av --progress src/ dest/` | Show progress |

```bash
rsync -av --progress /data/ user@backup:/backups/data/
```

Use for deploy artifacts, log archives, and VM migrations.

---

## Permissions & ACLs

| Command | Description |
| --- | --- |
| `lsattr file` | Extended attributes |
| `chattr +i file` | Immutable |
| `chattr -i file` | Remove immutable |
| `getfacl file` | View ACL |
| `setfacl -m u:user:rwx file` | Set ACL |

Basics: [[Commands/Linux — Permissions & Users]].

---

## Compare & Integrity

| Command | Description |
| --- | --- |
| `diff file1 file2` | Line diff |
| `cmp file1 file2` | Binary compare |
| `md5sum file` | MD5 |
| `sha256sum file` | SHA-256 |

---

## Line Endings & Type

| Command | Description |
| --- | --- |
| `dos2unix file` | CRLF → LF |
| `unix2dos file` | LF → CRLF |
| `file -i file` | MIME type |

---

## Split / Merge

```bash
split -b 100M large.iso part_
cat part_* > large.iso
```

---

## Open Files & Locks

| Command | Description |
| --- | --- |
| `lsof file` | Processes using file |
| `lsof +D /var/log` | Under directory |
| `fuser -v /var/log/syslog` | PIDs using file |

```bash
lsof -i :8000   # who listens on port 8000
```

---

## Temp & Timestamps

```bash
mktemp
mktemp -d
touch -t 202601011200 file
ls -lt
```

---

## Tree & Size

```bash
tree -L 2 /etc/nginx
du -sh *
du -h --max-depth=1 /var
```

```bash
brew install tree   # macOS
```

---

## Related Notes

- [[Linux]]
- [[Commands/Linux — Essentials]]
- [[Commands/Linux — Disk & Storage]]

---

## Tags

#linux #rsync #acl #links #files
