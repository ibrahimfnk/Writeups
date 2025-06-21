
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
	Ex: `nmap -sn 10.211.11.0/24`


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





## Domain Enumeration

Having a better understanding of our target's network, it's time to enumerate users to help us identify **valid accounts** and **potential targets**.

**Usernames** can be gathered through various **unauthenticated methods**, each relying on different misconfigurations.

### Technique 1: `LDAP Enumeration` (Anonymous Bind)

- Lightweight Directory Access Protocol (LDAP) is a widely used protocol for accessing and managing directory services, such as Microsoft Active Directory.

- LDAP helps locate and organize resources within a network, including users, groups, devices, and organizational information, by providing a central directory that applications and users can query.

- Some LDAP servers allow anonymous users to perform read-only queries. This can expose user accounts and other directory information.

#### Tool: `ldapsearch`

We can check if ldap anonymous bind is enabled with:

`ldapsearch -x -H ldap://10.211.11.10 -s base`

 `-x`: Simple authentication, in our case, anonymous authentication.
 `-H`: Specifies the LDAP server.
 `-s`: Limits the query only to the base object and does not search subtrees or children.

If it is enabled, we should see lots of data, similar to the output below:

~~~
dn:
domainFunctionality: 6
forestFunctionality: 6
domainControllerFunctionality: 7
rootDomainNamingContext: DC=tryhackme,DC=loc
ldapServiceName: tryhackme.loc:dc$@TRYHACKME.LOC
isGlobalCatalogReady: TRUE
supportedSASLMechanisms: GSSAPI
supportedSASLMechanisms: GSS-SPNEGO
supportedSASLMechanisms: EXTERNAL
supportedSASLMechanisms: DIGEST-MD5
~~~~

We can then query user information with this command:

`ldapsearch -x -H ldap://10.211.11.10 -b "dc=tryhackme,dc=loc" "(objectClass=person)"`

Output(Example) - 

```
# katie.thomas, Consulting, People, tryhackme.loc
dn: CN=katie.thomas,OU=Consulting,OU=People,DC=tryhackme,DC=loc
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: katie.thomas
sn: Thomas
title: Associate
givenName: Katie
distinguishedName: CN=katie.thomas,OU=Consulting,OU=People,DC=tryhackme,DC=loc
instanceType: 4
whenCreated: 20250430150644.0Z
whenChanged: 20250523142551.0Z
displayName: Katie Thomas
uSNCreated: 21146
memberOf: CN=Internet Access,OU=Groups,DC=tryhackme,DC=loc
uSNChanged: 123832
department: Consulting
name: katie.thomas
objectGUID:: 1I2VoFIma0qxmIAyfv0uhQ==
userAccountControl: 66050
badPwdCount: 3
codePage: 0
countryCode: 0
badPasswordTime: 133925673655651362
lastLogoff: 0
lastLogon: 0
```

### Technique 2: `Enum4linux-ng`

**enum4linux-ng** is a tool that automates various enumeration techniques against Windows systems, including user enumeration. It utilizes SMB and RPC protocols to gather information such as user lists, group memberships, and share details.

We can run the following command to get as much information as possible from the DC:

`enum4linux-ng -A 10.211.11.10 -oA results.txt`

 `-A`: Performs all available enumeration functions (users, groups, shares, password policy, RID cycling, OS information and NetBIOS information).
 `-oA`: Writes output to YAML and JSON files.

Output (Example) - 

~~~
 =====================================
|    Users via RPC on 10.211.11.10    |
 =====================================
[*] Enumerating users via 'querydispinfo'
[+] Found 32 user(s) via 'querydispinfo'
[*] Enumerating users via 'enumdomusers'
[+] Found 32 user(s) via 'enumdomusers'
[+] After merging user results we have 32 user(s) total:
'1609':
  username: sshd
  name: sshd
  acb: '0x00000210'
  description: (null)
'1616':
  username: gerald.burgess
  name: Gerald Burgess
  acb: '0x00000010'
  description: (null)
'1617':
  username: nigel.parsons
  name: Nigel Parsons
  acb: '0x00000010'
  description: (null)
'1618':
  username: guy.smith
  name: Guy Smith
  acb: '0x00000010'
  description: (null)

~~~~

This tools gives a very detailed and systematic report on the users, groups, policies, etc.

### Technique 3:  `RPC Enumeration`

- Microsoft Remote Procedure Call (MSRPC) is a protocol that enables a program running on one computer to request services from a program on another computer, without needing to understand the underlying details of the network.

- RPC services can be accessed over the SMB protocol.

- When SMB is configured to allow null sessions that do not require authentication, an unauthenticated user can connect to the `IPC$ share` and enumerate users, groups, shares, and other sensitive information from the system or domain.


#### Tool: `rpcclient`

