---
layout:     post   				    
title:      linux Kbuild详解系列 		
subtitle:      -- 开篇 
date:       2019-06-10 				
author:     Downeyboy 				
header-img: img/blog-post-bg.jpg	
catalog: true 					
tags:	    linux Kbuild
---

# 内核Kbuild详解系列-开篇

## 前言  
对于linux的学习而言，我想大部分人遇到的第一个坎就是linux的内核编译了吧，想当初我从windows系统刚开始转而学习linux开发的时候，编译linux的内核就是特别让我兴奋的一件事，觉得自己进入了一个自由的世界：连操作系统源代码都是自己编译的，以后我想怎么玩就怎么玩了。  

随着学习的深入，我逐渐发现了一个linux开发中一个非常残酷的事实，它叫做自由的代价：如果你只是跟着教程敲指令而不知道自己究竟在做什么，linux总是会百般地刁难你，直到你弄清楚原理或者是放弃。   

原来，自由只是对强者而言，而灵活，也意味着复杂。   

既然无路可退，就只能勇往直前。  

## 写在前面的话
本系列博客为内核 Kbuild 系统，包含两个部分：
* Kbuild 系统的使用，通俗来说也就是 linux 内核源码的编译，先了解其基本的使用
* 探究 Kbuild 编译内核源码背后的原理，旨在深入到 Kbuild 系统中，了解 linux 内核编译的背后细节与实现

对于 Kbuild 系统的学习，一方面是为了解惑，知其然而不知其所以然总是一件痛苦的事情，我们知道如何敲一个命令，然后等待返回结果，却不知道这中间发生了什么，长此以往，我们只会成为电脑旁的流水线工人，工作中为了效率尚且可以理解，但是我们应该更多地对事情背后的东西保持强烈的好奇心。  

话说回来，如果你了解了 Kbuild 系统，那么对 linux 的裁剪和移植将会有非常大的帮助。  

另一方面，也是更值得关注的一方面：就是了解系统设计背后的思想，软件的发展日新月异，虽然底层软件框架的更新速度较应用层框架的更新速度慢了很多，但是，我们不应该止步于了解应该怎么做，而是更多地去了解为什么这么做，只有学到内功心法，才能灵活地运用各种招式。  

甚至，到某一天，我们可以为国产操作系统设计它的框架，毕竟，梦也是需要做的。      

## 本系列博客规范

