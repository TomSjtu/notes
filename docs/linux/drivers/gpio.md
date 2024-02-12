# GPIO子系统

GPIO全称“General Purpose Input/Output”，通用输入输出。GPIO可能是芯片自带的，也可能通过I2C、SPI接口扩展。

## 设备树描述

```C
/*rk3568.dtsi*/
gpio0: gpio@fdd60000 {
	compatible = "rockchip,gpio-bank";
	reg = <0x0 0xfdd60000 0x0 0x100>;
	interrupts = <GIC_SPI 33 IRQ_TYPE_LEVEL_HIGH>;
	clocks = <&pmucru PCLK_GPIO0>;
	gpio-controller;
	#gpio-cells = <2>;
	interrupt-controller;
	#interrupt-cells = <2>;
};
```

## 数据结构

```C
struct gpio_device {
	int			id;
	struct device		dev;
	struct cdev		chrdev;
	struct device		*mockdev;
	struct module		*owner;
	struct gpio_chip	*chip;
	struct gpio_desc	*descs;
	int			base;
	u16			ngpio;
	const char		*label;
	void			*data;
	struct list_head        list;
	struct blocking_notifier_head notifier;
	struct rw_semaphore	sem;

#ifdef CONFIG_PINCTRL
	/*
	 * If CONFIG_PINCTRL is enabled, then gpio controllers can optionally
	 * describe the actual pin range which they serve in an SoC. This
	 * information would be used by pinctrl subsystem to configure
	 * corresponding pins for gpio usage.
	 */
	struct list_head pin_ranges;
#endif
};
```

每个GPIO controller都会用一个`struct gpio_device`结构体来表示，其中：

- 在`struct gpio_chip`中提供引脚操作函数

- 在`struct gpio_desc`中提供每个引脚的信息

```C
struct gpio_chip {
	const char		*label;
	struct gpio_device	*gpiodev;
	struct device		*parent;
	struct module		*owner;

	int			(*request)(struct gpio_chip *gc,
						unsigned int offset);
	void			(*free)(struct gpio_chip *gc,
						unsigned int offset);
	int			(*get_direction)(struct gpio_chip *gc,
						unsigned int offset);
	int			(*direction_input)(struct gpio_chip *gc,
						unsigned int offset);
	int			(*direction_output)(struct gpio_chip *gc,
						unsigned int offset, int value);
	int			(*get)(struct gpio_chip *gc,
						unsigned int offset);
	int			(*get_multiple)(struct gpio_chip *gc,
						unsigned long *mask,
						unsigned long *bits);
	void			(*set)(struct gpio_chip *gc,
						unsigned int offset, int value);
	void			(*set_multiple)(struct gpio_chip *gc,
						unsigned long *mask,
						unsigned long *bits);
	int			(*set_config)(struct gpio_chip *gc,
					      unsigned int offset,
					      unsigned long config);
	int			(*to_irq)(struct gpio_chip *gc,
						unsigned int offset);

	void			(*dbg_show)(struct seq_file *s,
						struct gpio_chip *gc);

	int			(*init_valid_mask)(struct gpio_chip *gc,
						   unsigned long *valid_mask,
						   unsigned int ngpios);

	int			(*add_pin_ranges)(struct gpio_chip *gc);

	int			base;
	u16			ngpio;
	u16			offset;
	const char		*const *names;
	bool			can_sleep;
};
```

> label：GPIO控制器名称

> gpiodevice：GPIO控制器设备

> parent：GPIO控制器父设备

> base：GPIO引脚基值

> ngpio：GPIO引脚数量

> offset：GPIO引脚偏移

> names：GPIO引脚名称

> can_sleep：是否可以在睡眠状态下访问GPIO

使用`gpiochip_add_data()`宏来注册`struct gpio_chip`。

每个引脚都对应一个`struct gpio_desc`，引脚信息被保存在一个链表中：

