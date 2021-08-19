---
layout: post
title: "Burpsuite Academy系列教程-Access control"
date:   2021/8/19
tags: [Burpsuite Academy系列教程]
comments: true
author: c-uncle
---

# What is access control?

access control，访问控制，指限制人（或物）的行动（执行某些操作或访问某些资源）。

Authentication，身份认证，用来证明人（或物）的身份。

Session management，会话管理，用来证明一系列HTTP请求来自同一用户。

在Web应用中，Authentication和Session management用的比较多。



从用户的角度看，访问控制可以被分为以下三类。

- Vertical access control，垂直访问控制
- Horizontal access control，水平访问控制
- Context-dependent access control，上下文相关访问控制



## Vertical access control

定义：不同类型的用户有不同的权限，并且不同类型的用户不一定会拥有对方的所有权限。

## Horizontal access control

定义：不同的用户拥有对某种类型的资源中的一部分的访问权限。

举例：银行账户，每个人都有账户，但都只能操作自己的账号，不能操作其他人的账号。

## Context-dependent access control

定义：对使用某个功能的权限，需要另外一个功能完成后的认证。

举例：购物，付款后就不能再修改账单中的物品。修改密码，改密码前需要先验证身份。



# Examples of broken access controls

## Vertical privilege escalation

如果用户可以访问他们不能访问的功能，那么就会造成垂直越权。

### Unprotected functionality

当应用没有对敏感功能做保护，导致攻击者也可以执行敏感函数，这时就导致了垂直越权。

例如：没有对管理员页面做保护，导致攻击者可以直接访问管理员页面，从而实现垂直越权。

#### Lab: Unprotected admin functionality

访问/robots.txt，拿到后台url为/administrator-panel

应用为了反爬虫，将自己后台地址放在的/robots.txt，但是没有对后台做任何权限验证。

#### Lab: Unprotected admin functionality with unpredictable URL

ctrl+U，查看源代码，在JS代码中，找到后台地址/admin-y4ecy5。

虽然应用对后台地址进行加密（指加上奇怪后缀，防止攻击者爆破），但是依然可以在前端中脚本代码找到蛛丝马迹。

### Parameter-based access control methods

基于参数的访问控制，通过在http请求中加上参数来验证来者身份。这种做法很无用，因为人人都可以构造参数，根不防不住。

#### Lab: User role controlled by request parameter

直接访问/admin是不行的。可以用教程提供的账号登录一下。通过抓包可以看到，cookie字段中有一个Admin=false，利用bp修改为Admin=true即可绕过限制。

### Broken access control resulting from platform misconfiguration

利用平台错误配置来绕过访问限制。

应用程序可以通过限制用户可访问的URL来实现访问控制，举例：可以定义"DENY: POST, /admin/deleteUser, managers"，该规则限制用户不能对/admin/deleteUser发起POST请求。

但是在HTTP协议中，可以利用一些头部字段来绕过限制，例如，X-original-URL和X-Rewrite-URL。

#### Lab: URL-based access control can be circumvented

访问/admin会现实Access Denied。可以在HTTP请求的头部加入X-Original-URL字段，其值为/admin，即可绕过限制。

删除用户名的操作就是添加头部字段X-Original-URL: /admin/delete，注意删除操作所需要的参数要填在第一行的URL后面。

将参数填在X-Original-URL是没有效果的。

```
GET /?username=carlos HTTP/1.1
Host: ac0f1fa61e91cc46805190b400d70019.web-security-academy.net
X-Rewrite-URL: /admin/delete
```

#### Lab: Method-based access control can be circumvented

这个lab的目标是在不使用Administrator账户的情况下，提高wiener账户权限。

先登录Administrator账户，给carlors账户提高权限，抓取数据包。

