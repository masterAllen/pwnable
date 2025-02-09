# Linux Rootkit 系列一：LKM 的基础编写及隐藏

一篇很简单，但是却非常有帮助的文章，转载一下。

## 转载说明
本文作者：arciryas \
本文来源：FreeBuf.COM \
本文出处：https://www.freebuf.com/articles/system/54263.html \
作者邮箱：arciryas.yang \
发布时间：2014-12-17 05:07:20

## 前言
首先介绍最基础的 LKM 模块的编写与加载以及如何让 `lsmod` 命令无法发现我们的模块（也就是本文的内容），然后是介绍lkm rootkit中最重要的技术，系统调用挂钩，我将会给大家介绍三种不同的系统调用挂钩技术，以便于在不同的场景中选择最恰当的一种。接下来便是系统实战，使用我们之前的知识来进一步完善我们的rootkit，包括如何隐藏进程，隐藏端口，彻底隐藏lkm，以及如何向现有的系统LKM模块注射我们的代码来改造成我们自己的lkm模块。

## LKM 介绍
LKM 的全称为 Loadable Kernel Modules，中文名为可加载内核模块。主要作用是用来扩展 linux 的内核功能。LKM 的优点在于可以动态地加载到内存中，无须重新编译内核。由于 LKM 具有这样的特点，所以它经常被用于一些设备的驱动程序，例如声卡，网卡等等。当然因为其优点，也经常被骇客用于 rootkit 技术当中，至于这种技术是什么，请不要着急。

### LKM 编写
下面是一个最基本的 LKM 的实现，接下来我会对这个例子进行讲解。
```cpp
/*lkm.c*/
#include <linux/module.h>    
#include <linux/kernel.h>   
#include <linux/init.h>        
 
static int lkm_init(void) {
    printk("Arciryas:module loaded\n");
    return 0;    
}
 
static void lkm_exit(void) {
    printk("Arciryas:module removed\n");
}
 
module_init(lkm_init);
module_exit(lkm_exit);
```

这个程序并不是很复杂：其中我们的 `lkm_init()` 是初始化函数，在该模块被加载时，这个函数被内核执行，有点构造函数的感觉；与之相对应的，`lkm_init()` 是清除函数，当模块被卸载时，内核将执行该函数，有点类似析构函数的感觉。
**注意，如果一个模块未定义清除函数，则内核不允许卸载该模块。**

为什么我们的初始化与清除函数中，使用的是 `printk()` 函数，而并非是我们熟悉的 `printf()` 函数呢？注意下我们这个程序包含的头文件，在 LKM 中，是无法依赖于我们平时使用的 C 库的，模块仅仅被链接到内核，只可以调用内核所导出的函数，不存在可链接的函数库。这是内核编程与我们平时应用程序编程的不同之一。`printk()` 函数将内容记录在系统日志文件里，当然我们也可以用 `printk()` 将信息输出至控制台：
```
printk(KERN_ALERT "output messages");
```
其中 KERN_ALERT 指定了消息的优先级。

module_init 和 module_exit 是内核的特殊宏，我们需要利用这两个特殊宏告诉内核，我们所定义的初始化函数和清除函数分别是什么。

### LKM 编译装入
代码的描述就到这里，接下来我们需要对我们的 LKM 程序进行编译，下面是编译所需的 Makefile：
```
obj-m   := lkm.o
 
KDIR    := /lib/modules/$(shell uname -r)/build
PWD    := $(shell pwd)
 
default:
$(MAKE) -C $(KDIR) SUBDIRS=$(PWD) modules
```
接下来我们键入 `make` 命令开始编译，除去编译的中间产物外，我们仅仅需要的是 lkm.ko。

装载LKM我们需要insmod命令。键入 `insmod lkm.ko` 回车，这时你会发现什么都没有发生，没有关系，这是因为我们并没有对于我们的消息指定 KERN_ALERT 优先级,此时 `printk` 将消息传输到了系统日志 syslog 中，我们可以在 `/var/log/messages` 中查看，当然不同的发行版以及不同的 syslog 配置中，该文件的路径不同。

我们可以 `cat /var/log/messages` 或者利用 `dmesg` 命令查看 `printk` 输出的消息，如下图所示：

![捕获2.PNG](source/20201120001.png)

为了方便起见我只显示了最后一条信息，也就是我们 LKM 中初始化函数所输出的信息。

我们再输入 `lsmod` 命令查看我们的模块。`lsmod` 命令的作用是显示已载入系统的模块。如下图：

![捕获3.PNG](source/20201120002.png)

其中 lkm 当然是我们的模块名称，676 则代表的是模块大小，0 表示模块的被使用次数。有兴趣的同学可以自己试试 `lsmod` 命令查看下系统所加载的其他模块。

