---
layout:     post   				    
title:      linux automake 详解(0) 		
subtitle:      -- 使用示例	 
date:       2019-05-25 				
author:     Downeyboy 				
header-img: img/blog-post-bg.jpg	
catalog: true 					
tags:	    linux automake
---

# linux automake 详解(0)  - 使用示例	
通常，针对小型工程的编译时，仅仅在命令行进行编译或者是自己写个对应的makefile使用make工具进行编译。  

但是，这种做法在面临中大型工程的时候就有些捉襟见肘了。对于大型工程而言，编写makefile的工作量将非常大，而且不能保证makefile在兼容性、扩展性等等。  

不过linux下的一切难题都难不倒分布在各地的linux维护者们，各种平台、针对各种语言的开源构建工具相继出现。其中，在C语言的构件上值得称道的有cmake，其拥有语法简洁易用的特性。  

但是我们今天要讨论的并不是cmake这一款自动构建工具，而是另一个血统更为纯正的构建工具：GNU Autotools。没错，看到GNU开头的三个大字，相比你就知道了这是GNU官方组织推出的构建工具，这是一款绝对成熟的构建工具，而且几乎不用担心平台兼容问题。  


****  

## 为什么选择autotools
autotoos工具包的目标其实是根据现有源码自动化地生成makefile，以便用户可以方便地进行编译和安装。  

通常，我们会使用autotools生成的配置文件与源码一起打包，做成一个跨平台的源码包供用户进行安装。  

使用autotools更多的是因为它的官方背景，相比于cmake，它的操作相对较为复杂，但是复杂的同时也意味着灵活，同时所有的发行版都支持autotoos。  

所以在制作需要发布或者代表官方的源码包时，我们有必要使用更为标准的autotools。  


****  

## 自动构建工具的作用
在上文中提到的自动构建工具到底是什么。有些朋友对于这个概念并不是太熟悉，我们就来看看这个构建工具到底做了什么：
* 可配置地提取工程中的源码，为源码建立编译规则
* 生成配置文件，在一台新机器上进行编译时可以根据目标机器的架构、平台生成对应的makefile，这也是实现平台无关性的关键。  
* 生成的makefile支持make install、clean等常用编译选项，覆盖大多数基本需求。  


****  

## autools介绍
autotools实际上包含一系列的工具，主要的包括有autoscan、autoconf、autolocal等工具，在制作源码包的过程中这些工具会被分别调用以实现相应的功能。  

autotools的安装也是非常简单的，我们可以直接使用apt-get指令安装:

```
sudo apt-get install autotools-dev
```

简单介绍autotools之后，我们先来看一个简单的autotools使用示例，从示例入手，深入地解析autotools的使用。    

****  

## autotools使用示例


### 创建文件并编辑
我们需要创建以下五个文件：


```  
src/main.c src/Makefile.am README Makefile.am configure.ac 
```  

然后分别编辑上述的五个文件：  

#### src/main.c

源码包的源代码部分


```  
#include <config.h>
#include <stdio.h>

int main (void)
{
    puts ("Hello World!");
    puts ("This is " PACKAGE_STRING ".");
    return 0;
}
```  

***  

#### README

包含整个包的描述信息


```  
This is a demonstration package for GNU Automake.
Type 'info Automake' to read the Automake manual.
```  


***   

#### Makefile.am： 

Makefile的配置信息，提供描述信息给automake工具去生成Makefile。  


```  
bin_PROGRAMS = hello
hello_SOURCES = main.c
```  

***  

#### src/Makefile.am:

生成src目录下的Makefile


```  
SUBDIRS = src
dist_doc_DATA = README
```  

***  

#### configure.ac:  

configure文件的配置信息，同样的，这些信息将被提供给autotools工具以生成configure文件和Makefile文件。  


```  
AC_INIT([amhello], [1.0], [bug-automake@gnu.org])
AM_INIT_AUTOMAKE([-Wall -Werror foreign])
AC_PROG_CC
AC_CONFIG_HEADERS([config.h])
AC_CONFIG_FILES([
Makefile
src/Makefile
])
AC_OUTPUT
```  

****  

### 创建公共文件
在新版本中的autotools中，除了上述源文件，还要求以下几个文件的存在：
```
AUTHORS   COPYING   ChangeLog   NEWS
```
AUTHORS包含作者信息。    

COPYING中包含版权信息。   

ChangeLog包含变更的版本信息，通常这个文件对于包的维护是至关重要的。   

NEWS包含一些新发布的包信息。  

所以，我们通过以下指令手动创建这些文件,然后填入相应信息(可选，空文件可以，但是文件必须存在)：
```
touch AUTHORS COPYING ChangeLog NEWS
```

****  

### 调用autotools指令
创建并配置完上述五个文件和生成了一些控制文件，我们就可以开始生成源码包的工作了。  

首先，执行下面的指令生成一些必要的文件：


```  
autoreconf --install
```  

将会输出以下调试信息：


```  
configure.ac: installing './install-sh'
configure.ac: installing './missing'
configure.ac: installing './compile'
src/Makefile.am: installing './depcomp'
```  

这条指令将会创建上述log信息中出现的四个文件，然后执行一系列autotools工具集中的指令，以正确的顺序调用autotools中的各个程序，比如：autoconf，automake等等。  

到这里，整个hello_world版本源码包基本上就生成了，如果我们需要将其发布，只需要进行打包即可。  

当用户修改了configure.ac或者Makefile.ac文件时，不需要再重复执行**autoreconf**指令，只需要再次执行**make**即可，所有修改相关的部分都将自动更新。  


****  

### 源码包的使用
如果你曾有从源码安装程序的经验，就知道源码安装三部曲：  
* ./configure : 根据目标平台生成Makefile
* make ： 对源码进行编译工作，生成开发者指定的最终的目标文件，通常是共享库、可执行文件等，这个示例中是生成了src/hello可执行文件，可直接运行。  
* sudo make install：将生成的共享库、可执行文件copy到系统目录。默认将其copy到/usr/local/目录下对应的文件夹中,需要root权限。  


在安装完成之后，就可以在终端的任何目录执行编译源码生成的可执行文件：


```  
hello
```  

或者在make完成之后，无需安装，执行相应程序：


```  
src/hello
```  

输出结果为：


```  
Hello World!
This is amhello 1.0.
```  



****  

## 小结
从上述的过程来看，autotools编译生成源码包的操作是非常简单的，只需要创建几个对应的文件并对其进行配置即可，事实上，操作的流程是比较简单的，主要的难点在于配置文件的语法以及实际项目的复杂性。  

在下一章节，我们将深入autotools中，展现文件配置的细节，知其然更要知其所以然。  

***  
***  
***  


更多详细资料可以参考：[automake 官方文档](https://www.gnu.org/software/automake/manual/automake.html)


***  

好了，关于《linux automake 详解(0)  - 使用示例》  的讨论就到此为止啦，如果朋友们对于这个有什么疑问或者发现有文章中有什么错误，欢迎留言

***原创博客，转载请注明出处！***

祝各位早日实现项目丛中过，bug不沾身.




