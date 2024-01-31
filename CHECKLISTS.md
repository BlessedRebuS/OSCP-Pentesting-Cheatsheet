A list of important checks to perform in the OSCP certification (or in a real pentest) for each protocol. 
Check https://book.hacktricks.xyz to go deeper on the service pentesting.

# Privilege Escalation Checks
## Linux PE Checks

- [g0tmi1k blog](https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation)
- [HackTricks](https://book.hacktricks.xyz/linux-hardening/privilege-escalation)
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md)
## Windows PE Checks

- [HackTricks](https://book.hacktricks.xyz/windows-hardening/checklist-windows-privilege-escalation)
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md)
- [ViperOne Pentest Everything](https://viperone.gitbook.io/pentest-everything/everything/everything-active-directory/privilege-escalation/privilege-escalation-checklist)

### Note 

After the first foothold, remember to enumerate internal networking. This can lead to private hosted Web servers, MSSQL servers or other. Using pivoiting (eg: chisel) can help to scan internal network from the Kali machine.

## Active Directory

- [Chance-Penalty-6734's useful checklist](https://drive.google.com/file/d/1sLxTSGQImCxE8KbPi063OuH461ADzCR3/view) source: [this](https://www.reddit.com/r/oscp/comments/14by0mv/my_active_directory_notes_passed_ad_in_6_hours) Reddit post
  
![Chance-Penalty-6734's mindmap](img/ad-map.png)
  
# FTP port 21

- Enumerate the service with **nmap**, usually it is available on port 21 using `nmap --script ftp-* -p 21 <IP>`
- Check if the FTP version is vulerable
- Check if it is allowed anonymous login using `ftp anonymous@<IP>` and eventually read the files
- As the last chance run a bruteforce attack `hydra -L path/to/usernames.txt -P path/to/wordlist.txt <IP> ftp`
- If gained write access and ftp is linked to a webserver, try uploading a revserse shell

# SSH port 22

- Enumerate the service both on port 22 and 2222 where it is usually hosted
- Try password spreading with found credentials
- Once in the system check /etc/passwd for other users and then perform basic bruteforce (eg: username:username)
- Check if there is also a **SFTP** share
- If there is an **RSA** key protected by password, crack it with **ssh2john** and **john**
- As the last chance run a bruteforce attack `hydra -L path/to/usernames.txt -P path/to/wordlist.txt <IP> ssh`

# WEB port 80, 443

- Enumerate the webservice with basic nmap script `nmap --script=http-enum <IP>`
- Check /robots.txt and /sitemap.xml for additional information
- Enumerate web directories with Feroxbuster, Dirbuster, Gobuster
- Check for default credentials in the exposed service (eg: admin:admin)
- If found an API endpoint, try to FUZZ it with Gobuster
- Get the version of the webserver and search on searchsploit for known exploits
- Enumerate with `whatweb` the service, find is a known exploitable CMS. (eg: umbraco)
- Enumerate with `wpscan` in order to find if the webserver runs on **WordPress** and search for plugin vulnerabilities
- If found a login page try SQL injection with some [cheatsheet](https://portswigger.net/web-security/sql-injection/cheat-sheet)
- Perform a Path Traversal and LFI when a `?page=X` is found. In case of a Windows machine try a RFI with Pass the Hash attack

# KERBEROS, 88

- Enumerate the domain with nmap
- Perform a [Kerbrute attrack](https://www.hackingarticles.in/a-detailed-guide-on-kerbrute)
- With known credentials if the Kerberos pre-authentication is not enabled (**DONT_REQ_PREAUTH**), perform a [AS-Rep roast attack](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/asreproast)
- Once in the system perform analysis on [Kerberoastable](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/kerberoast) users and eventually perform Kerberoasting
- If found any hashes with `responder` or other methods, crack them with `hashcat` or perform Relay Attack

# SNMP port 161

- Enumerate the version of the service. It runs on SNMP and requires sudo to scan `sudo nmap -p 161 -sV <IP>`
- Try `snmpwalk` on the service and get all info about MIBs, check known MIBs (users, installed programs etc..)
- Try to get more information enumerating `NET-SNMP-EXTEND-MIB::nsExtendOutputFull`

# SMB 139, 445

- Enumerate SMB version, check if signing is enabled, in order to perform Relay Attacks
- Check if SMB anonymous share access is enabled. If so download all possible data. Perform data analysis
- Spray found credentials in the domain (or not, with **--local-auth**) with `crackmapexec`, repeat the process for every username and password found
- If there are **writable folders**, try to perform `PsExec` authentication with `impacket-psexec`
- Check interesting folders on the Domain Controllers public shares
- If gained write access and SMB is linked to a webserver, try uploading a revserse shell
