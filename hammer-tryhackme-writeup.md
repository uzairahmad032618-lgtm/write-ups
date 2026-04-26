Improved & Cleaned Writeup
# HAMMER Room Writeup (TryHackMe)

## Overview
This writeup documents my approach to solving the **HAMMER** room on TryHackMe (Web Pentesting Path).  
The lab focuses on real-world authentication attacks, including enumeration, OTP brute-forcing, JWT manipulation, and privilege escalation.

---

## Recon & Enumeration

Initial port scanning:

```bash
sudo nmap -sS -sV -p1-1000 <IP> -T4
sudo nmap -sS -sV -p1000-2000 <IP> -T4
Findings:
Port 22 (SSH) open
Port 1337 hosting a web application (login page)
 Authentication Bypass Attempts

Multiple techniques were attempted:

Hydra brute-force →  Failed
Burp Suite →  Blocked due to rate limiting
Python script (2200 common emails) →  No success
Directory fuzzing tools → Initially no results
 Source Code Review & Directory Fuzzing

While analyzing the source code, a naming pattern for directories was discovered.

Custom wordlist creation:
sed 's/^/hmr_/' /usr/share/wordlists/dirb/common.txt > hmr_dirs.txt
Fuzzing:
feroxbuster -u http://<IP>:1337 -w hmr_dirs.txt
Result:
Discovered /logs directory
Contained login attempt logs
Extracted a valid email address
 Password Reset & OTP Bruteforce

Triggered password reset for the discovered user.

Challenge:
OTP was time-restricted
Burp Suite failed due to rate-limiting
Solution: ffuf-based brute-force
ffuf -w otp.txt -u "http://<IP>:1337/reset_password.php" \
-X POST \
-d "recovery_code=FUZZ&s=147" \
-H "Cookie: PHPSESSID=td9tt0ubmj3qo6a1fe3ldbvl64" \
-H "X-Forwarded-For: FUZZ" \
-H "Content-Type: application/x-www-form-urlencoded" \
-fr "Invalid" -s
Result:
Successfully discovered a valid OTP
Gained access to the user account
 JWT Analysis & Session Behavior

After login:

JWT tokens observed
iat and exp timestamps were identical
Session was not persistent
Interesting Finding:
JWT contained a kid (Key ID) field
Indicated potential for JWKS spoofing
 JWKS Spoofing & Privilege Escalation

Discovered a secret key file accessible on the server:

curl http://<IP>:1337/path/to/secret
Attack Steps:
Extracted the secret key
Modified JWT payload:
role: user → role: admin
Set kid to point to the secret file
Re-signed the JWT
Final Step:
Replaced:
Authorization header
JWT cookie
Result:
Gained admin access
Retrieved the final flag
 Conclusion

The HAMMER room provided hands-on experience in:

Advanced enumeration techniques
Directory fuzzing with custom wordlists
OTP brute-forcing under constraints
JWT analysis and manipulation
JWKS spoofing for privilege escalation

A highly practical lab simulating real-world web application vulnerabilities.

 Tools Used
Nmap
Feroxbuster
ffuf
Burp Suite
curl
Custom Python scripts
