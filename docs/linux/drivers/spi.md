# SPI子系统

## SPI协议

SPI物理总线示意图如下所示：

![spi物理总线](../../images/kernel/spi-bus.png)

SPI有四根控制线，包括：

- SCK：时钟线，数据收发同步
- MOSI：数据线，主设备发送、从设备接收
- MISO：数据线，主设备接收、从设备发送
- CS：片选线，使能从设备



