# Linux Basics Part 3 — Commonly Used Commands (Interview-Ready)

## 1. System Info Commands (quick "tell me about this machine" questions)
```bash
uname -a                # kernel name, version, architecture — full system info
hostnamectl              # hostname + OS info (systemd systems)
cat /etc/os-release       # distro name and version
uptime                    # how long the system's been running + load average
free -h                   # RAM usage (human-readable): total/used/free/buff-cache
nproc                     # number of CPU cores available
lscpu                     # detailed CPU info (cores, threads, architecture)
```

### Reading `uptime`'s Load Average (common interview gotcha)
```
load average: 0.52, 0.58, 0.65
```
- Three numbers = average number of processes waiting for CPU over the **last 1, 5, and 15 minutes**.
- **Interview point**: A load average of `4.0` means nothing on its own — you must compare it to `nproc` (core count). Load `4.0` on a 4-core machine = fully utilized (roughly 100%); load `4.0` on a 16-core machine = mostly idle (~25%). Never state "load is high" without knowing core count.

## 2. Archiving and Compression

```bash
tar -czvf archive.tar.gz /path/to/dir      # create (c) gzip-compressed (z) archive, verbose (v), file (f)
tar -xzvf archive.tar.gz                    # extract
tar -tzvf archive.tar.gz                    # list contents WITHOUT extracting

gzip file.txt              # compress (produces file.txt.gz, removes original)
gunzip file.txt.gz         # decompress

zip -r archive.zip folder/
unzip archive.zip
```
- **Interview point on flags**: `c`=create, `x`=extract, `z`=gzip filter, `v`=verbose, `f`=file (must come last since it expects the filename right after) — muscle-memory command, expect to be asked to write it from scratch.

## 3. File Transfer — scp and rsync

```bash
scp file.txt user@host:/remote/path/          # copy a file to remote host
scp -r folder/ user@host:/remote/path/        # recursive, for directories
scp -i key.pem user@host:/remote/file.txt .   # with a specific key, copy FROM remote

rsync -avz source/ user@host:/remote/dest/    # a=archive mode, v=verbose, z=compress
rsync -avz --delete source/ dest/             # mirror exactly, deleting extra files in dest
rsync --dry-run -avz source/ dest/            # preview what would happen, no changes made
```
- **Interview point — scp vs rsync**: `scp` copies everything every time. `rsync` only transfers the **differences** (delta) between source and destination, making repeated syncs of large directories dramatically faster — this is why rsync is preferred for backups and repeated deployments over scp.

## 4. Process Priority — nice / renice

```bash
nice -n 10 ./heavy-script.sh       # start a process with LOWER priority (higher nice value = lower priority)
renice -n -5 -p 1234               # change priority of an already-running process (PID 1234)
```
- Nice values range from **-20 (highest priority) to 19 (lowest priority)**. Default is 0.
- **Interview point**: Counter-intuitive naming — a "nicer" process (higher nice value) is more polite/yields more CPU to others, hence lower actual scheduling priority. Only root can set negative (higher-priority) nice values.

## 5. Disk/CPU I/O Monitoring

```bash
vmstat 2                  # system stats every 2 seconds: CPU, memory, swap, I/O
iostat -x 2               # detailed per-disk I/O stats (needs sysstat package)
iotop                     # like top, but for disk I/O per-process (needs sudo)
sar -u 1 3                # historical/live CPU usage samples (needs sysstat)
```
- **Interview point**: If an app is slow and CPU/memory look fine, `iostat`/`iotop` is how you check if it's actually **disk I/O bound** — e.g., a database doing heavy writes on slow storage.

## 6. `find` — Common Real-World Variations

```bash
find /var/log -name "*.log"                   # by name pattern
find . -type f -mtime -7                       # modified in the last 7 days
find . -type d -empty                          # empty directories
find /app -perm 777                            # files with specific permission
find . -name "*.tmp" -exec rm {} \;             # find and delete matches
find . -name "*.tmp" | xargs rm                 # same idea via xargs (often faster for many files)

find / -type f -size +500M 2>/dev/null          # large files, suppress permission-denied noise
```
- **Interview point — `-exec` vs `xargs`**: `-exec ... {} \;` runs the command **once per file** (slower for huge result sets). `| xargs` batches multiple filenames into fewer command invocations, generally faster — `xargs` is preferred for large-scale operations, but be careful with filenames containing spaces (`find ... -print0 | xargs -0 ...` handles that safely).

## 7. `xargs` on Its Own

```bash
echo "file1 file2 file3" | xargs rm
cat urls.txt | xargs -I{} curl -O {}            # run a command once per input line, {} = placeholder
find . -name "*.log" -print0 | xargs -0 gzip    # safe handling of filenames with spaces/special chars
```
- **Interview point**: `xargs` converts stdin lines into command-line **arguments** for another command — useful because many commands (like `rm`, `curl`) don't read from stdin directly, they expect arguments.

## 8. `diff` and `comm`

```bash
diff file1.txt file2.txt        # show line-by-line differences
diff -u file1.txt file2.txt     # unified diff format (like a git diff)
diff -r dir1/ dir2/             # recursively compare directories

comm file1.txt file2.txt        # compare SORTED files, 3 columns: unique-to-1, unique-to-2, common
```

## 9. Log Rotation (logrotate)
- Prevents log files from growing forever and filling disk (ties back to Day 6's disk-full troubleshooting).
- Config lives in `/etc/logrotate.d/<service>`:
```
/var/log/myapp/*.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
}
```
- **Interview point**: `daily` + `rotate 7` = keep 7 days of rotated logs, compress old ones, don't error if the log is missing, don't rotate if the file is empty. This runs via a cron job/systemd timer automatically on most distros — you rarely trigger it manually, but you should know how to configure it.

## 10. Networking/Firewall — quick recall

```bash
ufw status                        # Ubuntu's simplified firewall status
ufw allow 22/tcp                  # allow SSH
ufw enable

iptables -L -n                    # list current iptables rules (lower-level, what ufw wraps)

curl ifconfig.me                  # quick way to check your machine's public IP
```

## 11. Rapid-Fire "What Does This Command Do" (self-quiz format)
Try explaining each of these out loud in one sentence, without looking:
1. `chmod -R 644 /app/config`
2. `ps aux | grep -v grep | grep node`
3. `df -h | grep -v tmpfs`
4. `tail -f /var/log/syslog | grep ERROR`
5. `du -sh /* 2>/dev/null | sort -rh | head -5`
6. `kill -9 $(pgrep -f myapp)`
7. `awk -F: '{print $1}' /etc/passwd | sort`
8. `find / -name "*.conf" -newer /etc/hosts`
9. `netstat -tulpn | grep :8080` (or `ss -tulpn | grep :8080`)
10. `history | grep docker`

---

## Quick Self-Test (do this without looking)
1. What's the actual performance difference between `scp` and `rsync` for repeated backups, and why?
2. A load average shows `8.0` — is the server overloaded? What extra piece of information do you need to answer that?
3. Why is `find ... | xargs` sometimes preferred over `find ... -exec ... \;`, and what's the safe way to handle filenames with spaces in that pipeline?
4. Explain what a `nice` value of `-10` vs `10` actually means for a process's CPU scheduling priority.