```
POST /admin-roles HTTP/1.1
Host: ac681fef1f7fe43480c313d8000f002b.web-security-academy.net
Connection: close
Content-Length: 30
Cache-Control: max-age=0
sec-ch-ua: "Chromium";v="92", " Not A;Brand";v="99", "Microsoft Edge";v="92"
sec-ch-ua-mobile: ?0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.131 Safari/537.36 Edg/92.0.902.73
Origin: https://ac681fef1f7fe43480c313d8000f002b.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://ac681fef1f7fe43480c313d8000f002b.web-security-academy.net/admin
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: session=oA1FQoflXSDzHIQvYbMH70ZL9Fbjzemh

username=carlos&action=upgrade
```

尝试使用Repeat模块重复数据包失败，显示401，未授权。

将数据包改为GET请求，再次发送数据包，修改成功。

#### 小结

一般来说，通过平台来实现访问控制有两种方法，一是限制URL，二是限制HTTP请求的模式。

前者可以尝试使用X-original-URL或X-rewrite-URL来绕过，后者可以尝试使用修改请求方法来绕过。



## Horizontal privilege escalation

第一种水平越权的攻击方法，就是直接修改URL里的参数，导致出现越权访问。

利用查看用户信息的URL由参数id来显示对应的用户信息，但是没有做保护，导致攻击者可以任意查看用户信息。

#### Lab: User ID controlled by request parameter

目标：获取carlos的API key。

先用wiener:peter登录，点击myaccount，可以看到一个API KEY。

然后看这个页面的URL，可以看见这个页面中带有一个参数id，其值为wiener。（如果没有就点一下myaccount，登录后跳转的myaccount页面没有这个参数）

构造id=carlos，成功获得carlos的API KEY

---------

有时候，参数id的值是不可预测，应用可能会用一个唯一的数值（globally unique identifiers， GUIDs，全局唯一标识符）来表示用户。但是我们可以从其他地方来获取用户的ID，例如用户的消息或评论。

#### Lab: User ID controlled by request parameter, with unpredictable user IDs

目标：找到carlos的GUID，获取其API KEY。

这个lab网站是一个博客，把文章挨个看一下，其中有一篇文章是carlos发的，点击carlos的名称，可以看到url中的userid。

用wiener:peter登录，将myaccount页面的URL中wiener的id替换为carlos的id即可。

------------

有些应用会检测发起请求的用户是否有权限，如果没有就返回一个重定向，但是有时候没处理好，我们依然可以在重定向的响应中获取敏感信息。

#### Lab: User ID controlled by request parameter with data leakage in redirect

目标：绕过限制获取carlos的API KEY

先用wiener:peter登录，点击myaccount，可以看到一个API KEY。

然后看这个页面的URL，可以看见这个页面中带有一个参数id，其值为wiener。（如果没有就点一下myaccount，登录后跳转的myaccount页面没有这个参数）

构造id=carlos，虽然响应是302（重定向），但是在bp里依旧能看见越权成功的内容，成功获得carlos的API KEY

-------------

## Horizontal to vertical privilege escalation

水平到垂直的越权。有时候，攻击者可以利用水平越权看到管理员账户的页面，这个页面可能会泄露密码或提供了修改密码的功能。这时，攻击者就实现一个水平到垂直的越权。

#### Lab: User ID controlled by request parameter with password disclosure

目标：获取管理员账户，并删除carlos的账户。

登录wiener账户，可以看见在myaccount页面里有被掩盖的password，使用F12查看真实的password。

利用水平越权漏洞，查看administrator的密码，并登录administrator，删除carlos的账户。

---------------------------

### Insecure direct object references

不安全的直接对象引用。应用程序使用用户提交的参数作为对象直接引用，导致攻击者可以控制参数实现攻击。举例：任意文件读取。

#### Lab: Insecure direct object references

目标：获取carlos的密码，并登录其账户。

点击live chat->view transcript，这时开始下载聊天记录，通过bp抓包，发现这是静态资源，通过下载1.txt获取密码。（为什么是1.txt？因为第一次点下载的是2.txt，说明1.txt已经存在了）

