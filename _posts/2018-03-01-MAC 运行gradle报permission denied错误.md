---
layout:     post
title:      MAC 运行gradle报permission denied错误
subtitle:   运行gradle报permission denied错误
date:       2018-03-01
author:     PXF
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - gradle
    - Android
---
# MAC 运行gradle报permission denied错误

命令行中输入：./gradlew xxxx  

出现下错误：

**zsh: permission denied: ./gradlew**

解决方式：

先输入：**chmod +x gradlew**，再输入你的命令行./gradlew xxxx 即可