```C
struct gpio_desc {
	struct gpio_device	*gdev;
	unsigned long		flags;
/* flag symbols are bit numbers */
#define FLAG_REQUESTED	0
#define FLAG_IS_OUT	1
#define FLAG_EXPORT	2	/* protected by sysfs_lock */
#define FLAG_SYSFS	3	/* exported via /sys/class/gpio/control */
#define FLAG_ACTIVE_LOW	6	/* value has active low */
#define FLAG_OPEN_DRAIN	7	/* Gpio is open drain type */
#define FLAG_OPEN_SOURCE 8	/* Gpio is open source type */
#define FLAG_USED_AS_IRQ 9	/* GPIO is connected to an IRQ */
#define FLAG_IRQ_IS_ENABLED 10	/* GPIO is connected to an enabled IRQ */
#define FLAG_IS_HOGGED	11	/* GPIO is hogged */
#define FLAG_TRANSITORY 12	/* GPIO may lose value in sleep or reset */
#define FLAG_PULL_UP    13	/* GPIO has pull up enabled */
#define FLAG_PULL_DOWN  14	/* GPIO has pull down enabled */
#define FLAG_BIAS_DISABLE    15	/* GPIO has pull disabled */
#define FLAG_EDGE_RISING     16	/* GPIO CDEV detects rising edge events */
#define FLAG_EDGE_FALLING    17	/* GPIO CDEV detects falling edge events */
#define FLAG_EVENT_CLOCK_REALTIME	18 /* GPIO CDEV reports REALTIME timestamps in events */

	/* Connection label */
	const char		*label;
	/* Name of the GPIO */
	const char		*name;
#ifdef CONFIG_OF_DYNAMIC
	struct device_node	*hog;
#endif
#ifdef CONFIG_GPIO_CDEV
	/* debounce period in microseconds */
	unsigned int		debounce_period_us;
#endif
};
```

> gdev：属于哪个GPIO controller

> flags：标志位，表示引脚的状态

> label：一般等于`struct gpio_chip`的label

> name：引脚名

## GPIO函数接口

定义在<include/linux/gpio/consumer.h\>中，常用函数有：

获得GPIO：
```C
struct gpio_desc *__must_check gpiod_get(struct device *dev,
					 const char *con_id,
					 enum gpiod_flags flags);
struct gpio_desc *__must_check gpiod_get_index(struct device *dev,
					       const char *con_id,
					       unsigned int idx,
					       enum gpiod_flags flags);
struct gpio_descs *__must_check gpiod_get_array(struct device *dev,
						const char *con_id,
						enum gpiod_flags flags);

struct gpio_desc *__must_check devm_gpiod_get(struct device *dev,
					      const char *con_id,
					      enum gpiod_flags flags);
struct gpio_desc *__must_check devm_gpiod_get_index(struct device *dev,
						    const char *con_id,
						    unsigned int idx,
						    enum gpiod_flags flags);
struct gpio_descs *__must_check devm_gpiod_get_array(struct device *dev,
						     const char *con_id,
						     enum gpiod_flags flags);
```

设置方向：
```C
int gpiod_direction_input(struct gpio_desc *desc);
int gpiod_direction_output(struct gpio_desc *desc, int value);
```

读写值：
```C
int gpiod_get_value(const struct gpio_desc *desc);
void gpiod_set_value(struct gpio_desc *desc, int value);


释放GPIO：
```C
void gpiod_put(struct gpio_desc *desc);
void gpiod_put_array(struct gpio_descs *descs);
```

GPIO描述符标志位：
```C
enum gpiod_flags {
	GPIOD_ASIS	= 0,
	GPIOD_IN	= GPIOD_FLAGS_BIT_DIR_SET,
	GPIOD_OUT_LOW	= GPIOD_FLAGS_BIT_DIR_SET | GPIOD_FLAGS_BIT_DIR_OUT,
	GPIOD_OUT_HIGH	= GPIOD_FLAGS_BIT_DIR_SET | GPIOD_FLAGS_BIT_DIR_OUT |
			  GPIOD_FLAGS_BIT_DIR_VAL,
	GPIOD_OUT_LOW_OPEN_DRAIN = GPIOD_OUT_LOW | GPIOD_FLAGS_BIT_OPEN_DRAIN,
	GPIOD_OUT_HIGH_OPEN_DRAIN = GPIOD_OUT_HIGH | GPIOD_FLAGS_BIT_OPEN_DRAIN,
};
```

要操作一个引脚，首先要获取这个引脚，然后设置方向，读值或者写值。

## 与Pinctrl子系统交互

1. 在GPIO设备树中使用`gpio-ranges`来描述它们之间的联系
2. 解析这些联系，在注册`struct gpio-chip`时自动调用
3. 在GPIO驱动程序中，提供`gpio_chip->request`函数；在Pinctrl子系统中，提供pmxops->gpio_request_enable函数或者`pmxops->request`函数

![GPIO和Pinctrl的映射](../../images/kernel/GPIO和Pinctrl的映射.png)