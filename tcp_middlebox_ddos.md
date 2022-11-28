---
title: TCP Middlebox Amplification
author: i4
date: M11-28-2022
---

Recently I've subscribed the [ASN](https://www.arin.net/resources/guide/asn/) of the company I work for to [shadowserver](https://www.shadowserver.org/).  
A nonprofit security organization that helps securing the internet by working with different ASN holder (ISP, companies etc.) together.

One of their subscription list identifies [TCP Middlebox reflection](https://www.shadowserver.org/what-we-do/network-reporting/vulnerable-ddos-middlebox-report/).   
I highly recommend reading through their report & the [whitepaper](https://geneva.cs.umd.edu/papers/usenix-weaponizing-ddos.pdf).

To summarize the issue, it is possible to weaponize middleboxes through non-standard TCP requests.   
Usually amplification attacks are being done using the UDP protocol (since there is no need for a three-way handshake or [sequence number](https://packetlife.net/blog/2010/jun/7/understanding-tcp-sequence-acknowledgment-numbers/) guessing).

In this case however, middleboxes (for example firewalls or state censorship infrastructure) can respond to the initial first packet basically MitM the connection.


Since reading the whitepaper can quite complicated I've decided to quickly write a post to explain what the issue is, how you can verify and mitigate it.


### ZMAP VERIFY

There are multiple ways to verify the issue and dig deeper to find the root cause.  
The whitepaper suggest using a zMap [fork](https://github.com/Kkevsterrr/zmap) with their [custom module](https://github.com/Kkevsterrr/zmap/blob/master/src/probe_modules/module_forbidden_scan.c). You'll have to install some dependencies to compile zMap. If you don't want to compile stuff yourself, go further down to the __SCAPY VERIFY__ text.

__Dependencies__
```
sudo apt-get install build-essential cmake libgmp3-dev gengetopt libpcap-dev flex byacc libjson-c-dev pkg-config libunistring-dev
```

__Building__

- If you decide to compile it yourself, make sure to replace the `HOST` variable in `module_forbidden_scan.c`

_src/probe_modules/module_forbidden_scan.c_
```
17 #define HOST "youporn.com"
```

```
cmake . 
make -j4
sudo make install
```

Afterwards you can run it with the following arguments:

__zMap__

- Make sure to keep a wireshark/tcpdump instance open in the background to capture the response.
- Replace `-p 80` with the port you want to check
```
sudo zmap --probe-module=forbidden_scan -p 80 x.x.x.x/32
```




### SCAPY VERIFY

If you don't want to use zmap - maybe because you don't like installing stuff system wide - you can also use [scapy](https://scapy.net/).  

__Creating a virtual environment & install scapy__
```
# Needs venv installed
python -m venv .env
source .env/bin/activate
pip3 install scapy
```

To fire a simple `GET/SYN` request you can use the following script:
```
from scapy.all import *

def syn_get():
	packet = IP(dst="xxx.xxx.xxx.xxx")/TCP(dport=80, flags='S')/"GET / HTTP/1.1\r\nHost: youporn.com\r\n\r\n"
	return packet
		
	if __name__ == '__main__':
		pkt = syn_get()
		send(pkt)
```
or just use the interactive scapy shell

```
send(IP(dst="xxx.xxx.xxx.xxx")/TCP(dport=80, flags="S")/"GET / HTTP/1.1\r\nHOST: youporn.com\r\n\r\n")
```




### TRAFFIC

The reason why we're using `youporn[.]com` is quite simple. We want to trigger the adult filter on the middlebox.

Looking at the traffic in wireshark it's clear what exactly happens.

![TCP Wireshark](/b/images/tcp_wireshark.png)

We see 3 packets.

1. Request: Initial GET sent in a `SYN` request
    - Includes Host: youporn[.]com
2. Response: 503 response with a full html block page
	- `PSH, ACK` instead of `SYN, ACK`
3. Response: Rest of the 503 response 
    - The whole response didn't have enough place in a single package (segment data for TCP is max 1400 bytes)
	
Now if an attacker would spoof the `src` address of the initial GET request, he would be able to redirect the response of the middlebox to any victim. (amplification of around 24%)

### RESPONSE

The response is just a 503 explaining the page we're trying to visit has been blocked in accordance with the company policy.

```
...
<h1>Web Page Blocked</h1>
<b>Access to the web page you were trying to visit has been blocked in accordance with the company policy.</b><br/><br/>
<b>User:</b>xx.xxx.xx.xx <br/>
<b>URL:</b>youporn.com/ <br/>
<b>Category:</b>adult <br/>
...
```

This could in certain cases also leak internal information or be much much bigger (see whitepaper).


### MITIGATION

The simplest way to mitigate this issue is by disallowing answers on `SYN` packets that are larger then 100 byte.  
`SYN` packages that are larger then 0 bytes should be looked at more closely anyway.

Akamai has some more information about this: https://www.akamai.com/blog/security/tcp-middlebox-reflection
