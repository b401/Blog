---
title: PAM execution on logon
author: i4
date: M01-13-2023
---

## Why

During development of a project I wanted to be able to only allow a single logon through sFTP.  
Every subsequent logons should fail.
Although there are different ways to achieve the same functionality, I wanted PAM to initiate the sFTP credential deletion.

### Basic Idea

```
sFTP Logon with user + priv key
       v
[PAM] User & key is valid
       v
[PAM] Call script => remove public key
       v
[PAM] Logon successful
```

## What is PAM

GNU/Linux uses the Pluggable Authentication Modules (PAM) to handle authentication of different services.  
All PAM applicable modules are stored under the path `/etc/pam.d/`.

If we take a look at the `/etc/pam.d/sshd` module we see the following calls to object libraries:


- _Comments are removed_

```
@include common-auth
account    required     pam_nologin.so
@include common-account
session [success=ok ignore=ignore module_unknown=ignore default=bad]        pam_selinux.so close
session    required     pam_loginuid.so
session    optional     pam_keyinit.so force revoke
@include common-session
session    optional     pam_motd.so  motd=/run/motd.dynamic
session    optional     pam_motd.so noupdate
session    optional     pam_mail.so standard noenv # [1]
session    required     pam_limits.so
session    required     pam_env.so # [1]
session    required     pam_env.so user_readenv=1 envfile=/etc/default/locale
session [success=ok ignore=ignore module_unknown=ignore default=bad]        pam_selinux.so open
@include common-password
```

- Every "`session`" runs a different action on the new session logon.  
- `session` entries that have the `required` flag will hard fail and terminate the authentication.

## Adding your own PAM module 

- Needs sshd installed

Edit `/etc/pam.d/sshd` and append the following line right before `@include common-passwod`:

```
...
session required pam_exec.so seteuid /path/to/your/script.sh # << Add this line
@include common-password
```

- `seteuid` will instruct `pam_exec.so` to run the script with the permissions of the logon user and populates PAM user variables.
- Switch out `require` with `optional` you're testing.

## Accessing variables

Now on every ssh logon PAM will execute the script.  
Inside the script we can access individual variables to determine an action.


`pam_exec.so` populates the following variables:

- `PAM_RHOST` => IP of the remote host
- `PAM_RUSER` => Logon username
- `PAM_SERVICE` => Service that PAM used (sshd/sudo/etc.)
- `PAM_TTY` => path of the TTY
- `PAM_TYPE` => contains either `open_session` or `close_session` depending of the action.

This should now allow you to build a bash script to execute certain actions on specific service logons.

## Example script

```
#!/bin/bash

if  [ "$PAM_USER" == "user1" ] && [ "$PAM_TYPE" == "open_session" ]; then
    # overwrite a key file
    echo > /path/to/key
fi
```

# Hardening SFTP

As a small tidbit here's a configuration how you can harden the sFTP logon:

- `ForceCommand internal-sftp` forces ssh to load sftp
    - `-u 0666` sets the default umask
    - `-p realpath,open,write,close,lstat` blacklists the ftp commands

This allows the user1 to connect to sFTP and create files but not to download or read directory contents.

_/etc/ssh/sshd_config.d/sftp.conf_

```
Match User user1
    ForceCommand internal-sftp -u 0666 -p realpath,open,write,close,lstat
    AuthorizedKeysFile /path/to/keyfile
    PubKeyAuthentication yes
    PasswordAuthentication no
    ChrootDirectory /path/to/chroot/dir
    PermitTunnel no
    AllowAgentForwarding no
    AllowTcpForwarding no
    X11Forwarding no
```

### Chroot permission issues

The `ChrootDirectory` needs to be owned by root and not writable by anybody else:

```
$ ls /path/to/chroot/dir
drwxr-xr-x 3 root   root    4.0K    Dec 20  00:00 .
```

SSHD will create it's own user directory in there so there is no need to create additional directories.
