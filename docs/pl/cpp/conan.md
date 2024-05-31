# Conan

Conan是一个可以帮C/C++进行依赖管理的包管理器。它是免费、开源、跨平台的。目前支持Windows, Linux, OSX, FreeBSD, Solaris等平台。同时也支持嵌入式、移动端（IOS，Andriod）、或者直接基于裸机之上的目标程序开发。它当前支持各种构件系统，例如CMake，MSBuild，Makefiles，Scons等等。

Conan是分布式的，它允许运行自己私有的包管理器托管自己私有的包和二进制文件。Conan是基于二进制管理的，它可以为包生成各种不同配置、不同体系架构或者编译器版本的二进制文件。

Conan相对比较成熟和稳定，有一个全职团队在维护它。Conan保证前向兼容，社区相对成熟，从开源到商业公司都有使用。它有一个官方的中央仓[Conan Center](https://conan.io/center)。

Conan的架构是分布式的，客户端从远端server获取或上传包，然后在本地编译。

## 二进制的包管理

Conan最强大的特性使它能为任何可能的平台和配置生成和管理预编译的二级制文件。使用预编译的二进制文件可以避免用户反复的从源码进行构建，节省大量的开发以及持续集成服务器用于构建的时间，同时也提高了交付件的可重现性和可跟踪性。

Conan中的包由一个"conanfile.py"定义。该文件定义了包的依赖、包含的源码、以及如何从源码构建出二进制文件。一个包的"conanfile.py"配置可以生成任意数量的二进制文件，每个二进制可以面向不同的平台和配置（操作系统、体系结构、编译器、以及构件类型等等）。二进制的创建和上传，在所有平台上使用相同的命令，并且都是基于一套包的源码产生的。使用Conan不用为不同的操作系统提供不同的解决方案。


## 常用命令

```SHELL
# 安装conan相关配置
conan config install https://github.com/conan-io/conan-center-index.git

# 屏蔽conan仓库
conan remote disable conancenter

# 查找本地所有的依赖库
conan search

# 删除本地依赖库缓存
conan remove [package_name]

# 查看本地库信息
conan search [package_name]

# 查看添加的远程仓库
conan remote list

# 添加远程仓库
conan remote add name [repo_name]
```