阅读本系列博客之前需要有一定的阅读 Makefile 的基础，了解 Makefile 可以参考博主的另一系列博客：
[Makefile详解系列](http://www.downeyboy.com/2019/05/14/makefile_series_sumary/)

涉及到平台与内核版本的部分，统一使用 arm 平台，与内核版本 5.2.rc4.

同时，本博客研究使用主线内核，即发布在 github 上的内核版本，对于嵌入式开发而言，通常各厂商都会在主线内核版本上有些许修改，相对饮的，Makefile 也会有一些小变化，不过万变不离其宗，了解了主线内核再切换到各发行版，也是非常简单的一件事。  

Makefile 中的一些概念：

**模块**：指内核中的功能模块，通常对应内核源码下的文件夹，比如 网络模块、声卡模块 就对应源码中的 net/ sound/，不一定是根目录下的文件夹才是模块，drivers/ 下也有非常多的驱动模块。  

**外部模块、可加载模块**：主要指以 .ko 结尾的模块，支持在内核运行时动态加载的模块。  

**內建模块**：编译进镜像的模块

**top makefile**:即顶层 Makefile，特指源码根目录下的 Makefile。 

****

## 分篇博客索引

为了各位的查找方便，以下是整个系列博客的链接与描述：


### 应用部分
前面 3 篇博客是关于 Kbuild 系统的使用的，通俗来说，就是怎么使用 Kbuild 系统去编译内核和外部模块，从应用入手，先了解 Kbuild 怎么用，然后再深究背后的原理。  

他们分别是：

[linux Kbuild详解系列(0) - 内核的编译操作](http://www.downeyboy.com/2019/06/11/Kbuild_series_0/)    
主要讲解了linux 内核源码最基本的编译安装操作，这是使用 linux 的第一步。  

***  

[linux Kbuild 详解系列(1)-外部模块的编译](http://www.downeyboy.com/2019/06/12/Kbuild_series_1/)  
主要讲解了如果编译一个外部模块，对应嵌入式linux驱动开发而言，这是非常常用的操作，该文章详细讲解了如何编写 Makefile 编译外部模块，同时讲解相应编译的原理

***  

[内核Kbuild详解系列(2) - 将第三方模块编译进内核](http://www.downeyboy.com/2019/06/13/Kbuild_series_2/)  
主要讲解了如何将一个模块编译进内核，这个过程其实和外部模块的编译有很大区别，同时讲解了 Kbuild 系统中负责模块功能配置的 Kconfig 语法。

***  

### Kbuild 框架
中间的几篇博客主要讲解了一些 Kbuild 的预备知识，说是 Kbuild 的预备知识，其实倒不如说是研究内核编译背后原理的基础掌握，只有更好地掌握了这些预备之后，才能深入到内核 Makefile 以及各种脚本的研究中。  

[linux Kbuild详解系列(3) - Kbuild系统框架概览](http://www.downeyboy.com/2019/06/14/Kbuild_series_3/)  
这篇文章主要是翻译自官方文档，对 Kbuild 的框架做一个大概的介绍。  

***  

[linux Kbuild详解系列(4)-kbuild源码实现的框架](http://www.downeyboy.com/2019/06/15/Kbuild_series_4/)  
主要讲解了 Kbuild 框架中对应的源码实现，介绍整个内核编译时各个文件的作用以及生成文件的变化。  

***  


[linux Kbuild详解系列(5)-scripts/KBuild.include文件详解](http://www.downeyboy.com/2019/06/16/Kbuild_series_5/)    
主要讲解了 scripts/KBuild.include 脚本中重要函数的实现，这些函数会在整个内核编译过程中频繁出现。  

***  

[linux Kbuild详解系列(6)-scripts/Makefile.lib 文件详解](http://www.downeyboy.com/2019/06/17/Kbuild_series_6/)  
主要讲解了 scripts/Makefile.lib 文件的功能以及实现，scripts/Makefile.lib 文件在整个内核编译的过程非常重要，负责过滤及整理所有的待编译目标文件。

***  

[linux Kbuild详解系列(7)-scripts/Makefile.build文件详解](http://www.downeyboy.com/2019/06/18/Kbuild_series_7/)  
主要讲解了 scripts/Makefile.build 文件的功能以及实现，与 scripts/KBuild.include 和 scripts/Makefile.lib 文件一起，这三个文件组成了整个 Kbuild 的核心后台处理部分，参与整个内核编译的过程。  
其中，scripts/KBuild.include 负责提供通用函数以及变量的定义， scripts/Makefile.lib 负责整理待编译文件，将其赋值到对应变量，scripts/Makefile.build 负责编译由 scripts/Makefile.lib 处理完的目标文件，并完成递归地编译工作。  

****  

[linux Kbuild详解系列(8)-Kbuild中其他通用函数与变量](http://www.downeyboy.com/2019/06/19/Kbuild_series_8/)  
同样是预备工作，如果需要看懂 top Makefil，同时需要了解 Kbuild 中一些常用的变量以及操作.

***  

### 真正深入到 top Makfeile 的编译

[内核Kbuild详解系列(9)-top_Makefile的执行框架](http://www.downeyboy.com/2019/06/20/Kbuild_series_9/)  
咋一看，top Makefile 将近2000 行，这么复杂的 Makefile 博主也是头一次见。  

但是，他强由他强，清风拂山岗。我们只需要抓住其中的主线部分，也就是抓住那几个线头，然后顺着绳子往下缕就可以了，这篇博客讲解了整个 top Makefile 处理的几种情况，根据这些处理的情况再进行各个击破，看起来也并不是那么难。  

***  

[内核Kbuild详解系列(10)-Kbuild中make_config的实现](http://www.downeyboy.com/2019/06/22/Kbuild_series_10/)  
这是我们常用的内核编译的第一步，内核的配置，想不想知道内核配置背后实现的原理是怎么样的？就看这篇博客吧。  

***  

[内核Kbuild详解系列(11)-Kbuild中vmlinux以及镜像的生成(0)](http://www.downeyboy.com/2019/06/23/Kbuild_series_11/)

[linux Kbuild 详解系列(12)-Kbuild中vmlinux以及镜像的生成(1)](http://www.downeyboy.com/2019/06/24/Kbuild_series_12/)
终于迎来了最后的目标，我们研究 Kbuild 系统的目的就是为了弄清楚内核镜像是怎么编译出来的，将在这两篇博客中得到解答。

***  

## 小结
整个系列的文章跨度较大，从应用入手，一步步深入内核背后的细节。  
除了应用部分的前三篇，后面的 10 篇建议反复对照阅读，会有更好的效果。  





