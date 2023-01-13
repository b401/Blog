---
title: DPAPI Kerberos delegation
author: i4
date: M01-13-2023
---

> If you're talking too much about kerberos you will forget how it's working

Kerberos delegation allows systems and services to impersonate other accounts on behalf of those requesting access.

This is being done with so called SPNs => Service Principal Names. 
A service principal name is written in the following format:

- [Name-Formats](https://learn.microsoft.com/en-us/windows/win32/ad/name-formats-for-unique-spns)
    - `<service class>/<host>:<port: optional>/<service name: optional>`

*Example*: `WSMAN/ELSTR.heimat.erde`

## [Security.Cryptography.ProtectedData]::Protect()

After the removal of the unconstrained delegation on the AD object, a service no longer started and presented us a small trace.

```
Error An error occurred: 'CryptographicException: The requested operation cannot be completed. 
The computer must be trusted for delegation and the current user account must be configured to allow delegation.
'. Error details: at System.Security.Cryptography.ProtectedData.Protect(Byte[] userData, Byte[] optionalEntropy,
 DataProtectionScope scope)
```

This application was invoked through a remote powershell session. Through the trace we could isolate the issue and build a small powershell script for further testing.

```
Add-Type -AssemblyName System.Security
[byte[]] $byte = 6e,65,72,64
[Security.Cryptography.ProtectedData]::Protect($byte,$NULL,0)
```

- `[Security.Cryptography.ProtectedData]::Protect()` uses DPAPI to encrypt/decrypt.

Now we were able to simulate the failing part of the application without all the other code around.  
Using `runas /user:domain\service powershell.exe` and `Invoke-Command` gave us the same error again so we were trying to figure out where exactly the delegation is missing.



### DPAPI and why it needs delegation

After a lot of trial and error I realized how DPAPI works. (thanks mimikatz)

> dpapi::masterkey describes a Masterkey file and unprotects each Masterkey (key depending). In other words, it can decrypt and request masterkeys from active directory. It has the following command line arguments:
> ...
> /rpc: it can be used to remotely decrypt the masterkey of the target user by contacting the domain controller. According to Benjamin, in a domain, a domain controller runs an RPC Service to deal with encrypted masterkeys for users, 

By searching further we can also find [DPAPI-Masterkey-Backup-Failures](https://learn.microsoft.com/en-us/troubleshoot/windows-server/identity/dpapi-masterkey-backup-failures#resolution).

#### What service class to delegate

- `protectedstorage/RWDC`

Be aware that this allows the AD object where you set up this constrained delegation to impersonate all users and allow it to grab their DPAPI master key.  
At least it's better than allowing the master key to be backed up locally through the following key:
-  `HKEY_LOCAL_MACHINE\Software\Microsoft\Cryptography\Protect\Providers\df9d8cd0-1501-11d1-8c7a-00c04fc297eb` (DO NOT DO THIS!)

## Scenario where this is useful knowledge

- RWDC (Read/Write Domain Controller)
- ServerA 
    - Service gets started here
- ServerB
    - Service connects through PSRemote
- Service


```
                                  RWDC
                                 +===================+
                                 |                   |
                                 | DPAPI MasterKey   |
                                 |                   |
                                 |                   |
                                 +==========+========+
                                            |
                                            |
                                            |
                                            |
                                            |
                                            |
 ServerA                           ServerB  v
+==================+              +=================+
| Service          |  PSRemote    | Service         |
|                  +==============+                 |
|                  |              | DPAPI function  |
|                  |              |                 |
+==================+              +=================+
```

## Where to set the constrained delegation

|\| AD Object \| | Constrained Delegation \| | SPN \| |
|--|--|--|
| ServerA   | n/a | n/a|
| ServerB   | `protectedstorage\RWDC` | n/a |
| RWDC | n/a | n/a |

In case of a resource-based constrained delegation you'd have to run the following:

```
Set-ADComputer RWDC -PrincipalsAllowedToDelegateToAccount ServerB
```



