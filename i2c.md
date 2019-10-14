# I2C 协议
  i2c协议基于SCL、SDL两根线进行通信，i2c 设备分为主设备和从设备，
  一般通信从主设备开始发送起始信号，然后在发送8bit的数据，其中7bit
  表示寄存器信息，1bit表示读写，每次通信结束会发送一个结束信号。

  起始信号： SCL 为高电平时，SDL 由高电平变为低电平。
  结束信号： SCL 为高电平时，SDL 由低电平变为高电平。

 注意： SDL需要在SCL为高电平时保持稳定，在SCL为低电平时，SDL才可以进行电平
切换。

# Linux IIC 驱动

linux 驱动是基于device和driver模型。

在一块板子有可能存在多个i2c控制器，那麽该如何确定从设备是挂载到哪个主设备下呢？
linux 内核提供了 i2c_adapter 来解决。

主要分为主设备驱动和从设备驱动。

```
// i2c 驱动主要数据结构
struct i2c_msg;
struct i2c_algorithm;
struct i2c_adapter;
struct i2c_client;
struct i2c_driver;
union i2c_smbus_data;
struct i2c_board_info;
enum i2c_slave_event;
typedef int (*i2c_slave_cb_t)(struct i2c_client *, enum i2c_slave_event, u8 *);
```

1. 主设备驱动
  
  对于每个IIC控制器，在IIC系统注册时,需要注册 struct i2c_adapter,并且adapter需要绑定一个 struct i2c_algorithm，
  并且回调用i2c_add_adapter进行注册。

  定义一个struct platform_dirver 主设备平台驱动，然后 platform_register_driver注册到platform_bus中。
  在注册到platform总线中之后，会调用 probe 函数 初始化IIC主控设备，初始化一个i2c_adapter,通过i2c_add_adapter注册。
  i2c_add_adapter 会 在i2c_bus总线下device链表中注册相应的device设备。

  这样在注册slave驱动时，只需要将i2c_driver注册到i2c_bus下的链表中。
#### i2c_add_adapter ---> of_i2c_register_device ---> i2c_new_device 注册i2c_client,并且会绑定of_node节点，一边slaver在注册时进行匹配。

通过设备树和i2c_adapter向i2c_bus 下面的devices 链表注册struct device.

2. slave 从机驱动
  
  slave 驱动主要是注册一个 struct i2c_driver,通过i2c_register_driver.

```
/**
 * struct i2c_driver - represent an I2C device driver
 * @class: What kind of i2c device we instantiate (for detect)
 * @attach_adapter: Callback for bus addition (deprecated)
 * @probe: Callback for device binding
 * @remove: Callback for device unbinding
 * @shutdown: Callback for device shutdown
 * @alert: Alert callback, for example for the SMBus alert protocol
 * @command: Callback for bus-wide signaling (optional)
 * @driver: Device driver model driver
 * @id_table: List of I2C devices supported by this driver
 * @detect: Callback for device detection
 * @address_list: The I2C addresses to probe (for detect)
 * @clients: List of detected clients we created (for i2c-core use only)
 *
 * The driver.owner field should be set to the module owner of this driver.
 * The driver.name field should be set to the name of this driver.
 *
 * For automatic device detection, both @detect and @address_list must
 * be defined. @class should also be set, otherwise only devices forced
 * with module parameters will be created. The detect function must
 * fill at least the name field of the i2c_board_info structure it is
 * handed upon successful detection, and possibly also the flags field.
 *
 * If @detect is missing, the driver will still work fine for enumerated
 * devices. Detected devices simply won't be supported. This is expected
 * for the many I2C/SMBus devices which can't be detected reliably, and
 * the ones which can always be enumerated in practice.
 *
 * The i2c_client structure which is handed to the @detect callback is
 * not a real i2c_client. It is initialized just enough so that you can
 * call i2c_smbus_read_byte_data and friends on it. Don't do anything
 * else with it. In particular, calling dev_dbg and friends on it is
 * not allowed.
 */
struct i2c_driver {
	unsigned int class;

	/* Notifies the driver that a new bus has appeared. You should avoid
	 * using this, it will be removed in a near future.
	 */
	int (*attach_adapter)(struct i2c_adapter *) __deprecated;

	/* Standard driver model interfaces */
	int (*probe)(struct i2c_client *, const struct i2c_device_id *);
	int (*remove)(struct i2c_client *);
	
  /* driver model interfaces that don't relate to enumeration  */
	void (*shutdown)(struct i2c_client *);

	/* Alert callback, for example for the SMBus alert protocol.
	 * The format and meaning of the data value depends on the protocol.
	 * For the SMBus alert protocol, there is a single bit of data passed
	 * as the alert response's low bit ("event flag").
	 */
	void (*alert)(struct i2c_client *, unsigned int data);

	/* a ioctl like command that can be used to perform specific functions
	 * with the device.
	 */
	int (*command)(struct i2c_client *client, unsigned int cmd, void *arg);

	struct device_driver driver;
	const struct i2c_device_id *id_table;

	/* Device detection callback for automatic device creation */
	int (*detect)(struct i2c_client *, struct i2c_board_info *);
	const unsigned short *address_list;
	struct list_head clients;
};

```

