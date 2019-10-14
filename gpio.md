# GPIO 

对于绝大多数的GPIO来说都是通过相关的寄存器来配置的。

GPIO 一般有 输入 和 输出、高点平 和 低电平、中断等配置，

对于输入、输出 需要 一个寄存器进行管理配置其工作状态。
对于高电平、低电平也需要一个寄存器来控制。
对于gpio 中断也需要一个寄存器来控制。

这些寄存器一般由GPIO控制器来进行管理。


对于一块开发板中存在很多个GPIO管脚，对于这些管脚一般进行分组管理，这样就需要多个GPIO控制器。

# GPIO 控制器 (**drivers/gpio**)

在linux驱动中对于这样的GPIO控制器，内核使用了struct gpio_chips 来描述，并且把所有的 gpio_chips用
链表来进行管理。对于GPIO 控制器的配置一般保存在设备树某个节点中。

对于某个具体的GPIO 管脚状态 struct gpio_desc 来描述。

```

// gpio 控制器设备树中 节点
// arch/arm/boot/dts/rk3188.dtsi
// linux 4.9
  
    #address-cells = <1>;
    #size-celss    = <1>;

		gpio0: gpio0@2000a000 {
			compatible = "rockchip,rk3188-gpio-bank0";
			
      reg = <0x2000a000 0x100>; // 控制器寄存器地址和大小

			interrupts = <GIC_SPI 54 IRQ_TYPE_LEVEL_HIGH>;// GIC_SPI 中断类型; 54 中断号;  IRQ_TYPE_LEVEL_HIGH 中断触发方式
			clocks = <&cru PCLK_GPIO0>;

			gpio-controller; // gpio 控制器
			#gpio-cells = <2>; //使用多少个cells 来描述gpio

			interrupt-controller; // 表示该节点为中断 控制器
			#interrupt-cells = <2>;
		};
```

