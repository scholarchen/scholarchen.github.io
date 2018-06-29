---
layout: post
title: openwrt移植osgi
category: openwrt
tags: osgi java felix
---

# osgi概念理解
## osgi介绍
OSGi是一个基于Java的服务平台规范，其目标是被需要长时间运行、动态更新、对运行环境破坏最小化的系统所使用。有许多公司（包括Eclipse IDE，它是第一个采用OSGi技术的重要项目）已经使用OSGi去创建其微内核和插件架构，以允许在运行时刻获得好的模块化和动态组装特性。
通俗的说就是有一套规范，规定可以把服务写成一个个小的模块，然后这些模块可以在osgi这个规范里动态增删改而不影响其他模块。提供了一个完整的模块化运行环境

## osgi缺点
增加开发难度，需要开发人员维护模块间依赖关系
增加测试调试难度，只有在模块发布运行后，才能看到模块间依赖是否正确，不能再编译期间检测
增加额外的运行环境，所有程序需要统一在osgi容器里运行
## osgi优点
项目模块化，不用发布整个项目，只需要更新相应模块
模块可在运行时发布，不会影响其他模块
# java虚拟机选择
# osgi框架选择
目前OSGi容器主要有Knopflerfish, Apache Felix, Equinox, spring DM
# osgi集成openwrt
## jamvm+classpath移植到linux
官网下载安装包
[jamvm](http://sourceforge.net/projects/jamvm)
[classpath](http://www.gnu.org/software/classpath/)

# helloworld 测试


# debug
classpath 编译
jamvm最新版本2.00编译
