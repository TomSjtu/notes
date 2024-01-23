# 驱动框架

> 驱动框架下各类子系统的介绍，需要有[设备树](./dts.md)的知识。

## Pinctrl子系统

通过寄存器映射来操作外设的编程方式，在驱动开发中是不建议的，因为每当芯片的寄存器发生变化，那么底层的驱动代码就得推翻重写。在驱动开发中，有没有办法可以不涉及到具体的寄存器操作呢？对于有些外设，是具备抽象条件的，Pintrl子系统就是其中之一。

Pinctrl：Pin Controller，引脚控制：

- 引脚枚举与命名
- 引脚复用：一个引脚可以用作GPIO，也可以用作I2C或UART
- 引脚配置：比如上拉、下拉

Pinctrl驱动由芯片厂家BSP工程师提供，一般的驱动工程师只需要在设备树里描述即可：

- 指明使用哪些引脚
- 复用为什么功能
- 配置为什么状态

### Pinctrl主要数据结构

pinctrl可以用一个结构体来表示它：`pinctrl_dev`。而`pinctrl_dev`可以用`pinctrl_desc`结构体描述，然后调用`pinctrl_register()`函数来注册它，其返回值就是一个`pinctrl_dev`的结构体：

```C
struct pinctrl_dev *pinctrl_register(struct pinctrl_desc *pctldesc, struct device *dev, void *driver_data);
```

pinctrl_desc:

```C
struct pinctrl_pin_desc {
    unsigned number;
    const char *name;
    void *drv_data;
};
```

pinctrl_ops：

```C
struct pinctrl_ops {
	int (*get_groups_count) (struct pinctrl_dev *pctldev);
	const char *(*get_group_name) (struct pinctrl_dev *pctldev,
				       unsigned selector);
	int (*get_group_pins) (struct pinctrl_dev *pctldev,
			       unsigned selector,
			       const unsigned **pins,
			       unsigned *num_pins);
	void (*pin_dbg_show) (struct pinctrl_dev *pctldev, struct seq_file *s,
			  unsigned offset);
	int (*dt_node_to_map) (struct pinctrl_dev *pctldev,
			       struct device_node *np_config,
			       struct pinctrl_map **map, unsigned *num_maps);
	void (*dt_free_map) (struct pinctrl_dev *pctldev,
			     struct pinctrl_map *map, unsigned num_maps);
};
```

pinmux_pos：

```C
struct pinmux_ops {
	int (*request) (struct pinctrl_dev *pctldev, unsigned offset);
	int (*free) (struct pinctrl_dev *pctldev, unsigned offset);
	int (*get_functions_count) (struct pinctrl_dev *pctldev);
	const char *(*get_function_name) (struct pinctrl_dev *pctldev,
					  unsigned selector);
	int (*get_function_groups) (struct pinctrl_dev *pctldev,
				  unsigned selector,
				  const char * const **groups,
				  unsigned *num_groups);
	int (*set_mux) (struct pinctrl_dev *pctldev, unsigned func_selector,
			unsigned group_selector);
	int (*gpio_request_enable) (struct pinctrl_dev *pctldev,
				    struct pinctrl_gpio_range *range,
				    unsigned offset);
	void (*gpio_disable_free) (struct pinctrl_dev *pctldev,
				   struct pinctrl_gpio_range *range,
				   unsigned offset);
	int (*gpio_set_direction) (struct pinctrl_dev *pctldev,
				   struct pinctrl_gpio_range *range,
				   unsigned offset,
				   bool input);
	bool strict;
};
```

pinconf_ops：

```C
struct pinconf_ops {
#ifdef CONFIG_GENERIC_PINCONF
	bool is_generic;
#endif
	int (*pin_config_get) (struct pinctrl_dev *pctldev,
			       unsigned pin,
			       unsigned long *config);
	int (*pin_config_set) (struct pinctrl_dev *pctldev,
			       unsigned pin,
			       unsigned long *configs,
			       unsigned num_configs);
	int (*pin_config_group_get) (struct pinctrl_dev *pctldev,
				     unsigned selector,
				     unsigned long *config);
	int (*pin_config_group_set) (struct pinctrl_dev *pctldev,
				     unsigned selector,
				     unsigned long *configs,
				     unsigned num_configs);
	void (*pin_config_dbg_show) (struct pinctrl_dev *pctldev,
				     struct seq_file *s,
				     unsigned offset);
	void (*pin_config_group_dbg_show) (struct pinctrl_dev *pctldev,
					   struct seq_file *s,
					   unsigned selector);
	void (*pin_config_config_dbg_show) (struct pinctrl_dev *pctldev,
					    struct seq_file *s,
					    unsigned long config);
};
```