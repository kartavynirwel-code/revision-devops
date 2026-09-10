# Linux Fundamentals — DevOps Revision Notes

## 1. File Permissions

### Reading Permission Strings
```
-rwxr-xr--  1 user group  4096 Jan 1 10:00 script.sh
```
- First char: file type (`-` regular file, `d` directory, `l` symlink).
- Next 9 chars in groups of 3: **owner / group / others**, each as `r w x` (read/write/execute).
- **Numeric (octal) form**: `r=4, w=2, x=1`, summed per group. `rwxr-xr--` = `754`.

### Common Commands
```bash
chmod 755 script.sh          # rwxr-xr-x
chmod +x script.sh           # add execute for all
chmod u+w,g-w file.txt       # symbolic: add write for user, remove for group

chown user:group file.txt    # change owner and group
chown -R user:group /app     # recursive
```

### Special Permission Bits (interview-relevant)
- **SUID** (`chmod u+s`): Executable runs with the **file owner's** privileges, not the caller's — classic example: `/usr/bin/passwd` runs as root via SUID so a normal user can update `/etc/shadow`.
- **SGID** (`chmod g+s`): On a directory, new files inherit the **directory's group** instead of the creating user's primary group — useful for shared team directories.
- **Sticky bit** (`chmod +t`): On a directory (e.g. `/tmp`), only the file's owner (or root) can delete/rename it, even if others have write access to the directory.

## 2. Process Management

```bash
ps aux                       # all running processes, full format
ps aux | grep java           # filter for specific process
top / htop                   # live process monitor (htop is the friendlier version)

kill <pid>                   # sends SIGTERM (graceful shutdown request)
kill -9 <pid>                # sends SIGKILL (immediate, un-catchable termination)
kill -l                      # list all signal names

nohup ./script.sh &          # run process immune to hangup (SIGHUP), backgrounded
disown                       # detach a backgrounded job from the current shell

jobs                         # list background jobs in current shell
fg %1 / bg %1                # bring job 1 to foreground/background
```

### SIGTERM vs SIGKILL (interview point)
- **SIGTERM (15)**: A polite request — the process CAN catch this signal and do graceful cleanup (close DB connections, finish in-flight requests) before exiting. This is what `kubectl delete pod` sends first, and what Docker/K8s wait `terminationGracePeriodSeconds` for before escalating.
- **SIGKILL (9)**: Cannot be caught, blocked, or ignored — the kernel just terminates the process immediately. No cleanup happens. Used as the last resort after the grace period expires.

## 3. systemd (Service Management)

```bash
systemctl status nginx           # check service status
systemctl start/stop/restart nginx
systemctl enable nginx           # start automatically on boot
systemctl disable nginx
systemctl daemon-reload          # reload unit files after editing them

journalctl -u nginx              # logs for a specific service
journalctl -u nginx -f           # follow live
journalctl --since "1 hour ago"
journalctl -p err                # filter by priority level
```

### Basic Unit File Structure
```ini
[Unit]
Description=My App
After=network.target

[Service]
ExecStart=/usr/bin/java -jar /app/app.jar
Restart=on-failure
User=appuser

[Install]
WantedBy=multi-user.target
```
- **Interview point**: `Restart=on-failure` + `systemctl enable` is the traditional (pre-Kubernetes) equivalent of what a Deployment's self-healing + restart policy gives you in K8s — useful context when explaining why containerization/orchestration solves a problem systemd alone handles less flexibly at scale.

## 4. Disk and Storage

```bash
df -h                         # disk space usage per filesystem, human-readable
du -sh /var/log                # size of a specific directory
du -sh * | sort -rh | head -10 # find the biggest directories in current path

lsblk                          # list block devices (disks, partitions)
mount / umount                 # mount/unmount filesystems
fdisk -l                       # list disk partitions (need sudo)

find / -type f -size +100M     # find files larger than 100MB
find /var/log -mtime +30 -delete   # delete files older than 30 days (careful!)
```

### Interview Point — Disk Full Troubleshooting Flow
1. `df -h` → which filesystem/mount is actually full.
2. `du -sh /* 2>/dev/null | sort -rh | head` → which top-level directory is the culprit.
3. Drill down recursively into that directory with the same `du` pattern.
4. Common culprits: `/var/log` (unrotated logs), Docker's `/var/lib/docker` (dangling images/containers/volumes — `docker system prune`), old kernel versions in `/boot`.

## 5. Text Processing (grep, awk, sed)

```bash
grep "ERROR" app.log                  # find matching lines
grep -i "error" app.log               # case-insensitive
grep -v "DEBUG" app.log               # invert match (exclude)
grep -r "TODO" ./src                  # recursive search in directory
grep -E "error|warn" app.log          # extended regex (OR)

awk '{print $1, $3}' file.txt         # print specific columns (whitespace-delimited)
awk -F: '{print $1}' /etc/passwd      # custom delimiter
awk '$3 > 100 {print $0}' data.txt    # conditional filtering

sed 's/foo/bar/' file.txt             # replace first occurrence per line
sed 's/foo/bar/g' file.txt            # replace all occurrences
sed -i 's/foo/bar/g' file.txt         # edit file in-place
sed -n '5,10p' file.txt               # print only lines 5-10
```

### Combining with Pipes (common real-world pattern)
```bash
cat access.log | grep "500" | awk '{print $1}' | sort | uniq -c | sort -rn
# find which IP hit the most 500 errors: filter → extract IP column →
# sort → count duplicates → sort by count descending
```
- **Interview point**: This pipe pattern (`grep | awk | sort | uniq -c | sort -rn`) is one of the most common "find the top N occurrences of X" idioms in log analysis — worth having memorized cold.

## 6. Networking Commands

```bash
curl -I https://example.com           # headers only (HEAD-like request)
curl -v https://example.com           # verbose, shows full request/response + TLS handshake
curl -X POST -d '{"key":"val"}' -H "Content-Type: application/json" url

ss -tulpn                              # modern replacement for netstat — listening ports + process
netstat -tulpn                         # (older, may not be installed by default now)

nslookup example.com / dig example.com  # DNS lookup
dig +short example.com                  # just the IP, no verbose output
traceroute example.com                  # hop-by-hop path to a host
ping -c 4 example.com                   # basic reachability check, 4 packets

nc -zv host port                        # quick TCP port reachability test (netcat)
```

### Interview Point — Debugging "Can't Reach a Service"
> "My order of operations: `ping` (basic reachability, though often blocked by firewalls so not conclusive alone) → `nc -zv host port` or `curl` (actual TCP/HTTP-level reachability) → `dig`/`nslookup` (is DNS resolving to the right IP at all) → `ss -tulpn` on the target host (is anything actually listening on that port)."

---

## Quick Self-Test (do this without looking)
1. What's the difference between SIGTERM and SIGKILL, and why does Kubernetes send one before the other?
2. What does the SUID bit actually change about how a program runs, using `passwd` as the example?
3. Write the pipe chain to find the top 5 most frequent IPs causing 500 errors in an access log.
4. Your disk is full — walk through your exact troubleshooting command sequence to find the cause.
