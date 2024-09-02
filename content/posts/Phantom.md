---
date: 2024-08-31
tags:
  - vulnlab
title: "Vulnlab Walkthrough #1 - Phantom"
---

# High-level Overview 📜

Phantom is a medium rated Windows machine on Vulnlab. To gain access to the administrator credentials I leveraged null SMB authentication, RID-Cycling, and Resource Based Constrained Delegation with a user that had a MachineAccountQuota of 0.

# Reconnaissance 📡

My default Nmap scan returned the following results.

Command:
```
 nmap -sVC -T4 -Pn 10.10.112.137 -oN phantom.txt

-sVC: Returns version of serivces and runs default NSE script for additional service enmeration such as smb2-security-mode

-T4: Allows for faster scannning. Default is T3, max is T5

-Pn: Disables host discovery, doesn't require machine to respond to a ping in order to find services.

-oN: Normal output to text file.

```

Response:
```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-08-31 13:46 EDT
Nmap scan report for 10.10.112.137
Host is up (0.14s latency).
Not shown: 987 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-08-31 17:46:39Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: phantom.vl0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: phantom.vl0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
3389/tcp open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2024-08-31T17:47:26+00:00; +2s from scanner time.
| rdp-ntlm-info: 
|   Target_Name: PHANTOM
|   NetBIOS_Domain_Name: PHANTOM
|   NetBIOS_Computer_Name: DC
|   DNS_Domain_Name: phantom.vl
|   DNS_Computer_Name: DC.phantom.vl
|   Product_Version: 10.0.20348
|_  System_Time: 2024-08-31T17:46:46+00:00
| ssl-cert: Subject: commonName=DC.phantom.vl
| Not valid before: 2024-07-05T19:49:21
|_Not valid after:  2025-01-04T19:49:21
5357/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Service Unavailable
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2024-08-31T17:46:49
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: mean: 2s, deviation: 0s, median: 1s
```

Due to the services listed, there seem to be few initial attack vectors. Here's my current thought process.

- Port 53: Me and my homies hate DNS exploitation, I believe the only attack I know is a zone transfer, which trying this early, would be a waste of time and possibly a rabbit hole.

- Port 88: Kerberos is used for Authentication, not necessarily something I change around and exploit without access to security policies. However, what we can keep in mind are possible misconfigurations such as AESPRoasting and RID-Cycling. 

![RID-Cycling](/RID.png)


RID-Cycling seems to have worked and provided, a userlist, we can use this userlist to try and AESPRoast as well as try to brute force passwords. 

![AESPRoast](/ASREP.png)

Unfortunately, AESPRoasting doesn't work due to UF_DONT_REQUIRE_PREAUTH not being enabled on user accounts, which would allow for me to grab the a Kerberos TGT  for the account without requiring a password, which in turn would allow for me to crack the hash and obtain a password. 


- Port 135/593: MSRPC is usually a dead end unless we have creds. Querying the endpoint shows a print system protocol is being used, potentially making it vulnerable to print nightmare. However further enumeration reveals nothing.

![MSRPC](/MSRPC.png)

- Port 139/445: Me and my homies love SMB. It's usually the most exploited service to gain initial access due to security misconfigurations and can usually be used in conjunction with other services for forced authentication

![SMBshares](/SMB.png)

As talked about, the file server allows us to authenticate with null credentials using the username Anonymous. This reveals a few network shares we can access access, most deny directory listings due to insufficient permissions; however the "public" network share holds a file called "tech_support_email.eml" which I downloaded to my local machine. Lets view its contents


![Email](/Email.png)

This seems to be an onboarding message that provides initial creds, spraying that against the userlist we generated provides a hit for ibryant. Having these creds, lets see if we can access some of the shares previously inaccessible to us, more specifically, the "Departments Share."

![creds](/creds.png)


![smb2](/SMB2.png)

ibryant does indeed have access to the "Departments share" which includes the IT, HR, and Finance departments, what interests us the most is the "IT_BACKUP_201123.hc" file under Departments share/IT. 

