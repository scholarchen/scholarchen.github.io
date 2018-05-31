---
layout: post
title: 编译运行路劲搜索顺序
category: compile
tags: 编译顺序
---

## 1、运行时，动态库的装载依赖于ld-linux.so.6的实现，它查找共享库的顺序如下：

（1）ld-linux.so.6在可执行的目标文件中被指定，可用readelf命令查看

（2）ld-linux.so.6缺省在/usr/lib和lib中搜索；当glibc安装到/usr/local下时，它查找/usr/local /lib

（3）LD_LIBRARY_PATH环境变量中所设定的路径

（4）/etc/ld.so.conf（或/usr/local/etc/ld.so.conf）中所指定的路径，由ldconfig生成二进制的 ld.so.cache中

## 2、编译时，搜索库的路径顺序如下：
（1）ld-linux.so.6由gcc的spec文件中所设定

（2）gcc --print-search-dirs所打印出的路径，主要是libgcc_s.so等库。可以通过GCC_EXEC_PREFIX来设定

（3）编译的命令行中指定的-L/usr/local/lib 然后LIBRARY_PATH环境变量中所设定的路径

（2）binutils中的ld所设定的缺省搜索路径顺序，编译binutils时指定。（可以通过“ld --verbose | grep SEARCH”来查看）

## 3、二进制程序的搜索路径顺序为PATH环境变量中所设定。
一般/usr/local/bin高于/usr/bin

## 4、编译时的头文件的搜索路径顺序，与library的查找顺序类似。
（1）gcc 从 -I参数寻找  
（2）gcc的环境变量 C_INCLUDE_PATH,CPLUS_INCLUDE_PATH,OBJC_INCLUDE_PATH  
（3）内定目录  
可以通过下面程序查看
```
echo 'main(){}'|./bin/arm-ugw-linux-uclibcgnueabi-gcc -E -v -
```
路径如下
```
#include "..." search starts here:
#include <...> search starts here:
 /projects/hnd/tools/linux/hndtools-arm-linux-2.6.36-uclibc-4.9.3/usr/lib/gcc/arm-ugw-linux-uclibcgnueabi/4.9.3/include
 /projects/hnd/tools/linux/hndtools-arm-linux-2.6.36-uclibc-4.9.3/usr/lib/gcc/arm-ugw-linux-uclibcgnueabi/4.9.3/include-fixed
 /projects/hnd/tools/linux/hndtools-arm-linux-2.6.36-uclibc-4.9.3/usr/lib/gcc/arm-ugw-linux-uclibcgnueabi/4.9.3/../../../../arm-ugw-linux-uclibcgnueabi/include
 /projects/hnd/tools/linux/hndtools-arm-linux-2.6.36-uclibc-4.9.3/usr/arm-ugw-linux-uclibcgnueabi/sysroot/usr/include
End of search list.
```
一般/usr/local/include高于/usr/include