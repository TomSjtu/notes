# GPIO子系统

GPIO全称“General Purpose Input/Output”，通用输入输出。GPIO可能是芯片自带的，也可能通过I2C、SPI接口扩展。

## 设备树描述

对于GPIO控制器，对应的设备节点需要声明gpio-controller属性，并设置#gpio-cells的大小，比如对于rk3399而言的GPIO控制器而言：

```C title="rk3399.dtsi"

gpio0: gpio0@ff720000 {
	compatible = "rockchip,gpio-bank";
	reg = <0x0 0xff720000 0x0 0x100>;
	clocks = <&pmucru PCLK_GPIO0_PMU>;
	interrupts = <GIC_SPI 14 IRQ_TYPE_LEVEL_HIGH 0>;

	gpio-controller;
	#gpio-cells = <0x2>;

	interrupt-controller;
	#interrupt-cells = <0x2>;
};

gpio1: gpio1@ff730000 {
	compatible = "rockchip,gpio-bank";
	reg = <0x0 0xff730000 0x0 0x100>;
	clocks = <&pmucru PCLK_GPIO1_PMU>;
	interrupts = <GIC_SPI 15 IRQ_TYPE_LEVEL_HIGH 0>;

	gpio-controller;
	#gpio-cells = <0x2>;

	interrupt-controller;
	#interrupt-cells = <0x2>;
};
```

对于需要用到GPIO控制器的设备，需要在节点中声明


```C title="rk3399-firefly.dts"

leds {
	compatible = "gpio-leds";
	pinctrl-names = "default";
	pinctrl-0 = <&work_led_pin>, <&diy_led_pin>;

	work_led: led-0 {
		label = "work";
		default-state = "on";
		gpios = <&gpio2 RK_PD3 GPIO_ACTIVE_HIGH>;
	};

	diy_led: led-1 {
		label = "diy";
		default-state = "off";
		gpios = <&gpio0 RK_PB5 GPIO_ACTIVE_HIGH>;
	};
};
```

!!! info

	设备驱动可以通过`of_get_named_gpio()`函数获取GPIO。

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

GPIO的函数接口有两套：legacy模式和基于descriptor的。由于第一套已被废弃，这里只介绍基于descriptor的。

函数定义在<include/linux/gpio/consumer.h\>中：

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
```

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

## 简单示例

```C
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/platform_device.h>      /* For platform devices */
#include <linux/gpio/consumer.h>        /* For GPIO Descriptor interface */
#include <linux/interrupt.h>            /* For IRQ */
#include <linux/of.h>                   /* For DT*/

/*
 * Let us consider the bellow mapping
 *
 *    foo_device {
 *       compatible = "packt,gpio-descriptor-sample";
 *       led-gpios = <&gpio2 15 GPIO_ACTIVE_HIGH>, // red 
 *                   <&gpio2 16 GPIO_ACTIVE_HIGH>, // green 
 *
 *       btn1-gpios = <&gpio2 1 GPIO_ACTIVE_LOW>;
 *       btn2-gpios = <&gpio2 1 GPIO_ACTIVE_LOW>;
 *   };
 */

static struct gpio_desc *red, *green, *btn1, *btn2;
static int irq;

static irqreturn_t btn1_pushed_irq_handler(int irq, void *dev_id)
{
    int state;

    /* read the button value and change the led state */
    state = gpiod_get_value(btn2);
    gpiod_set_value(red, state);
    gpiod_set_value(green, state);

    pr_info("btn1 interrupt: Interrupt! btn2 state is %d)\n", state);
    return IRQ_HANDLED;
}

static const struct of_device_id gpiod_dt_ids[] = {
    { .compatible = "packt,gpio-descriptor-sample", },
    { /* sentinel */ }
};


static int my_pdrv_probe (struct platform_device *pdev)
{
    int retval;
    struct device *dev = &pdev->dev;

    /*
     * We use gpiod_get/gpiod_get_index() along with the flags
     * in order to configure the GPIO direction and an initial
     * value in a single function call.
     *
     * One could have used:
     *  red = gpiod_get_index(dev, "led", 0);
     *  gpiod_direction_output(red, 0);
     */
    red = gpiod_get_index(dev, "led", 0, GPIOD_OUT_LOW);
    green = gpiod_get_index(dev, "led", 1, GPIOD_OUT_LOW);

    /*
     * Configure Button GPIOs as input
     *
     * After this, one can call gpiod_set_debounce()
     * only if the controller has the feature
     * For example, to debounce  a button with a delay of 200ms
     *  gpiod_set_debounce(btn1, 200);
     */
    btn1 = gpiod_get(dev, "btn1", GPIOD_IN);
    btn2 = gpiod_get(dev, "btn2", GPIOD_IN);

    irq = gpiod_to_irq(btn1);
    retval = request_threaded_irq(irq, NULL,
                            btn1_pushed_irq_handler,
                            IRQF_TRIGGER_LOW | IRQF_ONESHOT,
                            "gpio-descriptor-sample", NULL);
    pr_info("Hello! device probed!\n");
    return 0;
}

static int my_pdrv_remove(struct platform_device *pdev)
{
    free_irq(irq, NULL);
    gpiod_put(red);
    gpiod_put(green);
    gpiod_put(btn1);
    gpiod_put(btn2);
    pr_info("good bye reader!\n");
    return 0;
}

static struct platform_driver mypdrv = {
    .probe      = my_pdrv_probe,
    .remove     = my_pdrv_remove,
    .driver     = {
        .name     = "gpio_descriptor_sample",
        .of_match_table = of_match_ptr(gpiod_dt_ids),  
        .owner    = THIS_MODULE,
    },
};
module_platform_driver(mypdrv);

MODULE_AUTHOR("John Madieu <john.madieu@gmail.com>");
MODULE_LICENSE("GPL");
```

