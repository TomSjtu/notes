# Socket Can

Socket Can子系统是Linux基于CAN协议的一种实现。CAN是一种在世界范围内广泛用于自动控制、嵌入式设备和汽车领域的网络技术。Linux下最早使用CAN的方法是基于字符设备来实现的，与之不同的是Socket Can使用伯克利的socket接口和Linux网络协议栈，这种方法使得CAN设备驱动可以通过网络接口来调用，熟悉网络编程的程序员可以快速上手。

!!! question

    为什么要使用Socket Can？

在Socket CAN之前，内核中已经有了一些CAN的实现方法，这些方法大多数基于字符设备，且只为某个具体的硬件设备提供驱动程序。那么缺点就很明显了。第一耦合性强，如果更换了CAN控制器，那么同时也要更换另一个设备驱动；第二，基于字符设备的实现，不支持多进程的访问，导致效率低下。第三，功能少，比如帧队列和ISO-TP这样的高层协议必须在用户空间来实现。

于是Socket Can项目应运而生，它基于Linux网络协议栈，实现了基于套接字的CAN驱动，使得CAN控制器可以像网络设备一样被访问。
 
CAN控制器的设备驱动将自己作为一个网络设备注册进Linux的网络层，收到的CAN帧可以传输给高层的网络协议和CAN协议族，反之，发送的帧也会通过高层给CAN控制器。传输协议模块可以使用协议族提供的接口注册自己，所以可以动态的加载和卸载多个传输协议。事实上，CAN核心模块不提供任何协议，也不能在没有加载其它协议的情况下单独使用。同一时间可以在相同或者不同的协议上打开多个套接字，可以在相同或者不同的CAN ID上同时监听和发送(listen/send)。几个同时监听具有相同ID帧的套接字可以在匹配的帧到来后接收到相同的内容。如果一个应用程序希望使用一个特殊的协议（比如ISO-TP）进行通信，只要在打开套接字的时候选择那个协议就可以了，接下来就可以读取和写入应用数据流了，根本无需关心CAN-ID和帧的结构等信息。

## 功能详解

与广为人知的TCP/IP协议以及以太网不同，CAN总线没有类似以太网的MAC层地址，只能用于广播。CAN ID仅仅用来进行总线的仲裁。因此CAN ID在总线上必须是唯一的。

## 使用

Socket Can的使用方法与socket非常相似，这里就不详细展开了。

```C
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <net/if.h>
#include <sys/ioctl.h>
#include <sys/socket.h>
#include <linux/can.h>
#include <linux/can/raw.h>

int main() {
    int s, nbytes;
    struct sockaddr_can addr;
    struct ifreq ifr;
    struct can_frame frame;
    
    // 创建socket CAN
    if ((s = socket(PF_CAN, SOCK_RAW, CAN_RAW)) < 0) {
        perror("Socket");
        return 1;
    }

    // 设置CAN接口的名字，例如"can0"
    strcpy(ifr.ifr_name, "can0");
    ioctl(s, SIOCGIFINDEX, &ifr);

    // 设置CAN协议的地址
    addr.can_family = AF_CAN;           //必须为AF_CAN
    addr.can_ifindex = ifr.ifr_ifindex;

    // 绑定socket到CAN接口
    bind(s, (struct sockaddr *)&addr, sizeof(addr));

    // ... 在这里发送和接收CAN帧 ...

    // 关闭socket
    close(s);

    return 0;
}
```

### 设置过滤规则：

```C
struct can_filter {
	canid_t can_id;
	canid_t can_mask;
};
```

在我们的应用程序中，如果没有设置过滤规则，应用程序默认会接收所有 ID 的报文；如果我们的应用程序只需要接收某些特定 ID 的报文（亦或者不接受所有报文，只发送报文），则可以通过 setsockopt 函数设置过滤规则，譬如某应用程序只接收 ID 为 0x60A 和 0x60B 的报文帧，则可将其它不符合规则的帧全部给过滤掉，示例代码如下所示：

```C
struct can_filter rfilter[2]; //定义一个 can_filter 结构体对象
 
// 填充过滤规则，只接收 ID 为(can_id & can_mask)的报文
rfilter[0].can_id = 0x60A;
rfilter[0].can_mask = 0x7FF;
rfilter[1].can_id = 0x60B;
rfilter[1].can_mask = 0x7FF;
 
// 调用 setsockopt 设置过滤规则
setsockopt(sockfd, SOL_CAN_RAW, CAN_RAW_FILTER, &rfilter, sizeof(rfilter));
```

### 数据的发送和接收

CAN总线用`struct can_frame`结构体将数据封装成帧：

```C
struct can_frame {
     canid_t can_id; /* CAN 标识符 */
     __u8 can_dlc; /* 数据长度（最长为 8 个字节） */
     __u8 __pad; /* padding */
     __u8 __res0; /* reserved / padding */
     __u8 __res1; /* reserved / padding */
     __u8 data[8]; /* 发送的数据 */
};
```

发送CAN帧：

```C
// 初始化CAN帧
frame.can_id = 0x123; // 设置CAN ID
frame.can_dlc = 3;    // 数据长度为3字节
frame.data[0] = 0xAA; // 第一个数据字节
frame.data[1] = 0xBB; // 第二个数据字节
frame.data[2] = 0xCC; // 第三个数据字节

// 发送CAN帧
nbytes = write(s, &frame, sizeof(frame));
if (nbytes != sizeof(frame)) {
    perror("Write");
    return 1;
}
```

接收CAN帧：

```C
// 接收CAN帧
nbytes = read(s, &frame, sizeof(frame));
if (nbytes < 0) {
    perror("Read");
    return 1;
}

printf("Received CAN frame with ID 0x%X and length %d\n", frame.can_id, frame.can_dlc);
```