```

// include/linux/gpio/driver.h
// linux version : 4.9

/**
 * struct gpio_chip - abstract a GPIO controller
 * @label: for diagnostics
 * @dev: optional device providing the GPIOs
 * @cdev: class device used by sysfs interface (may be NULL)
 * @owner: helps prevent removal of modules exporting active GPIOs
 * @list: links gpio_chips together for traversal
 * @request: optional hook for chip-specific activation, such as
 *	enabling module power and clock; may sleep
 * @free: optional hook for chip-specific deactivation, such as
 *	disabling module power and clock; may sleep
 * @get_direction: returns direction for signal "offset", 0=out, 1=in,
 *	(same as GPIOF_DIR_XXX), or negative error
 * @direction_input: configures signal "offset" as input, or returns error
 * @direction_output: configures signal "offset" as output, or returns error
 * @get: returns value for signal "offset"; for output signals this
 *	returns either the value actually sensed, or zero
 * @set: assigns output value for signal "offset"
 * @set_multiple: assigns output values for multiple signals defined by "mask"
 * @set_debounce: optional hook for setting debounce time for specified gpio in
 *      interrupt triggered gpio chips
 * @to_irq: optional hook supporting non-static gpio_to_irq() mappings;
 *	implementation may not sleep
 * @dbg_show: optional routine to show contents in debugfs; default code
 *	will be used when this is omitted, but custom code can show extra
 *	state (such as pullup/pulldown configuration).
 * @base: identifies the first GPIO number handled by this chip;
 *	or, if negative during registration, requests dynamic ID allocation.
 *	DEPRECATION: providing anything non-negative and nailing the base
 *	offset of GPIO chips is deprecated. Please pass -1 as base to
 *	let gpiolib select the chip base in all possible cases. We want to
 *	get rid of the static GPIO number space in the long run.
 * @ngpio: the number of GPIOs handled by this controller; the last GPIO
 *	handled is (base + ngpio - 1).
 * @desc: array of ngpio descriptors. Private.
 * @names: if set, must be an array of strings to use as alternative
 *      names for the GPIOs in this chip. Any entry in the array
 *      may be NULL if there is no alias for the GPIO, however the
 *      array must be @ngpio entries long.  A name can include a single printk
 *      format specifier for an unsigned int.  It is substituted by the actual
 *      number of the gpio.
 * @can_sleep: flag must be set iff get()/set() methods sleep, as they
 *	must while accessing GPIO expander chips over I2C or SPI. This
 *	implies that if the chip supports IRQs, these IRQs need to be threaded
 *	as the chip access may sleep when e.g. reading out the IRQ status
 *	registers.
 * @irq_not_threaded: flag must be set if @can_sleep is set but the
 *	IRQs don't need to be threaded
 * @irqchip: GPIO IRQ chip impl, provided by GPIO driver
 * @irqdomain: Interrupt translation domain; responsible for mapping
 *	between GPIO hwirq number and linux irq number
 * @irq_base: first linux IRQ number assigned to GPIO IRQ chip (deprecated)
 * @irq_handler: the irq handler to use (often a predefined irq core function)
 *	for GPIO IRQs, provided by GPIO driver
 * @irq_default_type: default IRQ triggering type applied during GPIO driver
 *	initialization, provided by GPIO driver
 * @irq_parent: GPIO IRQ chip parent/bank linux irq number,
 *	provided by GPIO driver
 * @lock_key: per GPIO IRQ chip lockdep class
 *
 * A gpio_chip can help platforms abstract various sources of GPIOs so
 * they can all be accessed through a common programing interface.
 * Example sources would be SOC controllers, FPGAs, multifunction
 * chips, dedicated GPIO expanders, and so on.
 *
 * Each chip controls a number of signals, identified in method calls
 * by "offset" values in the range 0..(@ngpio - 1).  When those signals
 * are referenced through calls like gpio_get_value(gpio), the offset
 * is calculated by subtracting @base from the gpio number.
 */
struct gpio_chip {
	const char		*label;
	struct device		*dev;
	struct device		*cdev;
	struct module		*owner;
	struct list_head        list;

	int			(*request)(struct gpio_chip *chip,
						unsigned offset);
	void			(*free)(struct gpio_chip *chip,
						unsigned offset);
	int			(*get_direction)(struct gpio_chip *chip,
						unsigned offset);
	int			(*direction_input)(struct gpio_chip *chip,
						unsigned offset);
	int			(*direction_output)(struct gpio_chip *chip,
						unsigned offset, int value);
	int			(*get)(struct gpio_chip *chip,
						unsigned offset);
	void			(*set)(struct gpio_chip *chip,
						unsigned offset, int value);
	void			(*set_multiple)(struct gpio_chip *chip,
						unsigned long *mask,
						unsigned long *bits);
	int			(*set_debounce)(struct gpio_chip *chip,
						unsigned offset,
						unsigned debounce);

	int			(*to_irq)(struct gpio_chip *chip,
						unsigned offset);

	void			(*dbg_show)(struct seq_file *s,
						struct gpio_chip *chip);
	int			base;
	u16			ngpio;
	struct gpio_desc	*desc;
	const char		*const *names;
	bool			can_sleep;
	bool			irq_not_threaded;

#ifdef CONFIG_GPIOLIB_IRQCHIP
	/*
	 * With CONFIG_GPIOLIB_IRQCHIP we get an irqchip inside the gpiolib
	 * to handle IRQs for most practical cases.
	 */
	struct irq_chip		*irqchip;
	struct irq_domain	*irqdomain;
	unsigned int		irq_base;
	irq_flow_handler_t	irq_handler;
	unsigned int		irq_default_type;
	int			irq_parent;
	struct lock_class_key	*lock_key;
#endif

#if defined(CONFIG_OF_GPIO)
	/*
	 * If CONFIG_OF is enabled, then all GPIO controllers described in the
	 * device tree automatically may have an OF translation
	 */
	struct device_node *of_node;
	int of_gpio_n_cells;
	int (*of_xlate)(struct gpio_chip *gc,
			const struct of_phandle_args *gpiospec, u32 *flags);
#endif
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

```

// drivers/gpio/gpiolib.h

struct gpio_desc {
	struct gpio_chip	*chip;
	unsigned long		flags; // 下面define就是管脚的配置功能。
/* flag symbols are bit numbers */
#define FLAG_REQUESTED	0
#define FLAG_IS_OUT	1
#define FLAG_EXPORT	2	/* protected by sysfs_lock */
#define FLAG_SYSFS	3	/* exported via /sys/class/gpio/control */
#define FLAG_ACTIVE_LOW	6	/* value has active low */
#define FLAG_OPEN_DRAIN	7	/* Gpio is open drain type */
#define FLAG_OPEN_SOURCE 8	/* Gpio is open source type */
#define FLAG_USED_AS_IRQ 9	/* GPIO is connected to an IRQ */
#define FLAG_IS_HOGGED	11	/* GPIO is hogged */

	/* Connection label */
	const char		*label;
	/* Name of the GPIO */
	const char		*name;
};
```

# 内核 api

```

// include/linux/gpio.h

// CONFIG_GPIOLIB 内核配置

// gpio 申请
int devm_gpio_request(struct device *dev, unsigned gpio, const char *label);
int devm_gpio_request_one(struct device *dev, unsigned gpio,
			  unsigned long flags, const char *label);
void devm_gpio_free(struct device *dev, unsigned int gpio);

static inline int gpio_to_irq(unsigned int gpio) // gpio转换成IRQ号
static inline int gpio_get_value(unsigned int gpio) 
static inline void gpio_set_value(unsigned int gpio, int value)
```
