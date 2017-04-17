---
layout: post
title: github搭建博客
category: General
tags: github,jekyll,blog
---
## 1、github创建仓库
    github上注册，然后Create a New Repository，并且填写相关信息
	这里主要就是讲工程的名字配置为 name.github.io,然后上传相应页面。这里的name必须为github的账户名。然后通过模板创建。如果有域名，可以设置将博客名字映射为域名，就可以通过域名访问了

## 2、配置git
####   1）本地配置ssh key
    ssh-keygen -t rsa -C "your_email@youremail.com"  
    然后会生成.ssh文件夹，进去打开id_rsa.pub，复制key到github中account settings中ssh keys
    验证是否成功输入
    ssh -T git@github.com 

#### 2）本地代码上传

    git config --global user.name "your name" 
    git config --global user.email "your_email@youremail.com"
    git remote add origin git@github.com:yourName/yourRepo.git

```
  git remote add origin git@github.com:schloarchen/scholarchen.github.io
  git add *
  git commit -m "base blog web"
  git push origin master //这里可能出错，是远程上代码与本地冲突，可以通过 下面命令合并
  git fetch origin  
  git merge origin/master  
```

## 3、本地搭建jekyll网站工具

####    1）安装Ruby

    下载本地对应版本ruby: http://rubyinstaller.org/downloads/
    我下载Ruby 2.2.6 (x64)，然后一步一步安装，需要选中添加环境变量，或者后面手动添加到环境变量path里面
    控制面板-系统和安全-系统-高级系统设置-环境变量-PATH,在前面或末尾加上刚刚的安装路径C:\Ruby22-x64\bin;
    验证：在cmd中查看ruby -v

####    2）安装DevKit

    下载DevKit: http://rubyinstaller.org/downloads/
    我下载DevKit-mingw64-64-4.7.2-20130224-1432-sfx.exe
    将devkit安装到ruby中去：
    进入安装目录，cd c:/devkit 
    ruby dk.rb init //生成一个config.yml
    修改config.yml 最后一行为刚刚ruby的路径：- C:/Ruby22-x64
    ruby dk.rb review
    ruby dk.rb install

#### 3) 安装jekyll
    gem install jekyll
    这里可能会有部分依赖没有安装，安装好就可以了。如果报错 “ invalid byte sequence in UTF-8” ，则需要将文件修改为utf8格式就好了

#### 4）测试使用网站
	安装好后，通过jekyll serve命令运行服务， 可以看到 监听了本地4000端口
	Server address: http://127.0.0.1:4000/
	然后通过浏览器访问
  