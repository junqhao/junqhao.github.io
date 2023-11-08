---
layout:     post
title:      "首次使用Jekyll的问题"
subtitle:   ""
date:       2023-11-08 21:30:00
author:     "Self"
header-style: text
catalog: true
tags:
    - Web
---

[Untitled.png]:/img/post/20231108/Untitled.png
[Untitled1.png]:/img/post/20231108/Untitled1.png
[Untitled2.png]:/img/post/20231108/Untitled2.png

## 背景

我的个人blog使用的是JekyII，在执行bundle install的环节出现无法安装eventmachine的情况。经过不断的寻求答案和尝试最终找到解决办法，在这里记录一下。

![Untitled.png]

报错就一个，找不到openssl。mac自带了一个openssl，目前是在/usr/bin/openssl。我们需要使用homebrew安装的版本（我也不知道为什么）。如果你没有安装过homebrew，请先安装homebrew，然后执行brew install openssl，如果顺利的话会安装成功。

## 解决方法

[MacOS Ventura13.3 with Bundle Install eventmachine stuck in failing loop](https://stackoverflow.com/questions/75944204/macos-ventura13-3-with-bundle-install-eventmachine-stuck-in-failing-loop)

[How to resolve fatal error: 'openssl/ssl.h' file not found issue while install eventmachine gem](https://www.youtube.com/watch?v=BVZjBjPBZX8)

要同时执行以下2条命令，我在改的过程中一直在执行第二条命令忽略了第一条，导致一直失败。。

1. bundle config build.eventmachine --with-ssl-dir=/opt/homebrew/opt/openssl@1.1
2. bundle config build.eventmachine --with-cppflags=-I/opt/homebrew/opt/openssl@1.1/include

可以通过 brew --prefix openssl 查看brew中openssl的路径，打开opt文件夹其实可以看到好几个openssl版本，我这里使用1.1是没问题的，其他的以后有机会再尝试了。

![Untitled1.png]

最后执行 bundle install 应该就可以成功了！

![Untitled2.png]

## 启动本地服务

```bash
bundle exec jekyll serve
```

访问：[http://localhost:4000/](http://localhost:4000/)

## 其他问题

**[Jekyll 运行的时候提示错误 cannot load such file -- webrick (LoadError)](https://www.cnblogs.com/huyuchengus/p/15473035.html)**

从 Ruby 3.0 开始 webrick 已经不在绑定到 Ruby 中了，请参考链接： [Ruby 3.0.0 Released](https://www.ruby-lang.org/en/news/2020/12/25/ruby-3-0-0-released/) 中的说明。

webrick 需要手动进行添加。

```bash
bundle add webrick
```