---
layout:     post   				    
title:      linux deb包的制作详解(3) 		
subtitle:      -- 使用sbuild建立独立的编译环境
date:       2019-05-29 				
author:     Downeyboy 				
header-img: img/blog-post-bg.jpg	
catalog: true 					
tags:	    linux deb包制作
--- 

# linux下deb包的制作(3) -- 使用sbuild建立独立的编译环境  

## sbuild的由来
通常，在linux的使用尤其是开发中，通常会遇到一个十分棘手的问题：平台兼容性。不知道你有没有遇到过这样的情况：软件在一个平台上跑得好好的，放到另一个不同平台下就是跑不起来，或者直接编译不过。甚至有时候放到同一个平台下都运行不了。  

我相信大多数开发者都遇到过这种情况，有时候是因为不同平台的兼容性导致，有时候是因为相同平台下不同的软件版本导致。  

尤其是在嵌入式设备上，这个问题显得更为复杂，因为linux系统很可能运行在各种不同的架构上，比如X86、ARM、MIPS等等，由于这些平台架构、软件发行版和软件版本的区别，很可能出现一些问题，尤其是在交叉编译时，一不小心就掉进了坑。    

总结起来，这些问题出现的原因就是编译平台和目标平台不匹配，或者是编译平台的环境并非足够纯净。  

最理想的状态是这样：如果我们需要编译一个基于arm64架构的deb包，那么就从官方下载安装一个全新的基于arm64架构的发行版(ubuntu或debian)，这样的基础上编译出来的deb包是可以保证其在其他任何基于arm64架构的发行版上正常使用。  

实际上，这种执行方式是非常浪费时间的。于是，就有了一种轻量级的创建纯净编译环境的方法出现了，就是我们这一章节要讨论的：**sbuild构建工具集**。  


****  

## sbuild介绍
sbuild是一种官方提供的构建debian包的方式，事实上它是一系列命令集，与上一章节中提到的debuild类似，它也是一个perl脚本，当我们安装sbuild时，它通常被安装到/usr/bin/sbuild,可以使用编辑器查看其脚本实现，如果你能看懂perl脚本的话，就可以完全了解sbuild的所有细节。  

它可以为所有支持的体系结构构建二进制包，个人也可以使用它来测试软件包是否是在纯净环境下被构建，特别是这有利于确保软件包构建者没有错过任何构建依赖项。  


****  

## 创建环境

### 安装
第一步，当然是安装sbuild环境，我们可以使用下面的指令：
```
sudo apt-get install sbuild
```

****  

### 创建目标平台环境
创建一个新的chroot(纯净的目标平台)环境，换句话说，就是在当前主机上创建一个独立的debian环境以编译deb包，可以创建与主机不同的架构环境，比如在X86上创建arm32或者arm64的。  

创建环境需要用到**sbuild-createchroot**命令。  

sbuild-createchroot 调用了debootstrap，debootstrap是一个工具，主要功能是在一个debian系统的某一个目录下安装一个纯净的deb系统(可以是不同架构)，主要用来编译deb包，在纯净系统下编译出的deb包将不会存在依赖问题。  

相对于**debootstrap**，sbuild-createchroot同时会额外地做一些基础的设置工作，更适用于新手使用。  

当我们不知道一个命令怎么用时，第一步就应该想到的指令是：
```
sbuild-createchroot --help
```
输出结果为：
```
sbuild-createchroot [option\] SUITE TARGET-DIRECTORY DEBIAN-MIRROR-URI  
参数(原为英文，经博主翻译)： 
--arch=arch                 创建的目标架构
--chroot-suffix             添加创建包的后缀名，默认是-sbuild，一般使用默认值。
--resolve-deps              自动处理依赖问题，这是默认使能的选项
--no-resolve-deps           与上一个参数相反的作用
--keep-debootstrap-dir      在安装完成之后不删除/debootstrap目录
--debootstrap=debootstrap   生成deb系统工具默认使用debootstrap，一般不用修改
--include=packages          用逗号分隔的package名称，被指定的package将会被安装到新生成的系统中
--exclude=packages          移除本来应该安装在系统中的package，package之间用逗号分隔
--keyring=keyring-file      指定keyring文件，这里指gpg文件，默认使用/etc/apt/trusted.gpg，这个gpg文件主要用与检查下载文件的签名
```

