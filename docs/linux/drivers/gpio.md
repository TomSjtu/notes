# GPIO子系统

GPIO全称“General Purpose Input/Output”，通用输入输出。GPIO可能是芯片自带的，也可能通过I2C、SPI接口扩展。



## 数据结构

```C
struct gpio_device {
	int id;
	struct device dev;
	struct cdev chrdev;
	struct device *mockdev;
	struct module *owner;
	struct gpio_chip *chip;
	struct gpio_desc *descs;
	int base;
	u16	ngpio;
	const char *label;
	void *data;
	struct list_head list;
	struct blocking_notifier_head notifier;
	struct rw_semaphore	sem;

#ifdef CONFIG_PINCTRL
	struct list_head pin_ranges;
#endif
};
```

```C
struct gpio_desc {
	struct gpio_device *gdev;
	unsigned long flags;

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

	const char		*label;
	const char		*name;
};
```

