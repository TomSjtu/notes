# GIC

GIC支持三种类型的中断

SGI（Software Generated Interrupt）：软件产生的中断，可以用于多核间的通信，一个CPU可以通过写GIC的寄存器给另外一个CPU产生中断。

PPI（Private Peripheral Interrupt）：某个CPU私有外设的中断，只能发给绑定的那个CPU。

SPI（Shared Peripheral Interrupt）：共享外设的中断，可以发送给任何一个CPU。

ARM Linux的中断默认在CPU0上产生。