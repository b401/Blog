---
title: RC4_HMAC
author: i4
date: M05-27-2023
---

The RC4-HMAC Kerberos encryption type used by Microsoft Windows should have been disabled by default a long time ago.

Nowadays most EDR alert on TGT /TGS requests using RC4 (It's an indicator for Kerberoasting) but there is also another a little more lesser known security benefit that an environment can
gain by disabling the RC4 cipher.

## Overpass-The-Hash

If an actor gains access to a users NTHash he can either pass it (PtH - Pass The Hash) or use it to request a Kerberos TGT (OtH - Overpass The Hash).

One might ask why an attacker should do this and the explanation is quite simple - stealth and persistence.

By gaining a TGT the attacker has persistence until either - the system reboots, or the TGT expires (Usually 10 hours).
There is no other way to revoke an already issued TGT. Even worse since the TGT acts as a sign of an already authenticated user, the actor can request additional TGS. Even if the account
gets disabled or credentials rotated.

**Impacket**

```
    getTGT.py -hashes :[nthash] heimat.erde/i4
```

**Rubeus**

```
    Rubeus.exe asktgt /user:ir /rc4:[NTHash]
```

**Mimikat**

```
mimikatz # sekurlsa::pth /user:i4 /domain:heimat.erde /ntlm:[nthash}
```

### Why RC4 matters

Kerberos has three different ciphers used to authenticate the user that are enabled by default (DES is disabled).

- RC4\_HMAC\_MD5
- AES128\_HMAC\_SHA1
- AES256\_HMAC\_SHA1

Those ciphers also handle the initial Kerberos Pre-Authentication step. 

Now, dumping the NTHash is quite easy with mimikatz or lsassy but the attacker does not have access to the cleartext credential (except if WDIGEST was enabled or the credentials were saved in the vault).

The thing with RC4\_HMAC\_MD5 is, there is no need to.

While AES\_HMAC\_SHA1 requires the cleartext credentials, RC4\_HMAC\_MD5 does not and uses the NTHash for Kerberos. You don't need the cleartext credentials or any other information. 

This essentially means that if RC4 is disabled the attacker would need to:
- Steal a token from another process to impersonate another user (Can be noisy)
- Crack the NTHash (Depending on the complexity can take a long time)
- Not use OtH but instead PtH


### Detecting OtH

Look for the following log entries appearing at the same time:

**Windows**
```
EventID: 4624
- LogonTyp: 8
- Authentication Package: Negotiate
- Logon Process: seclogo
```
| Source Host | Destination Host | Domain Controller |
|--|--|--|
| 4648 - A logon was attempted using explicit credentials. | 4624 – An account was successfully logged on. (Logon Type = 3, Logon Process = Kerberos, Authentication Package = Kerberos) | 4768 – A Kerberos authentication ticket (TGT) was requested. (Encryption Type for RC4/AES128/AES256) |
| 4624 – An account was successfully logged on. (Logon type = 9 Logon Process = Seclogo) | 4672 – Special privileges assigned to new logon.	 | 4769 – A Kerberos service ticket was requested. (Encryption Type for RC4/AES128/AES256) |
| 4672 – Special privileges assigned to new logon. (Logged on user, not impersonated user)	|||

### Disabling RC4

_Computer Configuration/Policies/Windows Settings/Security Settings/Local Policies/Security Options_
```
Network security: Configure envryption types allowed for Kerberos

- DES_CBC_CRC:              Disabled
- DES_CBC_MD5:              Disabled
- RC4_HMAC_MD5:             Disabled
- AES128_HMAC_SHA1:         Enabled
- AES256_HMAC_SHA1:         Enabled
- Future encryption types:  Enabled
```

The GPO needs to be linked with the Domain Controllers.

Look for `0x17` as the encryption level for TGTs to see if RC4 gets used in the environment before disabling it.
