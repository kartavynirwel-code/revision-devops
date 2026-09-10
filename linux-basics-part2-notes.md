# Linux Basics Part 2 — More Interview Topics

## 1. Linux Boot Process (classic interview question)
```
1. BIOS/UEFI  → Power-on self-test, finds the boot device
2. Bootloader (GRUB)  → Loads the Linux kernel into memory
3. Kernel initialization  → Mounts initial root filesystem (initramfs), 
                            loads essential drivers
4. init system (systemd, PID 1)  → Takes over, mounts real root filesystem,
                                    starts all system services per target/runlevel
5. Target reached (e.g. multi-user.target / graphical.target)  → 
   Login prompt / services fully up
```
- **Interview point**: Know that `systemd` (PID 1) replaced the older SysV `init` on most modern distros — it manages services as **units** with dependency-based parallel startup instead of the older strictly sequential runlevel scripts.

## 2. Linux File System Hierarchy (FHS)
| Path | Purpose |
|---|---|
| `/etc` | System-wide configuration files |
| `/var` | Variable data — logs (`/var/log`), caches, spool files |
| `/home` | User home directories |
| `/usr` | User-installed programs and libraries (despite the name, not "user" data) |
| `/bin`, `/sbin` | Essential binaries (user commands / system admin commands) |
| `/tmp` | Temporary files, often cleared on reboot |
| `/opt` | Optional/third-party software packages |
| `/proc` | Virtual filesystem — live kernel/process info (not real files on disk) |
| `/dev` | Device files (e.g., `/dev/sda`, `/dev/null`) |
- **Interview point on `/proc`**: It's a pseudo-filesystem — reading `/proc/<pid>/status` or `/proc/meminfo` gives live kernel data, not stored files. This is literally how tools like `top`/`ps` get their information.

## 3. Users, Groups, and sudo

```bash
whoami                          # current user
id                               # current UID, GID, group memberships
useradd -m -s /bin/bash newuser  # create user with home dir + bash shell
usermod -aG docker newuser       # add existing user to a group (docker, in this case)
passwd newuser                   # set/change password

groups newuser                   # list groups a user belongs to
cat /etc/passwd                  # user account database (name:x:UID:GID:info:home:shell)
cat /etc/group                   # group database
```

### sudo vs su
- **`su`**: Switches to another user entirely (commonly root), requires that **user's** password. Starts a new shell session as them.
- **`sudo`**: Runs a **single command** as another user (default root), authenticated with **your own** password, and logged (`/var/log/auth.log` or similar) — the more auditable, least-privilege-friendly approach.
- **`/etc/sudoers`** (edit only via `visudo` — validates syntax before saving, preventing you from locking yourself out with a typo): defines who can run what as whom.
- **Interview point**: `sudo` is generally preferred in production/team environments because every privileged action is individually logged and attributable to a specific user, whereas `su` just drops you into a root shell with no per-command audit trail.

## 4. Package Management

```bash
# Debian/Ubuntu (APT)
apt update                       # refresh package index
apt install nginx
apt remove nginx
apt list --installed | grep nginx

# RHEL/CentOS/Fedora (YUM/DNF)
dnf install nginx
dnf remove nginx
dnf list installed | grep nginx

# Universal/language-specific
pip install requests --break-system-packages
npm install -g pm2
```
- **Interview point**: Know the family split — Debian-based (Ubuntu, Debian) use `.deb` packages via `apt`/`dpkg`; RHEL-based (CentOS, RHEL, Fedora, Amazon Linux) use `.rpm` packages via `dnf`/`yum`/`rpm`. Relevant when choosing base images for Docker or provisioning EC2/VMs.

## 5. Cron (Scheduled Tasks)

```bash
crontab -e                       # edit current user's crontab
crontab -l                       # list current cron jobs
```

