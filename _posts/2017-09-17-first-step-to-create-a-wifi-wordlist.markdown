---
layout: post
title:  "First step to create a wifi wordlist"
date:   2017-09-17 06:01:00
disqus: true
categories: Mobile
---
You can find on the internet various wordlists focused on cracking wifi passwords, but it's not the best wordlists to use in Brazil, so what about get some real brazilian passwords? There is an app called <a href="http://wifimagic.com/" target="_blank">MandicMagic</a>, it's a social network, where the users save password of wifi network that they use or used sometime, well here is our honeypot, we just need to intercept the requests and try to discover how the app works.
In my first attempt I failed, because all the requests made by the app are pinned, but thanks to  <a href="https://github.com/ac-pm/Inspeckage" target="_blank">Inspeckage</a> was really easy to bypass it, the Inspeckage hooks the pinning function, I've just followed the tutorial of how to use <a href="https://acpm.mobi/genymotion-xposed-inspeckage/" target="_blank">Genymotion + Inspeckage + Xposed</a> and intercept all the app requests.

Unfortunately the app does not work how I expect, because it does not use geographic coordinates, there to parameters x and y, and I can't figure out what they are, but I played with this numbers and get to different places, because the response shows the geographic coordinates of the wifi hotspot.

Here is the result I got so far, the function searchWifi is where you have the parameters x and y to play with,
{% highlight Python %}
import urllib
import requests

def searchWifi(headers,x,y):
    url = 'https://api.mandicmagic.com/v4/passwords/search?x=%d&y=%d' % (x,y)
    headers['Authorization'] = 'Digest username="750910213", realm="Protected Area", nonce="59bc6c27d757f", uri="/v4/passwords/search?x=5585&y=11071", response="5964e263fe176915645de1781b6d2b6e", qop=auth, nc=00000003, cnonce="51d6086c0c99359d", algorithm=MD5, opaque="2929b8e007e9c3edd69d915068815d71""'
    headers['Connection'] = 'close'
    r = requests.get(url,headers=headers)
    datas = r.json()
    for data in datas:
        print 'Name: %s' % data['name']
        print 'SSID: %s' % data['ssid']
        print 'Pass: %s' % data['password']
	print 'Latitude:  %s' % data['lat']
        print 'Longitude:  %s' % data['lng']
	print

def login(username,password):
    headers = dict()
    headers['Authorization'] = 'Digest username="562939110", realm="Protected Area", nonce="59bc6d0e57949", uri="/v4/users/login", response="b33fb67d794b23b2c9a989120707c283", qop=auth, nc=00000003, cnonce="08ec0e86bc30740b", algorithm=MD5, opaque="2929b8e007e9c3edd69d915068815d71"'
    headers['User-Agent'] = 'WiFi Magic 3.3.15(android)'
    url = 'https://api.mandicmagic.com/v4/users/login'
    data = {"email":username,"language":"en","password":password,"platform":2,"version":"3.3.15"}
    r = requests.post(url, json=data,headers=headers)
    if r.status_code == 200:
        headers['API-token'] = urllib.quote_plus(r.json()['token'])
        return headers
    else:
        return False

headers = login("email@email.com","yourpasswordhere")
if headers:
    searchWifi(headers,x=5533,y=11096)

{% endhighlight %}
