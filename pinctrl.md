# pinctrl

从硬件设备来对gpio来进行分组管理

比如 IIC SDL SCL 分成一组名字为IIC。
这IIC 组就已经配置好了SDL SCL 对应的GPIO。

如果要想使用只需要查找到IIC组，取出相应的IIC gpio设置就行了。

```
struct pinctrl_desc
struct pinctrl_pins_desc
struct pinctrl_dev
// 加入到pinctrl_dev_list

//每次驱动匹配时 自动设置pins
/*
*
* probe
*    ----> really_probe()
*                   ----> pinctrl_bind_pins
*/


/**
 * pinctrl_bind_pins() - called by the device core before probe
 * @dev: the device that is just about to probe
 */
int pinctrl_bind_pins(struct device *dev)
{
	int ret;

	dev->pins = devm_kzalloc(dev, sizeof(*(dev->pins)), GFP_KERNEL);
	if (!dev->pins)
		return -ENOMEM;

	dev->pins->p = devm_pinctrl_get(dev);
	if (IS_ERR(dev->pins->p)) {
		dev_dbg(dev, "no pinctrl handle\n");
		ret = PTR_ERR(dev->pins->p);
		goto cleanup_alloc;
	}

	dev->pins->default_state = pinctrl_lookup_state(dev->pins->p,
					PINCTRL_STATE_DEFAULT);
	if (IS_ERR(dev->pins->default_state)) {
		dev_dbg(dev, "no default pinctrl state\n");
		ret = 0;
		goto cleanup_get;
	}

	dev->pins->init_state = pinctrl_lookup_state(dev->pins->p,
					PINCTRL_STATE_INIT);
	if (IS_ERR(dev->pins->init_state)) {
		/* Not supplying this state is perfectly legal */
		dev_dbg(dev, "no init pinctrl state\n");

		ret = pinctrl_select_state(dev->pins->p,
					   dev->pins->default_state);
	} else {
		ret = pinctrl_select_state(dev->pins->p, dev->pins->init_state);
	}

	if (ret) {
		dev_dbg(dev, "failed to activate initial pinctrl state\n");
		goto cleanup_get;
	}

#ifdef CONFIG_PM
	/*
	 * If power management is enabled, we also look for the optional
	 * sleep and idle pin states, with semantics as defined in
	 * <linux/pinctrl/pinctrl-state.h>
	 */
	dev->pins->sleep_state = pinctrl_lookup_state(dev->pins->p,
					PINCTRL_STATE_SLEEP);
	if (IS_ERR(dev->pins->sleep_state))
		/* Not supplying this state is perfectly legal */
		dev_dbg(dev, "no sleep pinctrl state\n");

	dev->pins->idle_state = pinctrl_lookup_state(dev->pins->p,
					PINCTRL_STATE_IDLE);
	if (IS_ERR(dev->pins->idle_state))
		/* Not supplying this state is perfectly legal */
		dev_dbg(dev, "no idle pinctrl state\n");
#endif

	return 0;

	/*
	 * If no pinctrl handle or default state was found for this device,
	 * let's explicitly free the pin container in the device, there is
	 * no point in keeping it around.
	 */
cleanup_get:
	devm_pinctrl_put(dev->pins->p);
cleanup_alloc:
	devm_kfree(dev, dev->pins);
	dev->pins = NULL;

	/* Only return deferrals */
	if (ret != -EPROBE_DEFER)
		ret = 0;

	return ret;
}


```

## device Tree 

在device Tree 中pinctrl节点中描述相关的管脚信息。
按照不同功能进行分组配置。

```

// pinctrl 节点中配置

	pinctrl: pinctrl {
		compatible = "rockchip,rk3288-pinctrl";
		rockchip,grf = <&grf>;
		rockchip,pmu = <&pmu>;
		#address-cells = <1>;
		#size-cells = <1>;
		ranges;
    
    ...

		i2c0 {
			i2c0_xfer: i2c0-xfer {
	    /*
	    * the binding format is rockchip,pins = <bank pin mux CONFIG>,
	    * do sanity check and calculate pins number
	    */
				rockchip,pins = <0 15 RK_FUNC_1 &pcfg_pull_none>,
						<0 16 RK_FUNC_1 &pcfg_pull_none>;
			};
		};

    ...


    };

  
	// pinctrl 配置后使用
	
	i2c0: i2c@ff650000 {
		compatible = "rockchip,rk3288-i2c";
		reg = <0xff650000 0x1000>;
		interrupts = <GIC_SPI 60 IRQ_TYPE_LEVEL_HIGH>;
		#address-cells = <1>;
		#size-cells = <0>;
		clock-names = "i2c";
		clocks = <&cru PCLK_I2C0>;
		pinctrl-names = "default";
		pinctrl-0 = <&i2c0_xfer>;
		status = "disabled";
	};

```
