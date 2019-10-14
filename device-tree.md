# Device Tree
linux 系统中用来描述硬件的一个文件。

# Device Tree 语法

device tree 由多个节点组成。
```

#address-cells = <1>;  // 控制 子节点中 reg 中元素的个数
#size-cells    = <1>;  // 控制 字节点中 reg 中元素用几个u32数表示 

gic: interrupt-controller@1013d000 {
		compatible = "arm,cortex-a9-gic";
		interrupt-controller;
		#interrupt-cells = <3>; // 
		reg = <0x1013d000 0x1000>, // address = 0x1013d000 size = 0x1000
		      <0x1013c100 0x0100>; // address = 0x1013c100 size = 0x1000
	};
```
