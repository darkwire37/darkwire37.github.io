## Introduction
RootMe is a basic boot-to-root CTF on TryHackMe.  I figured it would be a nice and easy box to document as I get used to blogging.

## Enumeration
As always, enumeration/recon is the first step of the engagement.  This is infosec after all, information is our game.  The more we have, the more power we hold.
Let's start by finding services with the classic nmap.

### Nmap
I use the nmap with the -A flag (because it's a CTF and I can throw as much traffic as I want at it) to find just about everything. 
I also use the -sV flag as it gives just a little bit more info about the service than -A does.   I then output the scan to the nmap.scan file in the project's directory with the -o flag.  As with "real world" engagements, documenting as you go is a must.  
```
sudo nmap -A -sV 10.10.121.102 -o nmap.scan
Starting Nmap 7.92 ( https://nmap.org ) at 2022-07-25 13:13 EDT
Nmap scan report for 10.10.121.102
Host is up (0.22s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4a:b9:16:08:84:c2:54:48:ba:5c:fd:3f:22:5f:22:14 (RSA)
|   256 a9:a6:86:e8:ec:96:c3:f0:03:cd:16:d5:49:73:d0:82 (ECDSA)
|_  256 22:f6:b5:a6:54:d9:78:7c:26:03:5a:95:f3:f9:df:cd (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: HackIT - Home
|_http-server-header: Apache/2.4.29 (Ubuntu)
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.92%E=4%D=7/25%OT=22%CT=1%CU=40078%PV=Y%DS=4%DC=T%G=Y%TM=62DECF7
OS:C%P=x86_64-pc-linux-gnu)SEQ(SP=109%GCD=1%ISR=10B%TI=Z%CI=Z%II=I%TS=A)SEQ
OS:(SP=109%GCD=1%ISR=10A%TI=Z%CI=Z%TS=A)OPS(O1=M505ST11NW7%O2=M505ST11NW7%O
OS:3=M505NNT11NW7%O4=M505ST11NW7%O5=M505ST11NW7%O6=M505ST11)WIN(W1=F4B3%W2=
OS:F4B3%W3=F4B3%W4=F4B3%W5=F4B3%W6=F4B3)ECN(R=Y%DF=Y%T=40%W=F507%O=M505NNSN
OS:W7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%D
OS:F=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O
OS:=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W
OS:=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%R
OS:IPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 4 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 53/tcp)
HOP RTT       ADDRESS
1   82.64 ms  10.2.0.1
2   ... 3
4   227.51 ms 10.10.121.102

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 38.93 seconds
```
As we see here, ports 22 running SSH and 80 running HTTP are open.  I know from previous experience that neither of these version numbers have known vulnerabilities in them (at least, toward getting a shell), so I pass over those.  
The next thing to do is check the website for potential misconfigurations and vulnerable code.  

### Web Enumeration
My go to tool of choice for enumerating websites is GoBuster.  It's like dirb but faster and fancier.  As with nmap, I output the scan to a file (gobuster.scan), but I have to use tee for this one. 
```
sudo gobuster dir -u http://10.10.121.102/ -w /usr/share/wordlists/dirb/big.txt | tee gobuster.scan
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.121.102/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/big.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2022/07/25 13:29:36 Starting gobuster in directory enumeration mode
===============================================================
/.htaccess            (Status: 403) [Size: 278]
/.htpasswd            (Status: 403) [Size: 278]
/css                  (Status: 301) [Size: 312] [--> http://10.10.121.102/css/]
/js                   (Status: 301) [Size: 311] [--> http://10.10.121.102/js/] 
/panel                (Status: 301) [Size: 314] [--> http://10.10.121.102/panel/]
/server-status        (Status: 403) [Size: 278]                                  
/uploads              (Status: 301) [Size: 316] [--> http://10.10.121.102/uploads/]
Progress: 20469 / 20470 (100.00%) 
===============================================================
2022/07/25 13:37:15 Finished
===============================================================
```
Here we find the standard htaccess, css, js, etc. directories, but then we also find /panel, /server-status, and /uploads.  
My initial thought is that if we have some way to upload files on the panel, we can LFI our way to executing a php webshell.  This will depend of course, on the contents of the panel.  

> Uploading this portion so far, going to learn how to upload images next and continue. 
