# 快速入门

Linux的知识体系太过庞大，但是入门并不难，专注于基础命令和Linux环境下的C编程，对于打下基础非常重要。

首先，你需要一个安装了ubuntu或者其他发行版的虚拟机，但我更加推荐你用云服务器来学习Linux，因为：

1. 云服务器通常不提供桌面界面，这可以迫使你更深入地理解并掌握各种命令行操作。
2. 由于云服务器通常需要付费，你会更加珍惜这个学习机会，并投入更多的时间和精力去学习。
3. 云服务器只需要你电脑能联网就可以操作，而虚拟机对于一般的轻薄本来说是个不小的负担。
4. 云服务器对于网络环境做了优化，并且各大服务器厂商提供了一键配置环境的功能，使用起来非常方便。

当你购买了云服务器之后，你可以通过SSH远程连接到服务器，VSCODE的SSH插件可以进行远程开发，体验与本地开发无异。具体的步骤你可以参考网上的相关教程。

一旦你成功进入Linux环境，首先应该执行的是`sudo apt update`和`sudo apt upgrade`命令，以更新你的软件包。安装软件包可以使用`sudo apt install [pkg_name]`命令。如果你是在本地虚拟机上操作，建议在更新软件包前将软件源更换为国内的镜像源，以获得更快的下载速度。

在命令行界面下，你需要熟练掌握大约十几个基础命令，其他命令可以在需要时再查询，无需专门记忆。同时，你还需要理解管道和重定向的概念。

为了在命令行模式下编辑文件，你至少需要熟悉一种文本编辑器，例如VIM。但不要期望一开始就能熟练使用VIM，这是一个逐步学习的过程。记住，工具是为了提高效率，如果它降低了你的效率，那么暂时可以选择其他更适合你的编辑器。

请停止使用“RUN”按钮来运行程序，而是通过命令行来编译你的源代码。了解GCC如何将你的代码编译成可执行文件，以及多个文件是如何链接成一个程序的过程，这些知识将随着你的深入学习而逐渐掌握。

随着源文件数量的增加和项目工程的复杂化，手动输入冗长的GCC命令会变得低效。这时，你需要使用Makefile文件来组织你的工程。为了编写Makefile文件，你需要了解其基本语法。不过，我更推荐你使用CMake工具，它能够一键生成Makefile，并且其语法更加直观易懂，特别适合管理大型项目。

到达这一步，你已经对Linux环境下的C编程有了初步的认识。接下来，你需要熟悉Linux提供的C库函数，包括文件操作、输入输出、进程控制、多线程同步等。这些内容在许多书籍和视频中都有详细的讲解。

以上内容构成了Linux C开发的基础。对于更深入的学习，请根据个人兴趣和需求自行查找相关资料。

## 参考资料

- [趣谈Linux操作系统](https://time.geekbang.org/column/intro/100024701?utm_campaign=geektime_search&utm_content=geektime_search&utm_medium=geektime_search&utm_source=geektime_search&utm_term=geektime_search&tab=catalog)

- [深入Linux内核架构](https://book.douban.com/subject/4843567/)

- [精通嵌入式Linux编程](https://book.douban.com/subject/36479983/)

- [深入理解LINUX内核(第3版)](https://book.douban.com/subject/2287506/)

- [Linux内核设计与实现(第3版)](https://book.douban.com/subject/6097773/)

- [UNIX环境高级编程（第3版)](https://book.douban.com/subject/25900403/)

- [Linux命令行与shell脚本编程大全（第4版)](https://book.douban.com/subject/35933905/)

- [野火Linux开发实战指南](https://doc.embedfire.com/linux/imx6/base/zh/latest/index.html)

- [蜗壳科技](http://www.wowotech.net/)

