---
title: Kerberos
author: i4
date: M10-14-2023
---

![Kerberos](/b/images/kerberos.png)

Kerberos still seems to be a mythical protocol for many sysadmins, and many of them continue to use NTLM (Net-NTLMv2) in a client/server environment.

Some important facts to consider before we start to dive in:

- Kerberos **REQUIRES** a working DNS environment.
- Kerberos does not work if you use IP addresses to communicate with a system (e.g., \\\\10.0.0.10\ or http[:]//10.0.0.10/)."
- The client needs to have visibility to the KDC to request a TGT. 
- If a CNAME is added for a server, the same CNAME must also be added as an SPN to the service.
- Always use the FQDN of the service. While most services include the SPN for the hostname alone it's not certain.
- If there is no SPN or the connection occurs over IP it **will** fall back to NTLM.
- Ensure that sensitive accounts are added to the protected users group and/or that the 'This account is sensitive and cannot be delegated' flag is set."
- Ensure that there are no unconstrained delegations in your network.
    - Use wireshark/tcpdump to figure out what services are actually required for delegation.
- Reconsider if you really need a second hop in an architecture design.
- Disable weak Kerberos ciphers (e.g., DES/RC4)
- Be aware that target services do not verify with the KDC if the TGS ticket is valid. They trust the tickets they receive.
- If a TGT gets stolen (dumped or forged) there is no way to invalidate a TGT!
    - A TGT means that the user is authenticated and can request additional TGS tickets. Disabling or changing the password won't change this.
    - The TGT remains valid until it reaches its ticket expiration (usually 10 hours).
    - Make sure to reboot the compromised host or run 'klist purge -li 0x3e7' to invalidate all tickets on the host.

# Kerberos

I won't go in the historical facts about Kerberos but just you should now that it was created a long long time ago as an open-source standard. (Even X11 had an implementation at some point)  

Kerberos is supported on many different operation systems, including IBM2, GNU/Linux, OS2, Windows, Solaris and others.

Kerberos itself is an AAA (Authentication, Authorization, Audit) protocol that uses a trusted third-party server (KDC) to grant access to resources (TGS).

To avoid confusion with the different synonyms, here's a list that should clarify their meanings:

|Synonym | Full Name | Description |
|:--|:--|:--|
| AS | Authentication Server | Receives authentication requests and issues a TGT for the TGS. |  
| TGS | Ticket Granting Service | The service responsible for encrypting the ticket with a specific hash, typically the hash of the "CN=krbtgt,CN=Users,DC=xxx,DC=xxx" object in the case of Active Directory.|
| KDC | Key Distribution Center | The KDC can be a combined entity that includes both AS and TGS functions (Domain Controllers). In some cases, they can be separated into different systems, but typically, a single KDC handles both functions. 
| TGT | Ticket Granting Ticket | A ticket that allows access to various entities after obtaining it from the Authentication Server (AS)|
| ADDC | Active Directory Domain Controller | Domain Controller :) |
| SPN | Service Principal Name |A service identifier in the form of "service/hostname:port" that clients use to request services. In the case of service accounts or machines, this value needs to be set correctly. [List with valid SPNs](https://adsecurity.org/?page_id=183)|
| PAC | Privilege Attribute Certificate | A PAC (Privilege Attribute Certificate) contains encoded authorization information, including structures like "DOMAIN_GROUP_MEMBERSHIP," "GROUP_MEMBERSHIP," and others, which define a user's privileges and group memberships.|


Communication flow described by Microsoft:

1. Request TGT (this is normally done at initial login) - Also known as (__AS-REQ__)
2. KDC verifies the client and answers with an encrypted TGT (__AS-REP__). This step involves the AS responding to the client's initial request by sending back an encrypted TGT.
3. The client sends the TGT to the TGS if it exists and can include the SPN of the service the client wants to access. (__TGS-REQ__)
4. The KDC verifies the TGT and checks if the object has permission to access the service. If everything checks out, it answers with a valid session key. (__TGS-REP__)
5. The client sends the server a valid session key to prove access has been granted.
6. Access will been granted/denied.


Communication flow a little easier explained:

0. Logon
1. AES[NTLM(User Password)](NTLM(User password) + Timestamp) gets sent to the KDC to request a TGT. ( The timestamp prevents replay attacks )
2. The KDC verifies the user's credentials and checks other factors like group memberships and account restrictions before creating a TGT.
3. TGT is encrypted and digitally signed by the KDC which gets sent back to the client (AS-REP).
4. When the user now wants to connect to a service, he requests a TGS for that specific service. The DC check the TGT and validates the PAC checksum.
5. The TGS is encrypted using the target service accounts NTHash hash (If there is an `SPN` record) and sent back to the client.
6. The user connects to the service and presents the TGS. Since the service knows its own password it can decrypt the TGS.


And a diagram because why not:


```
  +=========+                 +=========+                 +=========+
  |  Client |                 |   KDC   |                 |Webserver|
  +=========+                 +=========+                 +=========+
      |                            |                            |
      | Request TGT (AS=REQ)       |                            |
      |===========================>|                            |
      |                            |                            |
      |                            | Verify client and          |
      |                            | send encrypted TGT         |
      |                            |     (AS=REP)               |
      |<===========================|                            |
      |                            |                            |
      |                            |                            |
      | Send TGT to TGS            |                            |
      | Include SPN for Webserver  |                            |
      |     (TGS=REQ)              |                            |
      |===========================>|                            |
      |                            |                            |
      |                            | Verify TGT and             |
      |                            | permissions                |
      | Send valid session key     |                            |
      |         (TGS=REP)          |                            |
      |<===========================|                            |
      |                            |                            |
      |                            |                            |
      | Send session key to        |                            |
      | Webserver                  |                            |
      | Request access             |                            |
      |========================================================>|
      |                            |                            |
      |                            |                            | Validate session key
      |                            |                            | Grant/Deny access
      |<========================================================|
      |                            |                            |
      | Connect to Webserver       |                            |
      |========================================================>|

```



### Tickets
You can list your current assigned tickets with the command ``klist``.

#### TGT

You can distinguish between TGTs and STs by looking at the server UPN part. In this case we see that this ticket is valid for the krbtgt service which is essentially
the KDC service.
```
#0>     Client: i4 @ HEIMAT.ERDE
        Server: krbtgt/HEIMAT.ERDE @ HEIMAT.ERDE
        KerbTicket Encryption Type: AES-256-CTS-HMAC-SHA1-96
        Ticket Flags 0x40e10000 -> forwardable renewable initial pre_authent name_canonicalize 
        Start Time: 10/14/2023 18:55:21 (local)
        End Time:   10/15/2023 4:55:21 (local)
        Renew Time: 10/21/2023 18:55:21 (local)
        Session Key Type: AES-256-CTS-HMAC-SHA1-96
        Cache Flags: 0x1 -> PRIMARY
        Kdc Called: TATSUMAKI
```

- _Client:_ Username for which the ticket is valid for.
- _Server:_ The service for which the ticket is issued is `krbtgt/HEIMAT.ERDE` which is the KDC service
- _KerbTicket Encryption Type:_ Speciffies the encryption type used to protect the ticket.
- _Ticket Flags:_ Will be explained more in depth further down 
- _Start Time:_ The time the ticket becomes valid, in local time.
- _End Time:_ The time when the ticket expires and is no longer valid, in local time.
- _Renew Time:_ The time until the ticket can still be renewed, in local time.
- _Session Key Type:_ Specifies the encryption type of the session key within the ticket.
- _Cache Flags:_   Supports different extensions [S4U2self](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/02636893-7a1f-4357-af9a-b672e3e3de13) for example or if this ticket can be used for delegation.
- _Kdc Called:_ Indicates the KDC server that was contacted for obtaining this ticket.

#### TGS Ticket / Service Ticket (ST)

A ST is in essence a copy of the original TGT but contains different flags (and session key) which is valid for a service in the same realm. While cross-realm kerberos authentication is possible, the complexity would be too much for this post.

```
...
#3>     Client: EULR @ heimat.erde
        Server: LDAP/Falke.heimat.erde/heimat.erde @ heimat.erde
        KerbTicket Encryption Type: AES-256-CTS-HMAC-SHA1-96
        Ticket Flags 0x40a50000 -> forwardable renewable pre_authent ok_as_delegate name_canonicalize
        Start Time: 11/10/2023 13:39:19 (local)
        End Time:   11/10/2023 23:38:34 (local)
        Renew Time: 18/10/2022 13:38:34 (local)
        Session Key Type: AES-256-CTS-HMAC-SHA1-96
        Cache Flags: 0
        Kdc Called: Falke.heimat.erde
...
```

Let's say we want to purge _all_ current tickets and request new ones: `klist purge`.  
This removes _every_ ticket and requests new ones as soon as we try to access a service that uses kerberos authentication.  

=> After `klist purge`
```
Current LogonId is 0:0x91819

Cached Tickets: (0)
```

=> Run `dir \\falke.heimat.erde\SYSVOL`

```
...
#2>     Client: EULR @ heimat.erde
        Server: cifs/Falke.heimat.erde @ heimat.erde
        KerbTicket Encryption Type: AES-256-CTS-HMAC-SHA1-96
        Ticket Flags 0x40a50000 -> forwardable renewable pre_authent ok_as_delegate name_canonicalize
        Start Time: 11/10/2022 14:19:44 (local)
        End Time:   11/10/2022 0:19:44 (local)
        Renew Time: 18/05/2022 14:19:44 (local)
        Session Key Type: AES-256-CTS-HMAC-SHA1-96
        Cache Flags: 0
        Kdc Called: Falke.heimat.erde
...

```

As you see a TGS Ticket and additionally a TGT (not visible here) has been automatically acquired.

What's interesting to notice is that the ticket uses `CIFS` as protocol name and not `SMB`.  
The reason `net user` uses cifs is that the request goes over `RPC` (139/TCP).

#### Ticket Flags

Ticket flags determine the behavior in which the ticket can be used.  
In case of the example above:
```
    Ticket Flags 0x40a50000 -> forwardable renewable pre_authent ok_as_delegate name_canonicalize
```

Example and descriptions of valid values:

|Flag|Description|
|--|--|
|forwardable| Allows forwarding of the clients identity to (SSO) |
|forwarded| Indicates that the ticket has been forwarded to another service.|
|proxiable| Allows the ticket to be proxied or delegated to another serivce, giving that service the authority to act on behalf of the client. [RFC4120.2.6](https://datatracker.ietf.org/doc/html/rfc4120/#page-19) |
|pre_authent| Requires the client to authenticate before requesting a service ticket.|
| initial | Set for the TGT issued during the authentication process |
| renewable | Allows the renewable of that ticket without additional authentication |
| postdated | Indicates a ticket that is not yet valid but will become valid in the future. |
| invalid | Indicates that the ticket is no longer valid and should not be used for authentication | 
|ok\_as_delegate| Allows ticket delegation. This is crucial to disable on sensitive accounts (i.e. Domain Admins) - see Delegation|
|name_canonicalize| Indicates that the identity within the ticket should be canonicalized. (Converting a principal name to a standardized, canonical form.) |

Flags are documented [here](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/de260077-1955-447c-a120-af834afe45c2)

### Delegation
Delegation allows services to impersonate users to access/forward their requests on their behalf.  
A common use case for delegation is shown by SharePoint. In a SharePoint environment, various users interact with documents and resources stored on the platform. To ensure that users can access resources according to their permissions, SharePoint itself needs to act on behalf of those users when retrieving or modifying data.

![Delegation](/b/images/delegation.png)

Delegation is handled by the `TrustedForDelegation` flag or the `msDS-AllowedToDelegateTo` attribute.

```
...
SamAccountName             : CHARIOT$
ServicePrincipalName       : {WSMAN/Chariot, WSMAN/Chariot.heimat.erde, TERMSRV/CHARIOT, TERMSRV/Chariot.heimat.erde...}   
TrustedForDelegation       : True
TrustedToAuthForDelegation : False
...
```

#### Unconstrained delegation

Unconstrained delegations are a big issue in domain environments. They effectively allow services or computer accounts that have this flag to impersonate by requesting and using TGS tickets for any other service on behalf of any user.

This means that in case of a machine with unconstrained delegation getting compromised an adversary can use the collected tickets/hashes for Pass-The-Ticket/Pass-The-Hash attacks to any other resource or create a so called [Silver Ticket](https://book.hacktricks.xyz/windows/active-directory-methodology/silver-ticket#silver-ticket).

The reason this works is that with unconstrained delegation, the system stores the TGS tickets in the `lsass` process of the service/server for later use.

If a server with unconstrained delegation gets compromised the dumped `lsass` process will include all cached TGS.

### AS-REP Roasting

In an initial TGT request the client sends a timestamp in cleartext. If preauthentication is enabled the server will answer with ``KRB5KDC_ERR_PREAUTH_REQUIRED`` and requests a new connection from the client.  
On the second connection the client sends the timestamp encrypted with the credential to the server.  
After decrypting the timestamp with the user password (Which is stored in `Ntds.dit`) it knows that this request is valid and not a relay/replay (since the timestamp is included) or guessing attack.

Without preauthentication the KDC answers on the request for the specific user with an encrypted TGT using a portion of the users password hash. This in turn can be used to crack offline later.  

AS-REP roasting requires the attribute `DONT_REQUIRE_PREAUTH` on an account to be set (The `DoesNotRequirePreAuth` property). 

```
Name                  : i4
DoesNotRequirePreAuth : True
```

For this attack **no** domain account is needed, only a connection to an ADDC since the permissions is given to the `everyone` group.  
They only issue is finding a valid target without having the ability to query the domain controller.

More info about AS-REP Roasting: [harmj0y](http://www.harmj0y.net/blog/activedirectory/roasting-as-reps/)  
Checking for AS-REP Roasting vulnerable accounts: [GetNPUsers.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/GetNPUsers.py) 

```
# 
GetNPUsers.py heimat.erde/i4 -no-pass
```


### Kerberoasting
Kerberoasting is used to crack NTLM passwords of users that are used as service accounts (Accounts with a valid servicePrincipalName value).  
Found with the command: ``Get-ADUser -Filter * | Where {$_.servicePrincipalName -ne $null}``

When a TGS ticket gets requested, the session key gets hashed with a timestamp and the service accounts credentials (The account with the SPN) and sent back to the requester. It is not important if the user has access to the service yet since the TGS ticket gets created anyway.

Since the attacker knows 1/2 variables (the timestamp) he can just dump the TGS ticket from memory and bruteforce it locally until he finds the valid password hash.

Kerberoasting itself is a persistent issue by design and can only be mitigated by the following startegies:

- Disabling weak encryption algorithms like RC4 is a good practice.
- Monitoring for downgrade attacks is important to detect and prevent them.
- Regularly reviewing accounts with SPNs helps identify potential targets.
- Ensuring that service account passwords are long, complex, and rotated regularly can make it more challenging for attackers to crack the hashes.


### The KRBTGT user

Every windows active directory domain has a central user called krbtgt this user's credential (usually 128bit) provide the means to sign and hash Ticket Granting Tickets to make Kerberos work. 

The KRBTGT user is unique to each domain and should have the following properties:  

_Member of following groups only:_
- Domain Users
- Denied RODC Password Replication Group
_Status_: Disabled
_Location_: CN=krbtgt,CN=Users,DC=x,DC=x
_SID_: S-1-5-domain-502

RODC have their own unique krbtgt user (called krbtgt_xxxxxxx). This ensures isolation between trusted Domain Controllers and untrusted RODCs.

#### KRBTGT Password change
Microsoft recommends changing the password regularly to ensure a safe password hygiene. 
The KRBTGT password needs to be changed twice to ensure that there's no password history.   

Which password gets set doesn't matter as the DC will change the password to a 128bit long passphrase afterwards.  
Just make sure to wait after the first password reset until the password was replicated on all other DCs. (10h in case for maximum TGT lifetime)

Be aware that all RODCs have unique krbtgt accounts which need to be rotated separately. 

Microsoft has released a script that helps resetting the krbtgt password: https://github.com/microsoft/New-KrbtgtKeys.ps1