可以看到，它的使用方式为：
```
sbuild-createchroot [option] SUITE TARGET-DIRECTORY DEBIAN-MIRROR-URI  
```
我们撇开选项部分，先来看看sbuild-createchroot运行时的必需参数：SUITE TARGET-DIRECTORY DEBIAN-MIRROR-URI。  


****  

#### SUITE                       
目标平台的发行版，不是版本号，而是发行版的代号，在ubuntu中有jessie、sid等等，使用

```  
lsb_release -a
```  
查看，在ubuntu 16.04.3 LES上的显示是这样的：

```  
Distributor ID:	Ubuntu
Description:	Ubuntu 16.04.3 LTS
Release:	16.04
Codename:	xenial
```  

这里的SUITE就是相应的Codename，即xenial。  

如果需要查看目标架构，我们可以使用以下指令：

```  
uname -a
```  

****  

#### TARGET-DIRECTORY  
生成的镜像放置位置，一般是放置在


```  
/srv/chroot/
```  

这一项参数的填写规则为：

```  
/srv/chroot/SUITE-ARCH-SUFFIX
```  

例如，如果要生成一个amd64架构，xenial的纯净镜像，需要这么写：

```  
/srv/chroot/xenial-amd64-sbuild(最后的后缀可以修改，但是不建议)
```  



****  

#### DEBIAN-MIRROR-URI
目标发行版的源URL，这个源是我们通常安装deb包时指定的官方源，也可能是国内镜像源，总之，常用的软件包通过这个源来进行下载安装。可以在目标主机上查看(例如：需要创建一个ubuntu16.04的纯净环境，我们可以到ubuntu16主机上查看，如果目前没有ubuntu16相应的主机，就需要到官网查debian发行版相应的源)：

```  
cat /etc/apt/sources.list
```  

然后将源的URL复制下来，在sbuild-createchroot指令中作为参数传入，在ubuntu下这个目录地址为：http://security.ubuntu.com/ubuntu。  

****  

### 创建chroot环境
所以最后如果要在ubuntu下生成一个全新的deb环境可以使用以下指令：


```  
sudo sbuild-createchroot --include=eatmydata,ccache,gnupg xenial /srv/chroot/xenial-amd64-sbuild http://security.ubuntu.com/ubuntu
```  



其中，--include指令表示新生成的环境需要追加安装的软件，以逗号分隔，这里安装了三个：

eatmydata:如果用户存储一些并不是很重要的数据时可以使用eatmydata以加快编译速度，在编译时通常就是这种情况，如果我们仅仅把这个环境用作编译的话，可以添加这个软件。  

ccache可以帮助编译时进行缓存，加快编译时间。

gnupg(gpg),是一个加密&签名软件，用作对软件包的签名。  


****  

### 结果 
这个生成过程需要一段时间，我们可以查看相应的目录下是否生成了文件：


```  
ls /srv/chroot/xenial-amd64-sbuild
```  

如果生成了这个文件，就会显示这个文件下的文件内容：


```  
bin   build        dev  home  lib64  mnt  proc  run   srv  tmp  var
boot  debootstrap  etc  lib   media  opt  root  sbin  sys  usr
```  


可以明显看出，/srv/chroot/xenial-amd64-sbuild这个目录下存在着一个完整的linux系统根目录。可以将/srv/chroot/xenial-amd64-sbuild看成是新系统的"/"目录


****  

### 删除系统
有创建就有删除，如果我们需要删除一个chroot，直接删除这个文件即可.  

```
sudo rm -r /srv/chroot/xenial-amd64-sbuild/
sudo rm /etc/schroot/chroot.d/xenial-amd64-sbuild-* /etc/sbuild/chroot/xenial-amd64-sbuild
```
  

****  

### 使用系统  
通常，我们可以将生成的系统作为一个独立的系统，我们可以使用以下的指令来进入系统的shell：
```
sudo  sbuild-shell  xenial-amd64
```     
可以在shell中进行大部分linux下的shell操作，包括读写执行文件等等。  
在上面说到，我们可以将/srv/chroot/xenial-amd64-sbuild/看做新系统的"/"目录，如果我们要与这个系统传输文件，可以在系统外直接操作/srv/chroot/xenial-amd64-sbuild/目录。  

