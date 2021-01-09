---
title: git学习网址
date: 2020-12-19
categories: 
 - 技术/git 
tags:
 - git
 - GitHub
---

# git 出现 everything up-to-date解决方法

#### 现象

明明已经更改了本地代码，但是git push的时候一直提示everything up-to-date，创建了新分支，依然push了origin master的版本。

#### 解决

方法特别简单，实际就是在push之前必须要写commit。

>git add .   
git commit -m "msg"  
git push  