### Cron Syntax
```
* * * * * command
│ │ │ │ │
│ │ │ │ └── day of week (0-6, Sunday=0)
│ │ │ └──── month (1-12)
│ │ └────── day of month (1-31)
│ └──────── hour (0-23)
└────────── minute (0-59)
```
- Example: `0 2 * * *` = every day at 2:00 AM (this is exactly the Velero schedule syntax from Day 14 — same cron format).
- `*/15 * * * *` = every 15 minutes.
- System-wide cron jobs also live in `/etc/cron.d/`, `/etc/crontab`, and `/etc/cron.{hourly,daily,weekly,monthly}/`.

## 6. Environment Variables

```bash
export MY_VAR="value"            # set for current shell + child processes
echo $MY_VAR
unset MY_VAR

env                               # list all environment variables
printenv PATH                    # print one specific variable

# Persisting across sessions:
# ~/.bashrc or ~/.zshrc (per-user, per-shell)
# /etc/environment (system-wide)
```
- **`PATH`**: Colon-separated list of directories the shell searches (in order) to find executables when you type a command name. `which java` shows exactly which PATH directory's binary gets used.
- **Interview point**: Difference between `export VAR=x` and just `VAR=x` — without `export`, the variable is only visible in the **current shell**, not passed down to child processes/scripts you run from it.

## 7. Hard Links vs Symbolic (Soft) Links

```bash
ln original.txt hardlink.txt          # hard link
ln -s original.txt symlink.txt        # symbolic link
```
| | Hard Link | Symbolic Link |
|---|---|---|
| What it points to | The same inode (actual data) as the original | A path/name pointing to the original file |
| Survives original deletion? | Yes — data persists as long as ANY hard link exists | No — becomes a broken/dangling link |
| Cross-filesystem? | No — must be same filesystem | Yes — can point across filesystems |
| Can link a directory? | No (generally disallowed) | Yes |
- **Interview point**: `df` shows disk usage per filesystem, and inodes are the actual data structures holding file metadata + data block pointers — a hard link is just another directory entry pointing to the **same inode**, which is why deleting the "original" doesn't delete the data until the last hard link referencing that inode is removed.

## 8. File Descriptors and Redirection

```bash
command > output.txt              # redirect stdout (fd 1), overwrite
command >> output.txt             # redirect stdout, append
command 2> error.txt              # redirect stderr (fd 2)
command > output.txt 2>&1         # redirect stdout AND stderr to same file
command &> output.txt             # shorthand for the line above (bash-specific)
command < input.txt               # redirect stdin (fd 0) from a file

command1 | command2               # pipe — command1's stdout becomes command2's stdin
```
- **Interview point**: `2>&1` must come **after** the `>` redirect to work correctly — `command 2>&1 > output.txt` (wrong order) sends stderr to the terminal (wherever stdout was pointing at that moment) and only stdout to the file, NOT what most people intend.

## 9. SSH Basics

```bash
ssh user@host                          # basic connection
ssh -i ~/.ssh/mykey.pem user@host      # specify private key
ssh -p 2222 user@host                  # non-default port

ssh-keygen -t ed25519 -C "email@example.com"   # generate a keypair
ssh-copy-id user@host                  # copy your public key to a remote host's authorized_keys

# ~/.ssh/config — save connection shortcuts
Host myserver
    HostName 192.168.1.10
    User admin
    Port 2222
    IdentityFile ~/.ssh/mykey.pem
```
- **Interview point**: Public key auth flow — your **private key** never leaves your machine; the server has your **public key** in `~/.ssh/authorized_keys`. During auth, the server sends a challenge encrypted with your public key, and only your private key can decrypt/sign it correctly — this is why leaking a public key is harmless but leaking a private key is a full compromise.

---

## Quick Self-Test (do this without looking)
1. What actually takes over after the kernel finishes initializing, and what replaced the old sequential runlevel system?
2. Why is `sudo` generally preferred over `su` in a team/production environment?
3. You delete the "original" file that has 2 hard links pointing to the same inode — does the data disappear? Why or why not?
4. Why does `command 2>&1 > output.txt` NOT redirect stderr into `output.txt`, but `command > output.txt 2>&1` does?