After research, it looks like an .hc file should be opened with veracrypt, a file encryption/decryption service, this would require a password to decrypt the given file. In a easier scenario, the .hc file  would be able to be cracked/decrypted using hashcat mode 13721; however, per the hint on vulnlab wiki, the hash for this file cannot be found on a regular wordlist like rockyou.txt and would take a custom wordlist.

To be completely honest, I did not make a custom wordlist, I guessed/bruteforced the password once learning the combination should be name + year + special character, which is honestly probably the case, or something similar, in most companies.

![drive1](/drive1.png)
![drive2](/drive2.png)
![foothold](/foothold.png)

Within the encrypted drive, theres a VyOS backup archive, meant for a virtual router, which hosts its configuration file.  Within the config, there's a VPN profile meant for user lstanley, which provides a username and password. We can do what we did previously, by using our user list to spray creds against the given password, doing that gives us access to the svc_sspr user. This technically should be our foothold, we'll save these creds for later, lets continue with enumeration of other ports.

![spray](/spray.png)

- Port 389/3268: There's not much LDAP would provide that we haven't already gained from rid-brute. a tool like windapsearch is mostly useful for enumerating users, groups, computers, or privileged users which we already have or will get from our next phase of weaponization. 

- Port 3389: We have creds we can test with RDP for svc_sspr, let see if the user is allowed to remote into the machine

![WinRM](/svcrm.png)

Indeed we can, and this gets us our user flag. Considering we can remote into the machine, we can also run bloodhound to enumerate the environment further. Considering we enumerated to gain credentials, which we are using to find an attack path, this constitutes our second phase, weaponization.

# Weaponization ⚔️

Lets run bloodhound on the environment and see if we can find an attack path.

![WinRM](/collection.png)

![Bloodhound](/bloodhound1.png)
![Bloodhound2](/bloodhound2.png)

According to bloodhound, svc_sspr has first degree object control over three users which allows the service account to change their password. All users seem to be apart of the ICT Security group, lets go check what special permissions members of this group may have.


![Bloodhound3](/bloodhound3.png)


![Bloodhound4](/bloodhound4.png)

Members of the ICT Security group are allowed to modify the permissions of msDS-AllowedToActOnBehalfOfOtherIdentity on a machine account. Modifying this permission allows a machine account or user, to act on behalf of another machine account. Ultimately, we could request kerberos tickets for the machine account, impersonating the Administrator and gaining their credentials. The act of this is called resource based constrained delegation, or RBCD.

Given the help provided from bloodhound and some research, lets see if we can escalate privileges by changing the password of a member in the ICT Security group, use one of their accounts to create another machine account which we can use during the RBCD attack

# Delivery 📬

Lets start with changing the password of `crose` and checking the MachineAccountQuota (MAQ) attribute of the account, which allows shows how many machine accounts a user can create and by default should be set to ten.

![MAQ](/changepsswd.png)

We have successfully changed the password but since msDS-MachineAccountQuota is set to 0, we are unable to preform RBCD in the way we normally would. After googling "rbcd with maq 0" (i am the google czar), I was lead to this article from [The Hacker Recipes](https://www.thehacker.recipes/ad/movement/kerberos/delegations/rbcd) which does a great job of breaking down how both RBCD and RBCD without being able to create machine accounts work. 

Here's a short breakdown.
1. Change msDS-AllowedToActOnBehalfOfOtherIdentity property on `crose`

![RBCD1](/rbcd1.png)

2. Request the user's TGT

![RBCD2](/rbcd2.png)
![RBCD3](/rbcd3.png)

3. Change the users hash to the session key
	1. This allows the user to request tickets without the KDC denying the request

![RBCD4](/rbcd4.png)

4. Abuse S4U2Self
	1. Service 4 User 2 Self allows `crose` to request a service ticket on behalf of the Administrator due to its msDS-AllowedToActOnBehalfOfOtherIdentity permissions over the machine and the fact that the KDC now allows `crose` to request tickets.
5. Abuse U2U
	1. U2U will now allow  `crose` which is impersonating `administrator` to request the admins encrypted TGT, which we can read, since we are indeed impersonating admin.

6. Get ccache and dump secrets

![RBCD5](/rbcd5.png)

7. Profit \$\$\$
![root](/root.png)
