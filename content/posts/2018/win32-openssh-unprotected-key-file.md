---
title: "Win32 OpenSSH Unprotected Key File"
date: 2018-01-21T23:51:22+10:00
draft: true
tags: [Win32-OpenSSH, "Windows Security", "SSH", Redis]
categories: [Software, Development]
---

I've been using the  [Win32-OpenSSH](https://github.com/PowerShell/Win32-OpenSSH)
project a lot recently. It's really great to have a native Microsoft supported
implementation in Windows.

However! Unlike, the MinGW or cygwin implementations of a Windows SSH client,
the Win32-OpenSSH project actually validates the security of private keys before
you are allowed to use them!

<!--more--> 

```batch
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions for 'C:/.../.vagrant/machines/default/hyperv/private_key' are too open.
It is required that your private key files are NOT accessible by others.
This private key will be ignored.
```

As is the error output, this can be particularly frustrating when using Vagrant,
which itself automatically provisions an insecure keypair for easy connection
between host and VM.

Near as I can tell, there seems to be no way to force the SSH client to ignore
the insecure key file and to be honest, that's probably a good thing, even if it
is annoying!

The fix for this solution on any Unix like system is pretty easy: grant `user`
read and write permissions, and revoke all permissions from `group` and `other`.

```bash
chmod u+rw,go-rwx
```

In Windows, however, things are more difficult. Permissions in Windows rarely
seem to matter--I can't quite decide if that is a good thing. Users never see
permissions and Windows definitely has a comprehensive implementation of ACLs.
Yet, those permissions are so complex few people know how to modify such permissions
on the command line!

Using guidance from the [Win32-OpenSSH wiki page](https://github.com/PowerShell/Win32-OpenSSH/wiki/Security-protection-of-various-files-in-Win32-OpenSSH)
these are the commands that work for me:


```powershell
# From an elevated PowerShell prompt
# NOTE: Be careful! 
$path = "C:/.../.vagrant/machines/default/hyperv/private_key"
$user_name = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name

# First ensure the owner is us
icacls $path /setowner $user_name

# Now remove any explicitly set ACLs by resetting permissions to that of parent
icacls $path /reset

# Next, disable permission inheritance and remove all inherited ACLs.
# Also, grant SYSTEM and our user full access.
icacls $path /inheritance:r /grant SYSTEM:F /grant "$user_name`:F"
        
```

Please Note, there's an incredibly complex, but fully featured, set of PowerShell
functions in the Win32-OpenSSH install directory
(`C:\Program Files\OpenSSH-Win64\OpenSSHUtils.psm1` on my machine) that can
automate these permission fixes for you. I however, preferred to work out the
commands myself.