`rpcclient -U "" 10.211.11.10 -N`

 `-U`: Used to specify the username, in our case, we are using an empty string for anonymous login.
 `-N`: Tells RPC not to prompt us for a password.

In the rpc session we can list the users using - `enumdomusers`

~~~
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[sshd] rid:[0x649]
user:[gerald.burgess] rid:[0x650]
user:[nigel.parsons] rid:[0x651]
user:[guy.smith] rid:[0x652]
user:[jeremy.booth] rid:[0x653]
user:[barbara.jones] rid:[0x654]
user:[marion.kay] rid:[0x655]
user:[kathryn.williams] rid:[0x656]
user:[danny.baker] rid:[0x657]
user:[gary.clarke] rid:[0x658]
user:[daniel.turner] rid:[0x659]
user:[debra.yates] rid:[0x65a]
user:[jeffrey.thompson] rid:[0x65b]
user:[martin.riley] rid:[0x65c]
user:[danielle.lee] rid:[0x65d]
user:[douglas.roberts] rid:[0x65e]
user:[dawn.bolton] rid:[0x65f]
user:[danielle.ali] rid:[0x660]
user:[michelle.palmer] rid:[0x661]
user:[katie.thomas] rid:[0x662]
user:[jennifer.harding] rid:[0x663]
user:[strategos] rid:[0x664]
user:[empanadal0v3r] rid:[0x665]
user:[drgonz0] rid:[0x666]
user:[strate905] rid:[0x667]
user:[krbtgtsvc] rid:[0x668]
user:[asrepuser1] rid:[0x669]
user:[rduke] rid:[0xa31]
user:[user] rid:[0x1201]
~~~

### Technique 4: `RID Cycling`

- In Active Directory, RID (Relative Identifier) ranges are used to assign unique identifiers to user and group objects. These RIDs are components of the Security Identifier (SID), which uniquely identifies each object within a domain. 
- Certain RIDs are well-known and standardized.


| RID Number/Range | Account/User          |
| ---------------- | --------------------- |
| 500              | Administrator Account |
| 501              | Guest                 |
| 512              | Domain Admins         |
| 513              | Domain users          |
| 514              | Domain guests         |
| 1000+            | User accounts         |
If `enumdomusers` is restricted, we can manually try querying each individual user RID with this bash command:

`for i in $(seq 500 2000); do echo "queryuser $i" |rpcclient -U "" -N 10.211.11.10 2>/dev/null | grep -i "User Name"; done`

~~~
Note: This command can take 2-3 minutes to complete.
~~~


We can use **enum4linux-ng** to determine the RID range, or we can start with a known range, for example, 1000-1200, and increment if we get results.


### Technique 5: `Username Enumeration With Kerbrute`

- **Kerberos** is the primary authentication protocol for Microsoft Windows domains.

- Unlike **NTLM**, which relies on a challenge-response mechanism, **Kerberos** uses a ticket-based system managed by a trusted third party, the **Key Distribution Centre (KDC)**.

~~~
**Kerbrute** is a popular enumeration tool used to brute-force and enumerate valid Active Directory users by abusing the Kerberos **pre-authentication**.
~~~~

Tools like **enum4linux-ng** or **rpcclient** may return _some_ usernames, but they could be:

- Disabled accounts
- Non-domain accounts
- Fake honeypot users
- Or even false positives

Running those through **kerbrute** lets us confirm which ones are real, active AD users, which allows us to target them more accurately with password sprays.

`~/Downloads/kerbrute userenum --dc 10.211.11.10 -d tryhackme.loc usernames.txt `

~~~
cropped output - 

2025/05/26 21:34:33 >  [+] VALID USERNAME:	 martin.riley@tryhackme.loc
2025/05/26 21:34:33 >  [+] VALID USERNAME:	 danielle.ali@tryhackme.loc
2025/05/26 21:34:33 >  [+] VALID USERNAME:	 jennifer.harding@tryhackme.loc
2025/05/26 21:34:33 >  [+] VALID USERNAME:	 strategos@tryhackme.loc
2025/05/26 21:34:33 >  [+] VALID USERNAME:	 krbtgtsvc@tryhackme.loc
2025/05/26 21:34:33 >  [+] VALID USERNAME:	 empanadal0v3r@tryhackme.loc
2025/05/26 21:34:33 >  [+] VALID USERNAME:	 drgonz0@tryhackme.loc
2025/05/26 21:34:33 >  [+] VALID USERNAME:	 strate905@tryhackme.loc
2025/05/26 21:34:33 >  [+] VALID USERNAME:	 asrepuser1@tryhackme.loc
2025/05/26 21:34:34 >  [+] VALID USERNAME:	 user@tryhackme.loc
2025/05/26 21:34:34 >  [+] VALID USERNAME:	 rduke@tryhackme.loc
2025/05/26 21:34:34 >  Done! Tested 32 usernames (27 valid) in 0.642 seconds

~~~

