## What & When

**Permissions**, ownership, and user administration. Concept: [[Linux — Architecture]]. Overview: [[Linux]].

---

## View Permissions

```bash
ls -l file
# -rw-r--r-- 1 alice dev 1234 Jan  1 12:00 file
```

| Field | Meaning |
| --- | --- |
| `-rw-r--r--` | type + user/group/other bits |
| `alice` | owner |
| `dev` | group |

---

## chmod

| Command | Description |
| --- | --- |
| `chmod 755 script.sh` | rwxr-xr-x |
| `chmod +x script.sh` | Add execute |
| `chmod u+w file` | User write |
| `umask` | Default mask for new files |

Octal: `7=rwx`, `6=rw-`, `5=r-x`, `4=r--`

---

## chown / chgrp

```bash
chown user file
chown user:group file
chgrp group file
```

---

## Identity

| Command | Description |
| --- | --- |
| `whoami` | Current user |
| `id` | UID, groups |
| `id alice` | User info |
| `groups user` | Group list |

---

## sudo

```bash
sudo cmd
sudo -u www-data cmd
visudo   # edit sudoers safely
```

---

## User admin (root)

| Command | Description |
| --- | --- |
| `useradd -m user` | Add user + home |
| `userdel -r user` | Delete + home |
| `passwd user` | Set password |
| `groupadd group` | Add group |
| `usermod -aG docker user` | Append group |

---

## su

```bash
su - otheruser    # login shell
sudo -i           # root login shell
```

---

## Related Notes

- [[Linux]]
- [[Commands/Linux — Files Advanced]]
- [[K8S]] — pod security contexts

---

## Tags

#linux #permissions #chmod #sudo #users
