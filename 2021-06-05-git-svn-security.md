---
title: "Scanning 2.5 million domains - part 2"
author: i4
date: M06-05-2021
---

## Hello friends!
Sorry friends I let it slide a little as I was building some other stuff for my internal network.  
Just as I said in my last post. I want to highlight the public exposed .git and .svn directories which are freely available on the .ch domains and in case of the .git and .svn folders can pose a substantial risk.

## Dataset
Sadly I found that my dataset is quiet lacking. I suspect that my initial configuration of 1 second timeout was way too low.   
Out of the 2.6 million domains I got around 270'000 responses on http&https.

![timeout](/b/images/timeout.png)

I can't really tell how many of them really had a accessable webserver on their domain running as I was scanning all domains and nothing holds you back to configure an A/CNAME/AAAA record for different purposes than serving the web.

*But do not fret!*  
A new scan is already running on a more powerful system.
I'll post the new findings in a short update as soon as it's done. (Maybe I'll even create a database where you can do the lookup yourself.)


## Git & SVN
If you ever worked on a bigger programming project chances are high you used a versioning system like git or SVN. The latter is not that much used as it's lacking a infrastructure like github or gitlab and the centralised approach doesn't see much use today as more and more services decentralize. (Also it's crap.)

>Git is a free and open source distributed version control system designed to handle everything from small to very large projects with speed and efficiency.
- [git](git.scm.com)

> Apache Subversion (SVN) is a software versioning and revision control system distributed as open source under the Apache License. Software developers use Subversion to maintain current and historical versions of files such as source code, web pages, and documentation
- [Wikipedia](https://en.wikipedia.org/wiki/Apache_Subversion)

### Exposed versioning systems
"Where's the problem. Everything is available on the site anyway" you might say.  
Programmers tend to be lazy and quite forgetful. There's a high chance that someone committed (added) credentials to their sourcecode and pushed it to a repository. While it isn't such a big deal (but still quite bad) to do this in a private repository. One can see how bad this would be if it's available to a malicious actor which in this case would be me ;D

Many admins disallow access to `/.git` or `/.svn` but tend to forget to also include files in the subdirectories like `./git/config` or `./git/HEAD`.
Ohayou takes advantage of that with this piece of code:

```
r = await self.get(".git/HEAD", grab = True)
self.flags["git"] = self.check_helper(r,"ref: refs/heads/")
```

On each domain Ohayou tries to access {domain}/.git/HEAD and checks if it includes the string `ref: refs/heads/`.
For .svn it's slightly different as SVN works with SQLITE. So it tries to access the database directly.

```
r = await self.get(".svn/wc.db", grab = True)
self.flags["svn"] = self.check_helper(r,"SQLite")
```

### Findings
#### GIT
So. let's get to the data.  
How many exposed directories did ohayou find?  
Less than anticipated and more than it should.

![git_diag.png](/b/images/git_diag.png)

Out of the 2'426'038 domains that gave a response before the timeout and had a HTTP/HTTPS service, **56** had their .git exposed.  

There wasn't much of a point to create a diagram including the whole dataset, but I like to bring some color into my posts!  
While this number isn't that super duper high I'm astonished at the HTTP services exposing .git.

![http git](/b/images/http_git.png)

#### SVN
Alright.  
What about .svn?

>> 0

Not that surprising as SVN is mostly used for inhouse development and is rarely exposed outside the environment.

If you want to hunt exposed .git and .svn directories yourself, check out the following Chrome extension from which I borrowed the regex:
- [dotgit](https://chrome.google.com/webstore/detail/dotgit/pampamgoihgcedonnphgehgondkhikel?hl=en)


Also special thanks to [Ant0i](https://blog.ant0i.net/) which gave me the idea to scan for .git directories.  
Check his blog out as it's much more professional and more in-depth than mine (aside from the lack of good memes.)! 

### What's next?

I thought about including the security.txt analysis in this post but decided against it as it's an important subject which will hopefully be realized in a full RFC.

**UPDATE**: security.txt is now a real RFC! Get started [here](https://securitytxt.org/)
