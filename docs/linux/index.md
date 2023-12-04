# 快速入门

Linux知识体系太过庞大，从入门的角度来讲，掌握最基本的命令，熟悉Linux环境下C编程就可以了。

首先，你需要一个安装了ubuntu或者其他发行版的虚拟机，或者你可以选择云服务器。我更加推荐你用云服务器的方式来学习Linux，因为：

1. 云服务器没有桌面界面，这会逼迫你只能通过命令行的方式操作Linux。
2. 花了钱的东西你才会心疼，你才会投注精力去学习。
3. 云服务器只需要你电脑能联网就可以操作，而虚拟机占用电脑资源大。
4. 云服务器对于网络环境做了适配，并且各大服务器提供商提供了更加便捷的操作环境（一键配置环境），等同于租房拎包入住，非常方便。

当你购买了某个公司的云服务器之后，你可以通过ssh的方式连接上服务器，在vscode下载ssh插件可以进行远程开发，就和本地开发一样方便。具体如何操作请参考网上教程。

当你进入Linux环境之后，你需要使用sudo apt update&&sudo apt upgrade命令来更新你的软件包。你可以使用sudo apt install [pkg_name]，来安装你需要的软件包。如果你是在自己的虚拟机上操作，在更新之前你还需要更换镜像源为国内镜像源。

为了能够在命令行下操作，你必须要熟练掌握以下几个命令：

- ls
- cd
- pwd
- rm
- mkdir
- touch
- cat
- cp
- mv
- tree
- ps
- grep
- find

同时你还需要了解管道和重定向的概念，为了能够快速打开文件进行编辑，你至少得掌握一种文本编辑器，比如VIM。要想熟练使用VIM，没有个几年的积累是不可能的，对于新手来讲，你至少得知道VIM有几种工作模式，分别是用来干什么的。你得知道如何进入编辑模式，如何进入命令行模式保存和退出文件。其他的操作你可以在日后的使用中逐渐深入。

请抛弃点击RUN按钮编译运行源代码的方式，而转而使用命令行方式编译你的源代码。你需要了解gcc的常用参数，比如-o是生成目标文件，-g是生成调试信息，-I是包含头文件路径等。

当你的源文件多起来，并且项目工程逐渐复杂起来时，每次敲一长串gcc命令会显得费时又费力。此时你需要Makefile文件来组织你的工程，为了编写Makefile文件你需要了解基本的Makefile语法。当然，我更加推荐你使用CMake工具来一键生成Makefile。CMake的语法更加通俗易懂，并且对于大型项目的组织更加高效。

走到这一步，你对Linux环境下C编程已经有个初步的了解。再进一步，你需要熟悉Linux环境提供的C库函数，这对于你的编程有着莫大的帮助。你起码得知道文件操作、输入输出操作、进程操作、多线程同步操作等。这部分内容各大书籍或者视频都有详细的介绍，就看你愿意深入了解多少。

学到这一步，恭喜你，现在你已经是个合格的Linux C程序员了，可以编写一些简单的程序。

往后的学习可以根据你个人的兴趣，你可以钻研Linux环境下网络编程，可以深入阅读Linux内核源码，可以走应用开发，也可以走驱动开发。


## 参考资料

- [Linux内核2.6.11源码](https://elixir.bootlin.com/linux/v2.6.11/source)

- [趣谈Linux操作系统](https://time.geekbang.org/column/intro/100024701?utm_campaign=geektime_search&utm_content=geektime_search&utm_medium=geektime_search&utm_source=geektime_search&utm_term=geektime_search&tab=catalog)

- [正点原子第四期Linux驱动开发](https://www.bilibili.com/video/BV1fJ411i7PB/?p=1)

- [简说Linux内核开发50讲](https://www.bilibili.com/video/BV1Vz4y157zx/?vd_source=2207c170508349d2e7697d61612ec630)

- [精通嵌入式Linux编程](https://book.douban.com/subject/36479983/)

- [深入理解LINUX内核(第3版)](https://book.douban.com/subject/2287506/)

- [Linux内核设计与实现(第3版)](https://book.douban.com/subject/6097773/)

- [UNIX环境高级编程（第3版)](https://book.douban.com/subject/25900403/)

- [Linux命令行与shell脚本编程大全（第4版)](https://book.douban.com/subject/35933905/)

