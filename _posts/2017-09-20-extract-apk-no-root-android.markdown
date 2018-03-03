---
layout: post
title:  "Extract apk from a non root Android"
date:   2017-09-17 11:30:00
disqus: true
categories: Mobile
---
When your device is not rooted, and you want to analyse some app, you can't just pull the application apk from it, so to do that, I use this trick.

**NOTE Your android need to have curl binary**
{% highlight Bash %}
adb shell find / -name curl
{% endhighlight %}

If the previous command return the curl path, access your smartphone using,

{% highlight Bash %}
adb shell
{% endhighlight %}

then execute the following command just changing the package name,

{% highlight Bash %}
 package="package_name_goes_here";path=$(pm path $(pm list packages | grep $package | cut -d ':' -f2) | cut -d ':' -f2);filename=$(basename $path);curl --upload-file $path https://transfer.sh/$filename
{% endhighlight %}

in the final you'll receive an transfer.sh link, where your apk file is saved.
