# 异步IO

异步 IO 一直是 Linux 系统的痛。Linux 很早就有 POSIX AIO 这套异步 IO 实现，但它是在用户空间自己开用户线程模拟的，效率极其低下。后来在 Linux 2.6 引入了真正的内核级别支持的异步 IO 实现（Linux aio），但是它只支持 Direct IO，只支持磁盘文件读写，而且对文件大小还有限制，总之各种麻烦。

随着 Linux 5.1 的发布，Linux 终于有了自己好用的异步 IO 实现，并且支持大多数文件类型（磁盘文件、socket，管道等），这个就是本文的主角：io_uring。

相比于Linux传统的异步I/O机制，io_uring的优势主要体现在以下几个方面：

1. 高效：一方面，io_uring 采用了共享内存的方式来传递参数，减少了数据拷贝；另一方面，采用 ringbuffer 的方式来实现批量的 IO 请求，减少了系统调用的次数。
2. 可扩展性强，具体表现在：
    - 支持的 IO 设备类型多样化，不仅支持块设备的 IO，还支持任何基于文件的 IO，例如管道、socket。
    - 支持的 IO 操作多样化，不仅支持常见的 read/write 操作，还支持 send/recv/close/sync 等。
3. 易用：io_uring 提供了配套的 liburing 库，对底层的系统调用进行了大量的封装。
4. 可伸缩：io_uring 提供了 poll 模式，对于 IO 性能要求较高的场景，允许用户牺牲一定的 CPU 来获得更高的 IO 性能。
   

总结 io_uring ：一套全新的 syscall，一套全新的 async API，更高的性能，更好的兼容性，来迎接高 IOPS，高吞吐量的未来。