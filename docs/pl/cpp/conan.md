# Conan

Conan是一个C/C++的包管理器，支持多个平台和构建系统。

```SHELL
$ conan --version
Conan versoin 2.3.0
```

Conan的设计理念与git类似，采用分布式架构。服务端负责包的存储，客户端从服务端获取或上传包，然后在本地编译。Conan提供官方仓库[Conan Center](https://conan.io/center)，用户也可以添加自己的仓库。

Conan中的包由一个"conanfile.py"定义。该文件定义了包的依赖、包含的源码、以及如何从源码构建出二进制文件。一个包的"conanfile.py"配置可以生成任意数量的二进制文件，每个二进制可以面向不同的平台和配置（操作系统、体系结构、编译器、以及构件类型等等）。

## Profile

使用Conan前，必须先创建一个profile。Conan会检测系统环境然后自动配置profile文件。比如"linux"环境的配置如下：

```SHELL
Host profile:
[settings]
arch=x86_64
build_type=Release
compiler=gcc
compiler.cppstd=gnu17
compiler.libcxx=libstdc++11
compiler.version=11
os=Linux

Build profile:
[settings]
arch=x86_64
build_type=Release
compiler=gcc
compiler.cppstd=gnu17
compiler.libcxx=libstdc++11
compiler.version=11
os=Linux
```

- Host：目标平台的配置
- Build：编译平台的配置

你可以通过profile相关命令生成和显示profile。与profile相关的命令有：

```SHELL
$conan profile -h

detect              Generate a profile using auto-detected values.
list                List all profiles in the cache.
path                Show profile path location.
show                Show aggregated profiles from the passed arguments.
```

注意：使用`conan install`命令时可选择系统中的profile配置，如不指定等同于`--profile=default`。

## conanfile

`conanfile`提供了更加精细的编译控制，一个简单的示例文件如下：

```python
from conan import ConanFile
from conan.tools.cmake import cmake_layout

class MyFile(ConanFile):
    settings = "os", "compiler", "build_type", "arch"
    requires = "zlib/1.2.11"
    generators = "CMakeToolchain", "CMakeDeps"
    
    def requirements(self):
        self.requires("zlib/1.2.11")
    
    def build_requirements(self):
        self.tool_requires("cmake/3.22.6")
    
    # using cmake layout
    def layout(self):
        cmake_layout(self)
```

## 交叉编译

Conan使用两种profile来创建最终的可执行程序：`conan install . --build=missing --profile:host=someprofile --profile:build=default`：

- `profile:host`：定义目标文件执行环境
- `profile:build`：定义目标文件编译环境

一种可能的交叉编译profile如下：

```SHELL
[settings]
os=Linux
arch=aarch64
compiler=gcc
build_type=Release
compiler.cppstd=gnu14
compiler.libcxx=libstdc++11
compiler.version=9
[buildenv]
CC=arm-linux-gnueabihf-gcc-9
CXX=arm-linux-gnueabihf-g++-9
LD=arm-linux-gnueabihf-ld
```

在conanfile中的体现方式是：

- `requirements()`：目标平台的依赖
- `build_requirements()`：编译平台的依赖

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

