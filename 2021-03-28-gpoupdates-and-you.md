---
author: i4
title: "The secrets behind GPOs"
date: M03-28-2021
---

## Hello friends!
Just a quick post.  
I've worked on a case where the command `gpupdate /scope:computer` replaced (removed and rewrote) certain registry keys and initialy created wrong values.  

I took the opperturnity to dive into the shitfest that's called GPO management. Let me show you some the the findings I made during this journey!



## GPO and you
Every active directory installation comes with a pre-defined group policy which is already active for all authenticated users. => "Default Domain Policy"  


**Beware**: **Authenticated Users** includes **every** AD-Object. -> Yes. A computer is also an authenticated user.

I don't intend to go deep into the GPO on the AD-Controller side. Just know that you can create/alter and delete them over the group policy management interface (or powershell).

You can see all defined GPOs from every ad connected computer on the following CIFS share:
- \\\\%USERDOMAIN%\\sysvol

![one](/b/images/one.png)


If you dive a little deeper you'll find every single GPO that's has been configured.   
![two](/b/images/two.png)

Now comes the interessting part. How does the system know if it needs to update the local gpos if you execute an ``gpupdate`` command?

![three](/b/images/three.png)

Basically it's pretty simple. In every GPO folder (The {UUID} thing), you'll find a GPT.ini file with the following content:

```
[General]
Version={incremented number}
displayName=GPO Name
```

If the version matches the number you have on your local machine (`C:\Windows\System32\GroupPolicy\DataStore\0\sysvol\{domainname}\Policies\{UUID}`) `gpupdate` does nothing.  
As soon as an admin changes something on the GPO the number gets incremented and `gpupdate` loads the new version.

`gpupdate /force` on the other hand, ignores the _Version_ and just grabs the version on the ADC.

Also if the ACLs are misconfigured anyone could make changes to the GPOs.  
Like replace the harmless.exe with an malicious.exe and set the Version to +1

### TLDR;
**AD GPO Store:** `\\%USERDOMAIN\SYSVOL\Policies`  
**Local AD GPO Store:** `C:\Windows\System32\GroupPolicy\DataStore\0\sysvol\{domainname}\Policies\{UUID}`

Don't use `gpupdate /force` as it ignores the Version and just resets everything again which is time consuming.


## 90 +- 30 minutes or more like days?
The official documentation from Microsoft states: 
> Group Policy refresh interval for computers configures all non-domain controller systems within the scope of the policy. By default this is set to every 90 minutes with a random time offset of 0 to 30 minutes, resulting in a refresh interval of 60 to 120 minutes per computer.
[Microsoft on gp refresh intervals](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-2000-server/cc940895)

Sadly that's not the entire truth as many computers don't have the connectivity to refresh the GP in this timely manner. So don't relay on that timeframe.  

## How does a computer refresh it's GPOs?
If we were on GNU/Linux it'd be pretty easy to answer this question => cron jobs or systemd timers.  
The closest thing we have on Windows would be scheduled tasks.

Look at that. There's even a library for GroupPolicy scheduled tasks. Altough it's empty.

Or is it?

```
Get-ScheduledTask | Where {$_.TaskPath -match "GroupPolicy"}

TaskPath                          TaskName
--                                --
\Microsoft\Windows\GroupPolicy\   {3E0A038B-D934-4930-9981-E89C9...}
\Microsoft\Windows\GroupPolicy\   {A7719E0F-10DB-4640-AD8C-490CC...}
```

![Hiding](/b/images/hiding.jpg)

#### {3E0A038B-D834-4930-9981-E89C9BFF83AA} & {A7719E0F-10DB-4640-AD8C-490CC6AD5202}
The `{3E0A038B-D834-4930-9981-E89C9BFF83AA}` task runs the following action every PT1H42M or 102 minutes. (90min + 0-30min offset)
```
Author: {computername}
Id: Group Policy Background Processing
Arguments: /target:computer
Execute: gpupdate.exe
```

Same with `{A7719E0F-10DB-4640-AD8C-490CC6AD5202}`. But instead of refreshing the computer gpos it's refreshing the user gpos. (In this case with PT1H35M interval - 95 minutes)
```
Author: {computername}
Id: Group Policy Background Processing
Arguments: /target:user
Execute: gpupdate.exe
```


## Conclusion
Group policy management isn't a straight forward task and has some obscure things lying in it's way.  
As the two tasks weren't documented anywhere, I've put some time researching those tasks as it was pretty unclear to me that they were legit.
You won't find any documentations on those two scheduled tasks besides some obscure russian forums and trojan / ransomware logs.

See you space cowboy.
