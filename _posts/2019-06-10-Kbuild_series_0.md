---
layout:     post   				    
title:      linux Kbuild详解系列 		
subtitle:      -- 内核的编译操作 
date:       2019-06-10 				
author:     Downeyboy 				
header-img: img/blog-post-bg.jpg	
catalog: true 					
tags:	    linux Kbuild
---

# linux Kbuild详解系列(0) - 内核的编译操作  


## 总览
本博客为系列作品，将会详细介绍从linux内核的编译操作到linux内核的编译安装的流程，也就是linux的Kbuild系统，在博主看来，整个Kbuild的系统的复杂性是比较高的，建议你看这个博客之前先重温Makefile的知识点。   

当然，对Makefile的掌握要求并不仅仅是简单地对着网上的模板能进行对应的增删改查，而需要对Makefile的语法有一个整体的掌握，对于这一部分，可以参考博主之前的系列博客：[深入理解Makefile系列](http://www.downeyboy.com/2019/05/14/makefile_series_sumary/)。   


相信我，如果你对linux 内核的编译安装机制有了一定的了解，对你之后的linux配置、调试、开发都将起到至关重要的作用。   

好了，废话不多说，我们先开始第一章的内容：内核的编译操作。  

*****   

## 内核的编译  
之所以首先讲内核的编译，是希望能从应用开始，带着疑问一步步深入，然后解析整体框架，如果一上来就是整个框架的解析，那么这将是一件非常枯燥无聊的事情，但是，学习本来应该是一件有趣的事情，对吧。  

****

### 第一步 ： 下载源码
想编译源码当然第一步是下载它，目前linux的代码被托管到github上，这也是最主流的代码管理方式，直接在github的搜索栏输入linux，就可以看到torvalds/linux。   

没错，作者就是大名鼎鼎的linus torvalds，这是linux的主线代码，值得注意的是，
对于嵌入式linux而言，大部分嵌入式板厂的linux发行版本都和主线代码有区别，具体的下载方式那就因厂商而异了。  

下载：  

```
git clone https://github.com/torvalds/linux.git
```

****  

### 第二步 ： 配置
下载下来源代码之后，进入到源代码根目录，我们需要对源代码进行配置。  

****  

#### 为什么要配置
我希望你看到第二步的标题就能想到这个问题，为什么要配置？    

linux实现的各种功能是以模块的形式存在的，比如存储、时钟、外设或者平台相关的功能选择等等，对于系统非必须的模块，linux内核支持动态地加(卸)载，用户可以指定将模块编译到内核镜像中还是编译成外部模块。通过这些配置，我们可以非常方便地对linux内核大小进行裁剪，这对于很多内存受限的嵌入式设备是非常重要的。  

****

####  怎么配置
通常，linux的配置方式主要是对各种模块进行选择，其中包括对平台的选择，大部分模块都有三种选择：
* Y - 将该模块编译进内核
* M - 将模块编译成外部模块
* N - 不编译该模块

而配置的方式有很多种，都是在源代码根目录下敲以下相应的命令：
* make config ： 这是linux内核配置的祖师爷了，对于每个模块，在终端将逐个询问你的配置意向：Y/N/M，这个配置方式基本不用了，太过于麻烦。  
* make xxx_defconfig : 按照对应平台的默认配置方式来配置，可以在 \$(SRC_ROOT)/arch/\$(ARCH)/configs 中找到对应的 xxx_def_config 。  
    比如x86平台，执行 make i386_defconfig 就是按照 arch/x86/configs/i386_defconfig  这个文件进行配置。  
* make menuconfig：这种配置方式是最常用的，使用图形界面的方式对内核模块进行配置，各内核模块基于树状结构，可以很方便地使用方向键和选择键进行配置。
* make kconfig/gconfig：这两种配置方式也是基于图形界面，但是都依赖于其他的开发环境，出场率不高。
* make oldconfig：这种配置方式可能出现在内核版本升级的情况下使用，即将旧工程的.config文件copy到新工程中，然后直接使用旧的配置。当然，新版本一般对于旧版本来说总会有一些更新，它也会根据kconfig的配置提示新版本中的更新部分。  

如果不出意外，配置完成的结果就是在源代码根目录下生成.config文件，这个.config文件中逐一列举了用户对内核模块的配置，这些配置选项将被提供给编译器进行内核的源码编译工作。    

****

#### 架构选择
事实上，在嵌入式开发时，通常会涉及到交叉编译，通常情况下，进行开发工作的目标开发板资源有限，且主频不高，如果直接在目标开发板上进行编译工作，效率是非常低的，所以我们需要采取另一种方案：交叉编译。   
(**注：以下统一将编译使用的PC机称为编译主机，目标开发板称为开发主机**)

即在功能强大的PC机上进行编译工作，然后将编译完成的资源如镜像、模块再 copy 到开发主机上运行，这样就可以提高编译效率。  

一般情况下，编译主机通常是X86的架构，而开发主机则各有不同，自然地，编译出来的固件需要运行在开发主机上而不是编译主机上，内核架构的不一样导致编译器也需要对开发主机进行适配。(当然，如果编译主机和开发主机是同一架构，就不存在这个问题)。  

linux的跨平台支持做得非常完善，当进行交叉编译时，我们需要做的就是两件事：
1. 在编译时指定开发主机的架构平台
2. 在编译时指定交叉编译器

***  

### 第三步 ：编译
#### 编译内核
如果编译主机和开发主机是同一个，可能你仅仅是想给自己的ubuntu升级内核，事情就非常简单了，直接键入：
```  
make
```
当不指定目标平台和交叉编译工具链时，默认情况就是将本机作为目标平台。  


但是，如果是进行交叉编译，就需要多做一些配置了，假设在开发主机为arm平台，就需要这样进行编译：  
```
make ARCH=arm CROSS_COMPILE=$(CROSS_COMPILE_DIR)/arm-linux-eabi-
```
交叉编译器不需要指定对应的gcc，只需要指定相应平台即可，在makefile中将会自动添加上gcc字段，即arm-linux-eabi-gcc。  

编译完成之后生成镜像文件：vimlinux。  

但是，通常 vmlinux 并不会被直接使用，在不同的架构中对其有不同的处理，处理完之后使用的镜像可能是以下的几种：vmlinuz、Image、zImage、bzImage等等。  

对于这几种镜像的差别，将会在后续的博客中详解。  

***  

#### 编译模块
值得注意的是，在配置阶段我们就将模块人为地分为两类：编译进内核和作为外部模块，顾名思义，编译进内核的模块直接存在于镜像中，在开发主机上运行时就存在于系统中。  

而外部模块则不然，需要对其进行手动加载才能使用它提供的功能，默认的后缀为 .ko 。   

在新版本的内核中，编译内核时，会同时编译外部模块。  

***  

### 第四步 ：安装

#### 安装内核
对内核进行编译的目的自然是要使用它，那么我们就需要把编译生成的镜像和模块进行安装。  

根据平台的不同，编译生成的镜像也是各有差别，主要是格式上的区分，如：vmlinuz、Image、zImage、bzImage等等。   

相同的是，这些镜像文件都是由vmlinux经过处理得到，不同的是对其进行处理的方式不同，根据开发平台不同的特性而做一些相应定制。  


通常，在嵌入式开发平台上，安装新镜像的方式就是使用编译生成的最终镜像文件替换原来的文件，同时可能还需要替换 System.map(内核符号)或者.dtb(设备树文件)等等，这一部分根据平台的差异而有些许不同。  

***  

#### 安装模块
如果编译平台和目标平台一致，模块的安装就非常简单，只需要使用下面的指令：
```
make module_install
```
模块就会被安装到系统中。  

如果目标开发平台和编译主机不是同一个，需要将模块安装到开发平台，那自然是不能直接进行安装，我们需要将所有模块安装到一个目录中，再将目录copy到开发平台相应的目录下，使用以下指令安装模块到目录：
```
sudo make  modules_install  INSTALL_MOD_PATH=module_dir
```
module_dir 是安装的目标目录。  

通常，在主机上，模块被放置在/lib/modules下，我们就可以将上一步中的目录copy到这个目录下。    

***

## 小结
内核编译及安装的几大步骤：
1、内核配置
2、编译内核和模块
3、安装内核和模块

同时，在嵌入式的开发中，通常需要考虑交叉编译的问题。   

***  
***  
***  



好了，关于 linux内核Kbuild系统-内核的编译操作 的讨论就到此为止啦，如果朋友们对于这个有什么疑问或者发现有文章中有什么错误，欢迎留言

***原创博客，转载请注明出处！***

祝各位早日实现项目丛中过，bug不沾身.



