# 安全启动

Secure boot 是一种安全机制，它可以防止恶意代码或恶意软件在启动时运行，保护系统免受恶意软件的攻击。厂商为了商业利益都要提供安全启动功能，防止"黑客"刷机篡改程序，窃取商业机密。

安全启动主要关注两件事：

1. 怎么从 bootrom 开始构建 image 的信任链？
2. 使用什么加密算法校验不会被破解？

下图是一个标准的安全启动的流程：

- ATF：Arm Trust Firmware，运行在 EL3 异常级别，负责启动过程的安全检查和认证。ATF = BL1、BL2、BL31、BL2、BL33，都运行在 EL3 级别。
- BL1：也叫 bootrom，在 CPU 出厂时就被写死，是 SoC 上电后执行的第一段代码，负责对镜像进行校验和解密。
- BL2：存放在 flash 中的一段可信安全启动代码，主要完成一些平台相关的初始化，比如对 ddr 的初始化等。因为 BL31 和 BL32 是一个runtime，也就是上电后一直运行的程序，那么需要加载到内存里面，需要先初始化内存 ddr，BL2 就干这个事情的。所谓的Loader。
- BL31：作为 EL3 最后的安全堡垒，它不像 BL1 和 BL 2是一次性运行的。如它的 runtime 名字暗示的那样，它通过 SMC 指令为Non-Secure OS 持续提供设计安全的服务，在 Secure World 和 Non-Secure World 之间进行切换。是对硬件最基础的抽象，对 OS 提供服务。例如一个 EL3 级别的特权指令，比如关机、休眠等 OS 是无权处理的，就会交给 BL31 来继续操作硬件处理。
- BL32：是所谓的 secure os，运行在 secure mode。在 ARM 平台下是 ARM 家的 Trusted Execution Environment（TEE）实现。OP-TEE 是基于 ARM TrustZone 硬件架构所实现的软件 Secure OS。
- BL33：就是所谓的 uboot，之后就会启动 linux 内核了。

每一个阶段，上一阶段都会对下一个阶段的镜像进行校验，发现有改动就终止启动了，主打一个防篡改。启动 BL1、BL2、BL31、BL32、BL33、Linux 是一个完整的 ATF 信任链建立流程，负责加载镜像的 BL1、BL2、BL33 都不是 runtime，只要 Linux 系统启动，就不会再有加载镜像的机会了。

## 镜像加密算法RSA

RSA 加密算法是使用最广泛的非对称加密算法，即加密和解密不使用同样的密码和规则，只要这两种规则之间存在某种对应关系即可。这种算法非常可靠，密钥越长，它就越难破解。根据已经披露的文献，目前被破解的最长 RSA 密钥是 768个二进制位。也就是说，长度超过 768 位的密钥，还无法破解（至少没人公开宣布）。因此可以认为，1024 位的 RSA 密钥基本安全，2048 位的密钥极其安全。

Secure boot中使用 RSA 加密算法对镜像进行签名，具体步骤为：

1. 使用 hash 算法计算镜像的 hash 值
2. 编译时用私钥将 hash 值签名，并存入镜像中，放在指定位置
3. 启动上电后拿到公钥，将存储在镜像中特定位置的 hash 值解密
4. 对比解密后的 hash 值和计算出的 hash 值是否一致，如果一致，则认为镜像没有被篡改，可以正常启动