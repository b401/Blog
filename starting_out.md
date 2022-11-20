---
title: Starting out in Security
author: i4
date: M11-19-2022
---

Security is an extremely wide topic that touches basically every single technology out there
starting out can be pretty dauting and it's oftentimes unclear what knowledge is required.  

I've seen a lot of people asking the same question on [matrix](https://matrix.to/#/#cybersecurity:matrix.org).  

> **Where do I start?**

Instead of responding to every single person individually I thought it would be beneficial to give people new into IT (and security) a starting point to introduce them to resources and the right mindset which distinguishes a good security / IT person from a [LARPER](https://en.wikipedia.org/wiki/Live_action_role-playing_game).

## IT OR SECURITY

I usually recommend people to get a job as a sysadmin or helpdesk before they consider moving to security.  
This might seem weird but the amount of knowledge that you can get by helping people and building systems which translates 1:1 into security is amazing.

I worked for some years in a sysadmin role where I also did first and second level support. While this was usually pretty boring work it showed me what impact my actions had. 

* Setup a new [GPO](https://en.wikipedia.org/wiki/Group_Policy) to restrict service accounts from interactive logon?
    * Phone starts ringing and users complaining that business processes stopped working.
* Disabled [SMBv1](https://en.wikipedia.org/wiki/Server_Message_Block) on Servers?
    * [ERP](https://en.wikipedia.org/wiki/Enterprise_resource_planning) stopped working.
	
If you have the opportunity to start in security right away, take it! If you don't consider to start working in one of the two mentioned spaces.  


Just don't stop learning. 

**EVER!**

## Mindset

Having the right mindset is probably the most important thing. This is not only related to security but IT in general.  
Back when I started out I was reading a lot of [E-Zines](http://www.phrack.org/). Although many of the issues are already a little old, don't discount them! Age does not matter much in terms of security vulnerabilities and how to protect resources. 

For example: We've seen [PoD(Ping of death)](https://en.wikipedia.org/wiki/Ping_of_death) coming back multiple times across different releases and the core technologies don't change that often. 

Teaching somebody with the right mindset is easier then somebody that already knows the basics but is unmotivated and thinks he's all knowing.

## READ EVERYTHING

**EVERYTHING** is related to security.  
There are no technologies that don't need protection (or can't be attacked).  
This includes forcing yourself to read boring articles and even horribly translated whitepapers and Microsoft KB pages.

Stuff that might appear useless **will** at the start come in handy!

## CORE TECHNOLOGIES

Learn the core technologies. 
* [Networking (TCP/IP)](https://www.amazon.com/TCP-Illustrated-Vol-Addison-Wesley-Professional/dp/0201633469)
* [Web technologies](https://www.amazon.com/Tangled-Web-Securing-Modern-Applications/dp/1593273886)
* Core protocols
  * [SMTP](https://www.cloudflare.com/learning/email-security/what-is-smtp/)
  * [FTP](https://en.wikipedia.org/wiki/File_Transfer_Protocol)
  * [DNS](https://en.wikipedia.org/wiki/Domain_Name_System)
  * [DHCP](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol)
  * [SSH](https://en.wikipedia.org/wiki/Secure_Shell)
  * [SNMP](https://en.wikipedia.org/wiki/Simple_Network_Management_Protocol)
  
## PROGRAMING & SCRIPTING

I'm in the strong opinion that somebody that works in Security should know their way around languages.  

It doesn't matter what language that you start out with but here are some ideas:
* [Python](https://www.python.org/)
* [Golang](https://go.dev/)
* [PHP](https://www.php.net/)
* [C](http://cslabcms.nju.edu.cn/problem_solving/images/c/cc/The_C_Programming_Language_%282nd_Edition_Ritchie_Kernighan%29.pdf)
* [Rust](https://www.rust-lang.org/)

I usually recommend people to start with python as it's the easiest language to quickly get satisfying results and go from there. If you decide later to pick up a more low-level language like C feel free to do so.  

Once you've learned a language it usually gets significantly easier to adapt to another.  
[Exercism](https://exercism.org/) is a great resource to learn new languages. Give it a try!

### OS "SPECIFIC"  

I keep this in quotations marks since they aren't really OS specific.  
You should learn Windows: [Powershell](https://learn.microsoft.com/fr-fr/powershell/scripting/overview?view=powershell-7.3) and Linux: [bash](https://exercism.org/tracks/bash).  
They will help you to quickly develop ugly scripts and interact with remote systems (in terms of powershell).

### DOMAIN SPECIFIC LANGUAGES (DSL)

While not necessarily a scripting or programming language knowing certain DSL is crucial.  
This includes:
* [SED](https://www.digitalocean.com/community/tutorials/the-basics-of-using-the-sed-stream-editor-to-manipulate-text-in-linux) - String manipulation
* [SQL](https://www.sqltutorial.org/) - Database interactions
* [AWK](https://exercism.org/tracks/awk) - You don't need to be a god. Just learn how to print different columns ;)

**Other "languages"**
* [Markdown](https://www.markdowntutorial.com/) - Used for writing manuals & documents
* [YAML](https://www.cloudbees.com/blog/yaml-tutorial-everything-you-need-get-started) - Human readable file format
* [JSON](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Objects/JSON) - Same as YAML but more targeted at machines
* [HTML](https://www.w3schools.com/html/) - I know w3schools had some contraversy but it's fine now.
* [CSS](https://developer.mozilla.org/en-US/docs/Learn/CSS/First_steps/Getting_started) - Apply formatting to HTML

## KNOW HOW TO ASK QUESTIONS

You shouldn't be afraid to not know every single thing but ask the right questions.  

**Bad**
> Sup guys I want to hack my friends machine (with his permission) how do I do that?

**Good**
> Sup guys I want to hack my friends machine (with his permission). I found this tool and tried to run it
> but it returns an error code.
> 
> [11/18/2022 08:06:29] [e(0)] core: NoMethodError : undefined method `+' for nil:NilClass
> ...
> 
> I tried to google for that error and found a github [issue](https://github.com/rapid7/metasploit-framework/issues/17276) but I don't know how to proceed.

- Don't be annoying. (Nobody owns you their time)
- Don't expect an answer right away.
- Don't expect spoon feeding.
- Include what you already tried

Security is hard. Security is a team effort. You are not alone! 

## DON'T BE A ZEALOT 

You might be wrong.   
Even new people can bring interesting perspectives to discussions or help you figuring stuff out. Don't discount someones opinion just because of their age or inexperience.

This includes OS fanatics as well. The times I've read: 

> Hurr Windows is insecure. Use GNU/Linux. All hackers use it!!111

is astonishing.

While I agree that knowing GNU/Linux is an important part of security, most big company infrastructures run on Windows.  

## BE OS AGNOSTIC!

I wouldn't recommend to just jump straight into GNU/Linux. Use [WSL2](https://learn.microsoft.com/en-us/windows/wsl/install) and get comfortable with the CLI.

If you feel secure enough try running [Ubuntu](https://ubuntu.com/) in a [virtual machine](https://www.virtualbox.org/).

It's kinda funny that people think that the common GNU/Linux distributions are [more](https://madaidans-insecurities.github.io/linux.html) secure than Windows 10/11. The main reason behind this is historically and usage based (No 2022/2023 isn't the year of the Linux desktop) as well as the horrible defaults that Microsoft ships Windows with.

## TRY TO IMPROVE

Try to not be a script kiddie. 
* Understand what impact the tools and commands you run have. Don't just fire off random metasploit modules against targets without knowing what they actually do.
* Write your own [NSE](https://nmap.org/book/man-nse.html) modules even if it means just rewriting something that already exists.
* Don't be afraid of making mistakes but take something away from them.
 * No point in running a command which fails without trying to figure out **why** it failed.


## CTFs

While I hold the opinion that CTF don't necessarily translates all the skills to the real-world. It teaches you the right mindset and common found issues so it's worth the time to do some.

As for a beginner I would recommend starting out with [OverTheWire - Bandit](https://overthewire.org/wargames/bandit/) which also teaches you bash and some common GNU tools. [HackTheBox](https://www.hackthebox.com/) also has some great starting machines.

## REDISCOVER THE WHEEL

Only because something already exists doesn't mean that you can't rediscover it.  
Building already existing tools in your own vision (and maybe in a different language) can be a great way to learn a **lot** of valuable knowledge.  

## FOSTER THE COMMUNITY

Try to be a part of the community. It doesn't matter if you're new or inexperienced. Try to learn and engage! Nobody started out as a wizard.

## BUILD A FOOTPRINT

You won't believe how much this impacts recruiting decisions. If I see somebody that applied to a job with a personal mail domain be sure that I'll check out their DNS records.

* Create a [Github](https://github.com/) account and upload your creations!
* Publish a blog or a personal website

## TLDR; GIVE ME RESOURCES

* **Books**  
    * [Networking (TCP/IP)](https://www.amazon.com/TCP-Illustrated-Vol-Addison-Wesley-Professional/dp/0201633469) - Network basics
    * [Web technologies](https://www.amazon.com/Tangled-Web-Securing-Modern-Applications/dp/1593273886) - Covers Browsers & Web technology security
    * [Black hat python](https://www.amazon.com/Black-Hat-Python-2nd-Programming/dp/1718501129) - Python for hackers
    * [Black hat go](https://www.amazon.com/Black-Hat-Go-Programming-Pentesters/dp/1593278659) - Golang for hackers
    * [Windows internals](https://learn.microsoft.com/en-us/sysinternals/resources/windows-internals) - Windows OS deep-dive
	* [Advanced Penetration Testing](https://www.amazon.com/Advanced-Penetration-Testing-Hacking-Networks/dp/1119367689) - Build your own C2 infrastructure
* **Online resources**
	* [Begin.RE](https://www.begin.re/) - Introduction to reverse engineering
    * [HackTricks](https://book.hacktricks.xyz/welcome/readme) - Collection of hacking techniques
    * [Ptest Method](https://ptestmethod.readthedocs.io/en/latest/index.html) - Collection of different security techniques
	* [Hacker Recipes](https://www.thehacker.recipes/) - Mostly AD based exploitation techniques
* **CTF**  
    * [OverTheWire - Bandit](https://overthewire.org/wargames/bandit/) - Online beginner CTF without the need to install anything.
    * [HackTheBox](https://www.hackthebox.com/) - CTF community with a huge amount of exploitable VMs
* **Tools**  
    * [Sysinternals](https://learn.microsoft.com/en-us/sysinternals/) - Windows advanced system utilities **learn this!**
    * [Metasploit](https://www.metasploit.com/) - Huge exploitation framework
    * [CrackMapExec](https://github.com/Porchetta-Industries/CrackMapExec) - Swiss army knife for remote windows exploitation
    * [SQLMap](https://sqlmap.org/) - Ezpz script kiddie sql injectsions
    * [Mimikatz](https://blog.gentilkiwi.com/mimikatz) - Windows secrets dumper
	* [nmap](https://nmap.org/) - Port & Security scanner
	* [OpenVAS](https://www.openvas.org/) - Vulnerability scanner
* **Virtualization**  
    Get comfortable with at least one
    * [Virtualbox](https://www.virtualbox.org/) - Free hypervisor
    * [VMWare](https://www.vmware.com/) - Free hypervisor
* **Containers**  
    * [Docker](https://www.docker.com/) - Seriously **learn** docker!
* **Distributions/OS**  
    * [Kali](https://www.kali.org/) - Distributions which already ships with a lot of tools.
    * [Windows Server 2022](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022) - Windows server ISO (Run in a VM and build an AD)
    * [Windows 10 iso](https://www.microsoft.com/nb-no/software-download/windows10) - Windows 10 ISO (Run in a VM and join to the domain)
    You can use [massgravel](https://github.com/massgravel/Microsoft-Activation-Scripts) to active the windows systems.
* **Blogs and Twitter**
    * [BleepingComputer](https://www.bleepingcomputer.com/) - Reports about vulns & breaches
    * [packet storm](https://packetstormsecurity.com/) - Public exploits & whitepapers
    * [exploit-db](https://www.exploit-db.com/) - Public exploits & whitepapers
    * [DFIR Report](https://thedfirreport.com/) - Reports about recent breaches
    * [SwiftOnSecurity](https://nitter.it/SwiftOnSecurity) - Sometimes good posts about IT in general