尽管新创建的chroot可以做不少事，但是它的主要工作还是被sbuild调用以进行编译deb包的工作，其他的我们就暂时不谈。  


****  

## sbuild的配置及使用
创建完chroot(纯净的系统)，就表示完成了第一步，接下来我们对其进行相应的配置，然后让它派上用场。  

linux下软件的配置文件通常被放置在 **~/.$PACKAGErc**文件中，比如：vim的配置文件被放置在 **~/.vimrc**中。   

同样的道理，sbuild的配置文件为 **~/.sbuildrc**。  


****  

### 通用配置
如果仅仅是个人编译deb包的行为，而不是作为针对特定架构的编译服务器，你可以设置以下的配置：
```
$build_arch_all = 1;
$distribution = 'xenial';
```
**build_arch_all**变量表示默认编译平台无关的源代码包，而专用的编译服务器通常针对不同的架构。  
**distribution**变量表示设置目标发行版为"xenial",当然，也可以在命令行使用sbuild时传递一个"-d"参数对发行版进行覆盖设置，这些行为最终的效果是会影响到debian/changelog文件。  


****  

### 使用lintian
使能lintian将提高软件包的生成质量

$run_lintian = 1;
$lintian_opts = ['-i', '-I'];


****  

### 使用piuparts
使能piuparts能提高软件包质量

$run_autopkgtest = 1;
$autopkgtest_root_args = [];
$autopkgtest_opts = [ '--', 'schroot', '%r-%a-sbuild' ];


****  

### 使用autopkgtest
使能autopkgtest，在编译完成之后就会自动测试生成的包

$run_autopkgtest = 1;
$autopkgtest_root_args = [];
$autopkgtest_opts = [ '--', 'schroot', '%r-%a-sbuild' ];



****  

## 执行编译命令生成deb包

在上一章节中，我们有讲到debmake指令将源码目录进行自动构建，然后使用debuild命令对其进行打包编译，那sbuild是怎样编译deb包的呢？  

我们只需要键入命令：
```
sbuild
```

与上一章节中的debuild指令相比，sbuild指令有什么区别呢，它的步骤如下：
* 准备源码(同上章)
* 源码打包(同上章)
* 使用debmake自动生成配置文件模板(同上章)
* 根据具体项目需求修改控制文件以及rules文件(同上章)
* 对上述生成的源码再次进行打包，在debian/的父目录下进行打包，将源文件目录打包成\$PKGNAME_\$VERSION.orig.tar.gz的形式，这里的源码包与包含控制文件，即debian目录，第一步中的打包仅包含程序源文件。
* 使用sbuild指令进行打包工作(上一章为调用debuild -us -uc)。  

sbuild指令将依次调用debootstrap进入新创建的chroot环境、**dpkg-buildpackage -us -uc**编译deb包(同样将使用到fakeroot)、根据选项调用lintian。  
   
sbuild与debuild在运行上不同的是：
* sbuild进入纯净环境编译，很小概率出现包的依赖问题。  
* sbuild在编译时即时的下载某些依赖软件，需要更长的时间，通常我们可以使用缓存机制来缓解这个问题。  


****  

### 获取deb包
在sbuild执行完成之后，我们就可以在debian/的上级目录看到一系列的文件包，其中后缀为.deb的软件包就是我们需要的deb包，可以对其进行测试。  


## 获取源码
从github[获取源码](https://github.com/linux-downey/sbuild_debuild_example)自己尝试编译deb包(编译方式请参考github仓库readme)，深入理解sbuild使用。  


更多详细资料请参考[官方文档](https://wiki.debian.org/sbuild)  


***  
***  

好了，关于《linux下deb包的制作(3) -- 使用sbuild建立独立的编译环境 》  的讨论就到此为止啦，如果朋友们对于这个有什么疑问或者发现有文章中有什么错误，欢迎留言

***原创博客，转载请注明出处！***

祝各位早日实现项目丛中过，bug不沾身.












