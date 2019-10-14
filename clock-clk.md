# linux clk framework
关于struct clk 结构体定义，在很多board上面没找到。
但是在达芬奇TI 下面找到struct clk定义

```
// arch/arm/match-davinci/clock.h
// 内核版本 4.9

struct clk {
	struct list_head	node;
	struct module		*owner;
	const char		*name;
	unsigned long		rate;
	unsigned long		maxrate;	/* H/W supported max rate */
	u8			usecount;
	u8			lpsc;
	u8			gpsc;
	u8			domain;
	u32			flags;
	struct clk              *parent;
	struct list_head	children; 	/* list of children */
	struct list_head	childnode;	/* parent's child list node */
	struct pll_data         *pll_data;
	u32                     div_reg;
	unsigned long (*recalc) (struct clk *);
	int (*set_rate) (struct clk *clk, unsigned long rate);
	int (*round_rate) (struct clk *clk, unsigned long rate);
	int (*reset) (struct clk *clk, bool reset);
	void (*clk_enable) (struct clk *clk);
	void (*clk_disable) (struct clk *clk);
};
```



# Rockchip

```

// 在rockchip 中clk控制配置都是在 设备树中描述的。
static void __init rk2928_gate_clk_init(struct device_node *node)
{
	struct clk_onecell_data *clk_data;
	const char *clk_parent;
	const char *clk_name;
	void __iomem *reg;
	void __iomem *reg_idx;
	int flags;
	int qty;
	int reg_bit;
	int clkflags = CLK_SET_RATE_PARENT;
	int i;

	qty = of_property_count_strings(node, "clock-output-names");
	if (qty < 0) {
		pr_err("%s: error in clock-output-names %d\n", __func__, qty);
		return;
	}

	if (qty == 0) {
		pr_info("%s: nothing to do\n", __func__);
		return;
	}

	reg = of_iomap(node, 0);

	clk_data = kzalloc(sizeof(struct clk_onecell_data), GFP_KERNEL);
	if (!clk_data)
		return;

	clk_data->clks = kzalloc(qty * sizeof(struct clk *), GFP_KERNEL);
	if (!clk_data->clks) {
		kfree(clk_data);
		return;
	}

	flags = CLK_GATE_HIWORD_MASK | CLK_GATE_SET_TO_DISABLE;

	for (i = 0; i < qty; i++) {
		of_property_read_string_index(node, "clock-output-names",
					      i, &clk_name);

		/* ignore empty slots */
		if (!strcmp("reserved", clk_name))
			continue;

		clk_parent = of_clk_get_parent_name(node, i);

		/* keep all gates untouched for now */
		clkflags |= CLK_IGNORE_UNUSED;

		reg_idx = reg + (4 * (i / 16));
		reg_bit = (i % 16);

		clk_data->clks[i] = clk_register_gate(NULL, clk_name, // 最终调用 clk_register注册到 clocks 链表中。
						      clk_parent, clkflags,
						      reg_idx, reg_bit,
						      flags,
						      &clk_lock);
		WARN_ON(IS_ERR(clk_data->clks[i]));
	}

	clk_data->clk_num = qty;

	of_clk_add_provider(node, of_clk_src_onecell_get, clk_data);
}

//定义__clk_of_table section，在 clk_init 中会回调rk_2928_gate_clk_init 初始化
CLK_OF_DECLARE(rk2928_gate, "rockchip,rk2928-gate-clk", rk2928_gate_clk_init);
```


