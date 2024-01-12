# 快速入门

Linux知识体系太过庞大，从入门的角度来讲，掌握最基本的命令，熟悉Linux环境下C编程就可以了。

首先，你需要一个安装了ubuntu或者其他发行版的虚拟机，或者你可以选择云服务器。我更加推荐你用云服务器的方式来学习Linux，因为：

1. 云服务器没有桌面界面，这会逼迫你只能通过命令行的方式操作Linux。
2. 花了钱的东西你才会心疼，你才会投注精力去学习。
3. 云服务器只需要你电脑能联网就可以操作，而虚拟机占用电脑资源大。
4. 云服务器对于网络环境做了适配，并且各大服务器提供商提供了更加便捷的操作环境（一键配置环境），等同于租房拎包入住，非常方便。

当你购买了某个公司的云服务器之后，你可以通过ssh的方式连接上服务器，在vscode下载ssh插件可以进行远程开发，就和本地开发一样方便。具体如何操作请参考网上教程。

当你进入Linux环境之后，你需要使用sudo apt update&&sudo apt upgrade命令来更新你的软件包。你可以使用sudo apt install [pkg_name]，来安装你需要的软件包。如果你是在自己的虚拟机上操作，在更新之前你还需要更换镜像源为国内镜像源。

要在命令行下操作，你需要熟练掌握大约10多个命令，其余的命令你用到再查即可，没有必要专门去记。同时你还需要了解管道和重定向的概念。

要在命令行模式下对文件进行编辑，你至少得熟悉一种文本编辑器，比如VIM。但是不要期望一上来就熟练掌握VIM，这是不可能的。工具只是工具，如果它影响了你的效率，那它就不配被使用，至少在这个阶段不适合你。

请不要再点击RUN按钮运行你的程序，转而使用命令行的方式编译你的源代码。gcc是怎么将你写的代码编译成可执行文件的？你编写的多个文件是如何链接成一个程序的？这些随着你的深入自然而然会掌握。

当你的源文件多起来，并且项目工程逐渐复杂起来时，每次敲一长串gcc命令会显得费时又费力。此时你需要Makefile文件来组织你的工程，为了编写Makefile文件你需要了解基本的Makefile语法。当然，我更加推荐你使用CMake工具来一键生成Makefile。CMake的语法更加通俗易懂，并且对于大型项目的组织更加高效。

走到这一步，你对Linux环境下C编程已经有个初步的了解。再进一步，你需要熟悉Linux环境提供的C库函数，比如最基本的文件操作、输入输出操作、进程操作、多线程同步操作等。这部分内容各大书籍或者视频都有详细的介绍。

以上内容是Linux C开发的基础，后续的深入学习请自行查找资料。

## 参考资料

- [趣谈Linux操作系统](https://time.geekbang.org/column/intro/100024701?utm_campaign=geektime_search&utm_content=geektime_search&utm_medium=geektime_search&utm_source=geektime_search&utm_term=geektime_search&tab=catalog)

- [深入Linux内核架构](https://book.douban.com/subject/4843567/)

- [精通嵌入式Linux编程](https://book.douban.com/subject/36479983/)

- [深入理解LINUX内核(第3版)](https://book.douban.com/subject/2287506/)

- [Linux内核设计与实现(第3版)](https://book.douban.com/subject/6097773/)

- [UNIX环境高级编程（第3版)](https://book.douban.com/subject/25900403/)

- [Linux命令行与shell脚本编程大全（第4版)](https://book.douban.com/subject/35933905/)

- [野火Linux开发实战指南](https://doc.embedfire.com/linux/imx6/base/zh/latest/index.html)



