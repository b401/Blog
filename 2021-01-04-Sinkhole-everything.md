---
author: i4
title: "Sinkhole everything!"
date: M01-04-2021
---

## Hello friends!
As I've got some time on my hands I've decided to bring back my homelab into a working condition. I'm a huge fan of hosting nearly everything myself (besides mail) and so it didn't take long until I had my bind9 working with an internal zonefile for my environment.

So I quickly wrote a program to create a correlated sinkhole list.

### What is a Sinkhole?
A sinkhole intercepts DNS requests and redirects them to a known location (like ::1 or anywhere else). This way, we suppress communication to known C&C servers or otherwise malicious sites.

If you want to learn more about sinkholes, use this resource: [SANS - DNS Sinkhole](https://www.sans.org/reading-room/whitepapers/dns/dns-sinkhole-33523)

### Misc infromation
#### Building a DNS infrastructure

##### Installing & configuring the requirements
Installing bind is as easy as running the following command:

```bash 
apt install bind9
```

The configuration is also straight forward:

_/etc/bind/named.conf.options_
```
options {
	directory "/var/cache/bind";
	allow-transfer {"none";};
	allow-recursion {10.0.0.0/24; localhost;};

	forwarders {
	 	dns.forwarder.ofyour.choosing; // 8.8.8.8 for example
	};
	dnssec-validation auto;
	listen-on-v6 { any; };
};
```

For the purpose of sinkholing certain domains we add another entry in the named.conf file.

_/etc/bind/named.conf_
```
include "/etc/bind/named.conf.blocklist";
```

You won't be able to start it right now as there is no named.conf.blocklist.

##### Internal Zones
I'm using a non-routeable (well not really it's dns) inside my LAN for stuff like [Gollum](https://github.com/gollum/gollum), [Gogs](https://github.com/gogs/gogs) and other services.

So the zonefile looks basically like this:
```
$TTL	604800
@	IN	SOA	localhost. admin.localhost. (
			      2		; Serial
			 604800		; Refresh
			  86400		; Retry
			2419200		; Expire
			 604800 )	; Negative Cache TTL
@	IN	NS	localhost.
@	IN	A	x.x.x.x

vault IN	A	x.x.x.x
cc	IN	A	x.x.x.x
git	IN	A	x.x.x.x
wiki	IN	A	x.x.x.x
```

And the entry in the named config:
```
zone "internal.uauth.io" {
	type master;
	file "/etc/bind/internal.uauth.io";
};
```


### Sinkhole
Introducing [*Blocker*](https://github.com/b401/Blocker) 

While it's not the most elegant solution it does it's job of correlating different blocklists into one coherent zonedefinition.

#### Installation

As we use curl underneath, we need to install the dependencies for the hooks:
``` bash
apt install libssl-dev pkg-config
```

Create the config folder and copy the config from the repository.
``` bash
mkdir /etc/blocker
mv config.yml /etc/blocker
``` 

#### Configuration
My main config looks like this:

``` yaml
named_path: /etc/bind/named.conf.blocklist
path: /etc/bind/blocker/
sinkhole: 127.0.0.1
sinkhole6: ::1
reset: true

blocklists:
  - 20min.ch

onlinelists:
  - https://raw.githubusercontent.com/davidonzo/Threat-Intel/master/lists/latestdomains.txt
  - https://raw.githubusercontent.com/nextdns/cname-cloaking-blocklist/master/domains
  - https://v.firebog.net/hosts/static/w3kbl.txt
  - https://v.firebog.net/hosts/Admiral.txt
  - https://adaway.org/hosts.txt
  - https://orca.pet/notonmyshift/hosts.txt 
  - https://hostfiles.frogeye.fr/firstparty-trackers-hosts.txt 
  - https://gitlab.com/curben/urlhaus-filter/-/raw/master/urlhaus-filter-hosts.txt
  - https://raw.githubusercontent.com/hoshsadiq/adblock-nocoin-list/master/hosts.txt 
  - https://raw.githubusercontent.com/bongochong/CombinedPrivacyBlockLists/master/newhosts-final.hosts 
```

#### Running
Just execute the binary which you can download from the release page on the repository.

##### crontab for automation
``` bash
0 3 * * * /usr/local/bin/binder
```


### Stats
#### Entries
With all the lists I have around **472 634** domains sinkholed. 

This doesn't comes without sacrifices. While my bind9 server beforehand needed around 128MB to work properly (lol), it requires now around 4GB. The list gets purged every night and replaced with a fresh one.


#### Zonefile
The zonefile initself is nothing special

_/etc/bind/blocker/blocker.zone_
``` bash
$TTL 604800
@ IN SOA localhost. admin.localhost. (
	1
	3H
	15M
	1W
	1D
);

	IN	NS	localhost.
	IN A 127.0.0.1
	IN TXT 127.0.0.1
	IN AAAA ::1
``` 

We redirect all NS, A(ipv4), AAAA (ipv6) an TXT requests to 127.0.0.1/::1.
All the domains get listed in the named.conf.blocklist.


_/etc/bind/named.conf.blocklist_
``` bash
zone "0-24bpautomentes.hu"{
        type master;
        file "/etc/bind/blocker/blocker.zone";
        check-names ignore;
};

zone "0-day.us"{
        type master;
        file "/etc/bind/blocker/blocker.zone";
        check-names ignore;
};
``` 


## Conclusion
**Would it have been easier to write it in python?**  
ofc but where's the fun in there?


**Does it work?**
``` bash
uauth@hakase:/etc/bind$ ping ads.youtube.com
PING ads.youtube.com(localhost (::1)) 56 data bytes
64 bytes from localhost (::1): icmp_seq=1 ttl=64 time=0.026 ms
``` 
