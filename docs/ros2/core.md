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




## 节点

## 话题

## 服务

## 通信接口

## 动作

