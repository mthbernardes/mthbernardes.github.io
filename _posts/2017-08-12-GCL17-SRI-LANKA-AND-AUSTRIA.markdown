---
layout: post
title: Write Up GCL17 - SRI LANKA and AUSTRIA
description: GCL17 SRI LANKA and AUSTRIA  challenge
keywords: gcl17, python, rtfm, write up, G4mbl3r
disqus: true
tags: [GCL17]
categories: Reversing
---
#### SRI LANKA
The first thing to do is to run the "file" command, which will indicate that the file is an elf32. After that the binary was analyzed on r2 and I ended up realizing that I forgot to do the most simple and obvious thing as I am certified SGSE ( Strings grep Specialist Engineer), so I just executed the following command:
{% highlight Bash %}
strings -e s c9c18a2c3f65bb8d1f6133765889774a1224cbe993edcc1641275be12b29b10a | less
{% endhighlight %}

And found the most beautiful string in the world, PYTHON
{% highlight Bash %}
mod is NULL - %s
PYTHONPATH
PYTHONHOME
import sys
del sys.path[:]
sys.path.append(r"%s")
argv
{% endhighlight %}

Now every thing became simple becasue that just looked like a challenge I've created a while ago. So my guess was that It's just a python code probably compiled with py2exe or pyinstaller just generating some bytecode behind, like java, so I used the pyinstaller decompiler(![PyInstaller Extractor download](https://sourceforge.net/projects/pyinstallerextractor/)) extracted the code, and executed the following command:

{% highlight Bash %}
➜ strings * | grep -R -l t0k3n
pyi_darchive
{% endhighlight %}

Opened the pyi_darchive file, and there was it,
{% highlight Python %}
t0k3n1 = [116, 104, 51, 32, 120, 120, 120, 32, 116, 48, 107, 51, 110, 32, 121,48, 117, 32, 115, 51, 51, 107, 32, 105, 115, 58, 32, 102, 48, 108,108, 48, 119, 32, 116, 104, 51, 32, 114, 52, 98, 98, 49, 116]
{% endhighlight %}

And to get the ascii, I write the resolv.py

{% highlight Python %}
t0k3n1 = [116, 104, 51, 32, 120, 120, 120, 32, 116, 48, 107, 51, 110, 32, 121,48, 117, 32, 115, 51, 51, 107, 32, 105, 115, 58, 32, 102, 48, 108,108, 48, 119, 32, 116, 104, 51, 32, 114, 52, 98, 98, 49, 116]
out = list()
for l in t0k3n1:
    out.append(chr(l))
print ''.join(out)
{% endhighlight %}

**FLAG:** th3 xxx t0k3n y0u s33k is: f0ll0w th3 r4bb1t

#### AUSTRIA
Continuing with this file after the CTF, I just read the rest of the file and found that if I created I file with the coordinates, it will print the flag to me, so debugging the pyi_darchive, I created the file uKBcWeOjxleffzzZmpWXLUrSubOHWzaeVO.wgz, and put the coordinates inside it, because the script compares the content of the file with the values on coords var, so the file looks like that

{% highlight Bash %}
➜ cat uKBcWeOjxleffzzZmpWXLUrSubOHWzaeVO.wgz
33.744970 -84.372165
52.369561 4.893806
38.905475 -77.031695
{% endhighlight %}

I just run the script, and then here is the flag,

**FLAG:** the yyy t0k3n y0u s33k is: d33p_1n_th3_0z4rks
