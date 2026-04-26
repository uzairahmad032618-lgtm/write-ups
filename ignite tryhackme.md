1. Initial Enumeration
Nmap Scan

nmap -sS -sV <IP> -T4
• Port 80 was open
• Visiting the page showed the application was running FUEL CMS

2. Vulnerability Identification
◇ Checked the CMS version on the webpage.
◇ Searched online → That version of Fuel CMS is vulnerable to RCE (CVE‑2018‑16763).
◇ Found the exploit on exploit‑db.

3. Exploitation (Remote Code Execution)
◇ Used the exploit to execute commands on the server.
◇ Successfully obtained command execution through the CMS vulnerability.
Gaining a Shell
◇ Sent a reverse shell payload through the exploit.
◇ Got a shell on the target as the user:
 www-data

4. Privilege Escalation
Running LinPEAS
◇ Uploaded and executed linpeas.sh on the target.
LinPEAS revealed:
◇ A backup directory containing a file:
database.php
◇ Inside the file, there were hardcoded MySQL root credentials

5. Root Access
Switching User
su root
◇ Entered the found password
◇ Successfully escalated privileges to root

6. Summary
◇ Enumerated with Nmap
◇ Identified a vulnerable Fuel CMS version
◇ Used RCE exploit to gain shell as www-data
◇ Ran LinPEAS for local enumeration
◇ Found hardcoded root creds in backup config files
◇ Used su to get root access

