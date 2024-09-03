# 核心概念

## 工作空间

工作空间就是在开发ROS项目时，存放代码、配置文件等文件的地方。一个典型的工作空间包含以下几个目录：

- src：存放源码文件
- build：存放编译后的文件
- install：存放编译后的可执行文件
- log：存放日志文件

在创建工作空间前，需要先使用`rosdepc`工具安装系统依赖项：

```SHELL
# 初始化
$ rosdepc init
$ rosdepc update

# 安装系统依赖项
$ rosdep install --from-paths <package_directory> --ignore-src 
```

依赖安装完成后，就可以使用`colcon build`命令编译项目。编译完成后，目录下新产生三个文件夹：build, install, log。

```SHELL
root@84036d2ba497:/userdata# tree
.
├── build
│   └── COLCON_IGNORE
├── install
│   ├── COLCON_IGNORE
│   ├── _local_setup_util_ps1.py
│   ├── _local_setup_util_sh.py
│   ├── local_setup.bash
│   ├── local_setup.ps1
│   ├── local_setup.sh
│   ├── local_setup.zsh
│   ├── setup.bash
│   ├── setup.ps1
│   ├── setup.sh
│   └── setup.zsh
└── log
    ├── COLCON_IGNORE
    ├── build_2024-07-15_14-32-45
    │   ├── events.log
    │   └── logger_all.log
    ├── latest -> latest_build
    └── latest_build -> build_2024-07-15_14-32-45

6 directories, 15 files
```

为了让系统能够找到我们的功能包和可执行文件，还需要设置环境变量：

```SHELL
$ echo " source ~/dev_ws/install/local_setup.sh" >> ~/.bashrc
```

## 功能包

功能包是组织和分发代码的基本方式，通过定义清晰的接口和依赖关系，功能包能够方便地在不同的项目和团队之间共享和复用。

一个典型的 ROS2 功能包可能具有以下目录结构：

```SHELL
my_package/
  CMakeLists.txt       # CMake构建配置文件
  package.xml          # 功能包元信息，包括名称、版本、依赖等
  src/                 # 源代码目录
    my_node.cpp
  include/
    my_package/        # 头文件目录
      my_node.hpp
  scripts/             # 脚本文件目录
    my_script.py
  msg/                 # 消息定义目录
    MyMessage.msg
  srv/                 # 服务定义目录
    MyService.srv
```

创建一个功能包：

```SHELL
ros2 pkg create <package_name> --dependencies [dependencies]
```

构建功能包：

```SHELL
colcon build --packages-select <package_name>
```

运行节点：

```SHELL
ros2 run <package_name> <node_name>
```

## 节点

节点，就是执行某些具体任务的单元，在 OS 的视角来看，其实就是一个进程。每个节点都必须是一个独立的可执行的文件。

节点的创建需要有四个步骤：

- 编程接口初始化
- 创建节点并初始化
- 实现节点功能
- 销毁节点并关闭接口

## 话题

ROS2 的话题通信采用的是发布/订阅模式，二者规定统一的数据的描述格式。


## 服务

## 通信接口

## 动作