应用程序会将聊天记录保存在服务器上，并提供给用户访问进行下载。

### Access control vulnerabilities in multi-step processes

多步骤访问控制漏洞。例如管理员更新用户信息操作，可分为三步。

1. 加载特定用户的信息
2. 修改用户信息
3. 查看更新后的用户信息

如果应用程序只对前两步进行访问控制，而忽略第三步，那么就会出现攻击者直接进行第三步，而程序以为前两步已完成，从而实现攻击。

#### Lab: Multi-step process with no access control on one step

目标：使用wiener:perter为wiener提高权限。

使用administrator账户为carlos提高权限，并观察。

发现提升权限需要两步。

1. 点击提权，跳转到新页面
2. 在新页面里点击Yes，成功提权。

这是第一次点击的POST请求。

```
POST /admin-roles HTTP/1.1
Host: ac161f681f87b09b807cb187009300f8.web-security-academy.net
Connection: close
Content-Length: 30
Cache-Control: max-age=0
sec-ch-ua: "Chromium";v="92", " Not A;Brand";v="99", "Microsoft Edge";v="92"
sec-ch-ua-mobile: ?0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.131 Safari/537.36 Edg/92.0.902.73
Origin: https://ac161f681f87b09b807cb187009300f8.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://ac161f681f87b09b807cb187009300f8.web-security-academy.net/admin
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: session=DEk9JzHeiQYy43t2Rs7JpHABuzuu765a

username=carlos&action=upgrade
```

这是点击Yes以后的POST请求

```
POST /admin-roles HTTP/1.1
Host: ac161f681f87b09b807cb187009300f8.web-security-academy.net
Connection: close
Content-Length: 45
Cache-Control: max-age=0
sec-ch-ua: "Chromium";v="92", " Not A;Brand";v="99", "Microsoft Edge";v="92"
sec-ch-ua-mobile: ?0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.131 Safari/537.36 Edg/92.0.902.73
Origin: https://ac161f681f87b09b807cb187009300f8.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://ac161f681f87b09b807cb187009300f8.web-security-academy.net/admin-roles
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: session=DEk9JzHeiQYy43t2Rs7JpHABuzuu765a

action=upgrade&confirmed=true&username=carlos
```

观察两次POST请求，可以看到第二次请求多了一个confirmed参数，也就是说，我们只需要构造第二个请求就可以实现越权攻击。

### Referer-based access control

基于Referer字段的访问控制。

有些站点会对HTTP请求的Referer字段进行验证，以限制发起该请求的用户的权限。

#### Lab: Referer-based access control

目标：提高wiener的权限。

先用administrator提升carlos的权限，观察HTTP请求。

```
GET /admin-roles?username=carlos&action=upgrade HTTP/1.1
Host: ac191f011eabfcd68121d84e00b8000f.web-security-academy.net
Connection: close
Cache-Control: max-age=0
sec-ch-ua: "Chromium";v="92", " Not A;Brand";v="99", "Microsoft Edge";v="92"
sec-ch-ua-mobile: ?0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.131 Safari/537.36 Edg/92.0.902.73
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://ac191f011eabfcd68121d84e00b8000f.web-security-academy.net/admin
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: session=stly6A95K5BYg0ljncq0HeXyqbgjPfvR
```

看到Referer，猜测程序会验证请求是否来自/admin。保留Referer字段，构造wiener的请求即可。

### Location-based access control

有些站点会通过用户的地理位置来限制权限，绕过的办法一般是使用VPN或网络代理，或操纵客户端地理定位机制。



# How to prevent access control vulnerabilities

1. 不要仅仅依赖混淆来控制访问
2. 一个资源除非被设定为公开资源，否则默认不允许访问
3. Wherever possible, use a single application-wide mechanism for enforcing access controls（不能理解）
4. 在代码层面，强制要去开发者设定资源的访问权限，默认设定拒绝访问。
5. 彻底审核和测试访问权限，确保设计有用。