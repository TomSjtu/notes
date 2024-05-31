# 容器技术

Docker是一种开源的{==容器化==}平台，用于构建、打包和运行应用程序和服务。它使用容器化技术(LXC)来封装应用程序及其依赖项，以便能够在不同的计算环境中进行快速、可靠的部署。

传统的部署方式需要开发人员针对不同的操作系统和环境安装不同的依赖，往往出现本地可以运行，但是到了部署环境就无法运行的问题。

Docker的解决方案是引入镜像(image)的概念。镜像是一个轻量级、独立的运行单元，包含了应用程序及其所需的所有组件（如库、依赖项、环境变量等）。每个镜像相互隔离，不会产生冲突，可以在任何支持Docker的环境中运行，无论是开发机、测试环境还是生产服务器。

Docker还提供了一套工具和命令行界面，使开发人员可以轻松地构建、打包、发布和管理Docker容器。这使得应用程序的交付和部署变得更加简单和可重复，加速了开发周期并提高了应用程序的可移植性和可扩展性。

Docker的整体架构如下图所示：

![Docker架构](../images/cloud/docker.PNG)

要构建一个镜像，可以有多种方式：

1. 从Docker官方仓库下载
2. 从本地容器构建
3. 从压缩文件tar构建
4. 从Dockerfile构建

## Docker常用命令

Docker API参考：[Docker命令](https://docs.docker.com/reference/cli/docker/)

这里只列出一些常用的命令：

| 命令 | 描述 |
| ---- | ---- |
| docker images | 查看镜像 |
| docker search | 搜索镜像 |
| docker pull | 拉取镜像 |
| docker rmi | 删除镜像 |
| docker ps | 查看容器 |
| docker run | 运行容器 |
| docker stop | 停止容器 |
| docker rm | 删除容器 |
| docker exec | 进入容器 |
| docker save | 将镜像打包 |
| docker load | 解包镜像 |

Docker命令总结：

![docker命令](../images/cloud/docker_cmd.png)

## Dockerfile

Dockerfile是一种特殊的脚本文件，可以由Docker引擎识别并从中构建镜像，它的语法比较简单，可以参考：[Dockerfile](https://docs.docker.com/reference/dockerfile/)

Dockerfile常用指令：

```dockerfile

FROM 指定基础镜像
 
RUN 执行命令
 
COPY 复制文件
 
WORKDIR 设置当前工作目录
 
VOLUME 设置卷，挂载主机目录
 
CMD 指定容器启动后的要干的事情
```
## Docker网络

Docker网络是Docker容器技术的一个核心组成部分，它负责管理容器之间以及容器与外部世界之间的通信。Docker网络的设计允许用户高效地配置和管理容器的网络连接，支持容器之间的隔离和通信，同时还能提供必要的安全性和可扩展性。

在Docker中，每个容器都可以被视为一个独立的网络实体，具有自己的IP地址、网络接口和路由规则。Docker网络为这些容器提供了各种连接选项，包括桥接、覆盖、主机网络等模式，使得容器的部署和管理更加灵活。

Docker网络模式主要有以下几种：

- 桥接模式：默认模式，容器拥有独立的网络命名空间
- 主机模式：与主机共享网络命名空间
- 无网络模式：没有网络接口，需要手动配置
- 覆盖模式：跨Docker主机通信

网络的配置和管理可以通过以下命令：

- `docker network create`：创建一个新的网络
- `docker network ls`：列出所有的网络
- `docker network inspect`：查看网络的详细信息
- `docker network connect`：将一个容器连接到网络
- `docker network disconnect`：将一个容器从网络断开

### 桥接网络

类似于VMware虚拟机中的 “桥接模式”，Docker的桥接网络（bridge network）是Docker容器使用的默认网络模式。在这种模式下，Docker宿主机会创建一个虚拟的网络桥接，允许容器通过这个桥接与外部网络通信。这种模式为每个容器提供了与主机隔离的网络环境，容器之间可以相互通信，同时也能与外部网络进行交互。

在Docker中，桥接网络提供了与VMware的桥接模式类似的功能，允许容器直接连接到物理网络，并且具有独立的IP地址。同时，Docker的端口映射功能则在某种程度上类似于VMware的NAT模式，它允许外部访问映射到宿主机端口的容器服务。Docker网络的这些特性使得它非常适合于容器化环境，为容器提供了灵活性和隔离性。

在桥接网络模式中，可以使用端口映射将容器内的端口映射到宿主机，从而使得外部网络可以访问容器内的应用。

### 主机网络

主机网络（host network）模式允许容器共享宿主机的网络命名空间。这意味着容器不会像在桥接或覆盖网络模式中那样获得自己的网络接口，而是直接使用宿主机的网络接口。当容器运行在主机网络模式下时，它能够无障碍地访问外部网络，同时也能够更高效地进行网络通信，因为不需要通过Docker的网络堆栈来转发数据。