### LKM 编译卸载
OK，现在我们可以对我们的 LKM 进行卸载了，卸载 LKM 的命令是 `rmmod`。键入 `rmmod lkm.ko` 后，我们再查看下系统日志：

![捕获4.PNG](source/20201120003.png)

可以看出清除函数中的信息也成功输出，这时再试试 `lsmod` 命令，你会发现我们的模块在其中不复存在了。

## LKM 隐藏

如果我们既不想让 `dmesg` 也不想让 `lsmod` 这两个命令察觉到我们的模块呢？对于 rootkit 来说，隐蔽性是非常重要的，一个 `lsmod` 命令就可以让我们的 LKM 遁形，这显然谈不上隐蔽。另外，还有 `sys` 目录中也有暴露 LKM 的地方，这又怎么隐蔽呢？

对于 `dmesg` 命令，我们只要删除掉 `printk()` 函数就好，这个函数所起的仅仅是示范作用。下面主要来谈如何规避 `lsmod` 命令和 `sys` 目录。

### 躲避 `lsmod` 命令

首先必须要知道 `lsmod` 原理。简单来说，其通过 `/proc/modules` 来获取当前系统模块信息的。而 `/proc/modules` 中的当前系统模块信息是内核利用 `struct modules` 结构体的表头遍历内核模块链表，从所有模块的 `struct module` 结构体中获取模块的相关信息来得到的。

结构体 `struct module` 在内核中代表一个内核模块。通过 `insmod` ( 触发 `init_module` 系统调用 )把自己编写的内核模块插入内核时，模块便与一个 `struct module` 结构体相关联，并成为内核的一部分。

所有的内核模块都被维护在一个全局链表中，链表头是一个全局变量 `struct module *modules`。任何一个新创建的模块，都会被加入到这个链表的头部，通过 `modules->next` 即可引用到。

为了让我们的模块在 `lsmod` 命令中的输出里消失掉，我们需要在这个链表内删除我们的模块，方式如下：
```cpp
list_del_init(&__this_module.list)
// list_del_init: include/linux/list.h
// this_module: current module
```
其中 `list_del_init` 就是链表删除某个元素的实现函数，没有什么好说的。现在将 `list_del_init(&__this_module.list)` 加入到我们的初始化函数中，保存，编译，装载模块，再输入 `lsmod`，这时你会发现，输出中我们的模块已经找不到了，我们躲避了 `lsmod` 命令！

### 躲避 `sys` 目录
除了 `lsmod` 命令和相对应的查看 `/proc/modules` 以外，我们还可以通过查看 `/sys/module/` 目录来发现现有的模块。

![捕获5.PNG](./source/20201120004.png)

在初始化函数中添加一行代码即可解决问题：
```cpp
kobject_del(&THIS_MODULE->mkobj.kobj);
```

`THIS_MODULE` 和刚才上面的 `__this_module` 都是指向当前模块。`&THIS_MODULE->mkobj.kobj` 代表的是 `struct module` 的成员 `struct module_kobject` 的一部分，结构体的定义如下：
```cpp
struct module_kobject{
      struct kobject kobj;
      struct module *mod;
};
```
其中 kobj 指的是 struct kobject 结构体，而 `kobject` 是组成设备模型的基本结构。

这时我们又要简单介绍下 sysfs 这个概念，sysfs 是一种基于 ram 的文件系统，它提供了一种用于向用户空间展现内核空间里的对象、属性和链接的方法。sysfs 与 `kobject` 层次紧密相连，它将 `kobject` 层次关系表现出来，使得用户空间可以看见这些层次关系。

通常， sysfs 是挂在在 `/sys` 目录下的，`/sys/module` 这个目录层次包含当前加载模块的信息。我们通过 `kobject_del()` 函数删除我们当前模块的 `kobject` 就可以起到在 `/sys/module` 中隐藏 LKM 的作用。

好了，这时再将 `kobject_del(&THIS_MODULE->mkobj.kobj)` 也添加在初始化函数里，保存，编译，装载模块，然后再去看看 `/sys/module`，是不是什么也看不到了？

## 结语
对于 LKM 的入门以及 LKM 的简单隐藏办法已经介绍完了，但是这只是通向 lkm rootkit 的长征路上第一步，在下次的文章中，我会介绍 lkm rootkit 编写中最为关键的技术：system call hook，也就是系统调用挂钩技术。

## 参考资料
- 关于 LKM 的编写，《linux设备驱动程序（第三版）》的第二章"构造和运行模块"里有基础的讲解。
- 关于 proc 和 sysfs 文件系统，可以参考《深入linux内核架构》中的第十章"无持久存储的文件系统"。
