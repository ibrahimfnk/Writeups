
## Learning Objectives

- Enumerate the target domain and network.
- Enumerate valid domain users and identify misconfigurations.
- Perform password spraying attacks to get your first pair of valid AD credentials.
- Discover sensitive credentials stored in configuration files.


##### Scope: `10.211.11.0/24`

## Mapping out the Network
### Host Discovery
________________________
- We are given with a subnet IP range
- Our first task is to run a **subnet host discovery scan**
- This allows us to discovery live hosts

#### Tool 1: fping
- fping uses **ICMP** requests just like **ping**
- However, with fping we can specify any number of targets (including subnets)
- Instead of sending one request at a time, it moves onto the next target until the reply is received


	`fping -agq subnet-range`
	Ex: `fping -agq 10.211.11.0/24`

	 `-a`: shows systems that are alive.
	 `-g`: generates a target list from a supplied IP netmask.
	 `-q`: quiet mode, doesn't show per-probe results or ICMP error messages.

~~~~~~~~~~
10.211.11.1 - Default gateway
10.211.11.10
10.211.11.20
10.211.11.250 - VPN Server
~~~~~~~~~~~~~~~~~~~

Thus the 2 live hosts are 

~~~~~~~~~~~~~
10.211.11.10
10.211.11.20
~~~~~~~~~~~~~~~

Storing them in a `hosts.txt` file


#### Tool 2: nmap
- can be used in ping scan mode (`-sn`)

	`nmap -sn subnet-range`
	Ex: `nmap -sn 10.211.11.0/24``


We must identify which host is the Domain Controller to determine which critical AD services are running and can be exploited. 


| Port | Protocol           | What it Means<br><br>                                                                                                                                                                                             |
| ---- | ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 88   | Kerberos           | Potential for Kerberos-based enumeration. It can be a goldmine for ticket attacks like Pass-the-Ticket and Kerberoasting.                                                                                         |
| 135  | MS-RPC             | <br>Potential for RPC enumeration (null sessions). This TCP port is used for Remote Procedure Calls (RPC). It might be leveraged to identify services for lateral movement or remote code execution via DCOM.<br> |
| 139  | SMB/NetBIOS        | Legacy SMB access. It can be abused for null sessions and information gathering.                                                                                                                                  |
| 389  | LDAP               | LDAP queries to AD. It is in plaintext and can be a prime target for enumerating AD objects, users, and policies.                                                                                                 |
| 445  | SMB                | Modern SMB access, critical for enumeration. Critical for file sharing and remote admin; abused for exploits like EternalBlue, SMB relay attacks, and credential theft.                                           |
| 464  | Kerberos (kpasswd) | Password-related Kerberos service                                                                                                                                                                                 |
| 636  | LDAPS              | This port is used by Secure LDAP. Although it is encrypted, it can still expose AD structure if misconfigured and can be abused via certificate-based attacks like AD CS exploitation.                            |
We can run a service version scan with these specific ports to identify the DC

`nmap -p 88,135,139,389,445,464 -sC -sV -iL hosts.txt`


 `-sV`: This enables version detection. Nmap will try to determine the version of the services running on the open ports.
 `-sC`: Runs Nmap Scripting Engine (NSE) scripts in the default category.
 `-iL`: This tells Nmap to read the list of target hosts from the file `hosts.txt`. Each line in this file should contain a single IP address or hostname.

Output - 

~~~~~~~~~~~
Nmap scan report for 10.211.11.10
Host is up (0.16s latency).

PORT    STATE SERVICE      VERSION
88/tcp  open  kerberos-sec Microsoft Windows Kerberos (server time: 2025-05-22 15:49:57Z)
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp open  ldap         Microsoft Windows Active Directory LDAP (Domain: tryhackme.loc0., Site: Default-First-Site-Name)
445/tcp open  microsoft-ds Windows Server 2019 Datacenter 17763 microsoft-ds (workgroup: TRYHACKME)
464/tcp open  kpasswd5?
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb2-time: 
|   date: 2025-05-22T15:50:10
|_  start_date: N/A
|_clock-skew: mean: 2s, deviation: 5s, median: -1s
| smb-os-discovery: 
	|   OS: Windows Server 2019 Datacenter 17763 (Windows Server 2019 Datacenter 6.3)
|   Computer name: DC
|   NetBIOS computer name: DC\x00
|   Domain name: tryhackme.loc
|   Forest name: tryhackme.loc
|   FQDN: DC.tryhackme.loc
|_  System time: 2025-05-22T15:50:15+00:00
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required

Nmap scan report for 10.211.11.20
Host is up (0.16s latency).

PORT    STATE  SERVICE       VERSION
88/tcp  closed kerberos-sec
135/tcp open   msrpc         Microsoft Windows RPC
139/tcp open   netbios-ssn   Microsoft Windows netbios-ssn
389/tcp closed ldap
445/tcp open   microsoft-ds?
464/tcp closed kpasswd5
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2025-05-22T15:50:09
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
|_clock-skew: -1s

Post-scan script results:
| clock-skew: 
|   2s: 
|     10.211.11.10
|_    10.211.11.20
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 2 IP addresses (2 hosts up) scanned in 35.73 seconds

~~~~~~~~~~~~~~~

We can use a more exhaustive scan in completely unfamiliar environments to ensure we haven't missed any critical services running on uncommon ports

`nmap -sS -p- -T3 -iL hosts.txt -oN full_port_scan.txt`

#### Findings from the nmap scan - 

Domain Controller - `10.211.11.10`
Domain Name - `tryhackme.loc`

## Network Enumeration with SMB

- Enumerating network shares using the Server Message Block(SMB) protocol.
- Use tools like `smbclient` and `smbmap` to access the contents of the shares.

### Listing SMB Shares


- Without any credentials, we can try to connect anonymously. This is called as a Null Session because it has no username and password.

#### Tool 1: `smbclient`

`smbclient -L //DC-IP -N`
Ex: `smbclient -L //10.211.11.10 -N`

~~~~~~~~
Anonymous login successful

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	AnonShare       Disk      
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share 
	SharedFiles     Disk      
	SYSVOL          Disk      Logon server share 
	UserBackups     Disk      
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.211.11.10 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available


~~~~~~~~~~~

#### Tool 2: `smbmap`

`smbmap -H TARGET_IP`
Ex: `smbmap -H 10.211.11.10`


#### Tool 3: `nmap`

The following command explores which shares give READ/WRITE, READ, or no access.

`nmap -p445 --script smb-enum-shares 10.211.11.10`


### Accessing SMB Shares

To connect to a share using `smbclient` - 

`smbclient //TARGET_IP/SHARE_NAME -N`

Ex: `smbclient //10.211.11.10/SharedFiles -N`

To access share with a username and password - 

`--user=USERNAME --password=PASSWORD` or ` -U 'username%password' `

~~~~~~~~~~
Note that for domain accounts, you need to specify the domain using `-W`
~~~~~~~~~~~


### Other Tools

`impacket-smbclient` - a python implementation of impacket

`CrackMapExec` - not only for post-exploitation but also for enumeration. It includes many SMB modules for listing shares, testing credentials, and many others.

`enum4linux/enum4linux-ng`: 
	
	enum4linux -a TARGET_IP



