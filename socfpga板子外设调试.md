## <center>socfpga板子外设调试</center>

**参考文档：**

Documentation/devicetree/bindings/leds/leds-gpio.txt
Documentation/devicetree/bindings/gpio/gpio-altera.txt

### **步骤：**

#### 1.修改添加pinctrl和gpio设备树

注：altera的引脚复用和配置不用pintcrl，由硬件工程师用quar生成文件给uboot配置。

```c
/ {
    /* 平台设备soc */
    soc {
        ...
		gpio0: gpio@ff708000 {
			#address-cells = <1>;
			#size-cells = <0>;
			compatible = "snps,dw-apb-gpio";
			reg = <0xff708000 0x1000>;
			clocks = <&l4_mp_clk>;
			resets = <&rst GPIO0_RESET>;
			status = "disabled";

			porta: gpio-controller@0 {
				compatible = "snps,dw-apb-gpio-port";
				gpio-controller;
				#gpio-cells = <2>;
				snps,nr-gpios = <29>;
				reg = <0>;
				interrupt-controller;
				#interrupt-cells = <2>;
				interrupts = <0 164 4>;
			};
		};

		gpio1: gpio@ff709000 {
			#address-cells = <1>;
			#size-cells = <0>;
			compatible = "snps,dw-apb-gpio";
			reg = <0xff709000 0x1000>;
			clocks = <&l4_mp_clk>;
			resets = <&rst GPIO1_RESET>;
			status = "disabled";

			portb: gpio-controller@0 {
				compatible = "snps,dw-apb-gpio-port";
				gpio-controller;
				#gpio-cells = <2>;
				snps,nr-gpios = <29>;
				reg = <0>;
				interrupt-controller;
				#interrupt-cells = <2>;
				interrupts = <0 165 4>;
			};
		};

		gpio2: gpio@ff70a000 {
			#address-cells = <1>;
			#size-cells = <0>;
			compatible = "snps,dw-apb-gpio";
			reg = <0xff70a000 0x1000>;
			clocks = <&l4_mp_clk>;
			resets = <&rst GPIO2_RESET>;
			status = "disabled";

			portc: gpio-controller@0 {
				compatible = "snps,dw-apb-gpio-port";
				gpio-controller;
				#gpio-cells = <2>;
				snps,nr-gpios = <27>;
				reg = <0>;
				interrupt-controller;
				#interrupt-cells = <2>;
				interrupts = <0 166 4>;
			};
		};
        ...
    };	
    ...
    /* 平台设备，连接led的gpio */
	leds {
		compatible = "gpio-leds";
		led0 {	//gpio53
			label = "FPGA_CFG";
			gpios = <&portb 24 1>;
		};     
		led1 {	//gpio54
			label = "Boot_State";
			gpios = <&portb 25 1>;
		};
	};
    ...
    /* 平台设备，不连接led的gpio */
	gpios {
        compatible = "altr,pio-1.0";
        gpio35 {
            label = "LOCK_CHIP_EN";
            gpios = <&portb 6 1>;
        };
        gpio44 {	//被mmc0占用了
          	label = "SDCARD_INSTRUCT" 
        	gpios = <&portb 15 1>;   
        };
        gpio48 {
          	label = "24VEN" 
        	gpios = <&portb 19 1>;    
        };  
        gpio49 {
          	label = "PH_EN" 
        	gpios = <&portb 20 1>;    
        };
        gpio50 {
          	label = "Reserve" 
        	gpios = <&portb 21 1>;    
        };  
        gpio51 {
          	label = "IC_EN" 
        	gpios = <&portb 22 1>;    
        };   
        gpio52 {
          	label = "BOARD_INDEX" 
        	gpios = <&portb 23 1>;    
        };  
        gpio55 {
          	label = "PHY1_INIT" 
        	gpios = <&portb 26 1>;    
        };
        gpio56 {
          	label = "PHY2_INIT" 
        	gpios = <&portb 27 1>;    
        };
        gpio59 {
          	label = "PCA9535_INIT" 
        	gpios = <&portc 1 1>;    
        };
        gpio61 {
          	label = "LOS_INDEX" 
        	gpios = <&portc 3 1>;    
        };
	 };
};
...
&gpio0 {
	status = "okay";
};
&gpio1 {
	status = "okay";
};
&gpio2 {
	status = "okay";
};
...
&mmc0 {
	cd-gpios = <&portb 15 0>;
	vmmc-supply = <&regulator_3_3v>;
	vqmmc-supply = <&regulator_3_3v>;
	status = "okay";
};
```



#### 2.内核配置

```c
已经默认配置了led驱动和gpio驱动
```



#### 3.编写驱动(使用平台驱动和杂项驱动)

```c
驱动已经被内核写好
```



#### 4.文件系统操作

```c
/* 查看文件系统是否加入了以上led和gpio节点 */
cd /proc/device-tree && ls 
//可以看到leds和gpios,然后对照看里面的内容和设备树一一对应

/* 操作非led的gpio */
cd /sys/class/gpio && ls
//有这些文件：export gpiochip1963  gpiochip1990  gpiochip2019  unexport
//export和unexport用来选择操作哪一个gpio引脚
//gpiochip1963，gpiochip1990，gpiochip2019分别是管理三组gpio引脚的gpio控制器，比如gpiochip1963
	cd  gpiochip1963 && ls
	//base .. label ..
		cat label	
		//ff70a000.gpio
		//ff70a000对应了设备树中的gpio2: gpio@ff70a000控制器，端口为portc
//举例操作gpio59，设备树中gpios = <&portc c 1>表明是c端口的1号gpio引脚，因此是1963+1=1964
echo 1964 > /sys/class/gpio/export	//生成gpio1964
echo in > /sys/class/gpio/gpio1964/direction //设置成输入
//echo out > /sys/class/gpio/gpio1964/direction		//设置成输出
cat /sys/class/gpio/gpio1964/value	//读取输入值
//echo N > /sys/class/gpio/gpio2040/value	//设置引脚输出值，N为0或1
echo 1964 > /sys/class/gpio/unexport	//释放

/* 操作led的gpio */
cd /sys/class/leds && ls
//Boot_State  FPGA_CFG刚好是设备树指定的两个gpio led
cat /sys/class/leds/FPGA_CFG/brightness
echo 1 > /sys/class/leds/FPGA_CFG/brightness	//点亮

/* 查看gpio占用情况 */
mount -t debugfs debugfs /sys/kernel/debug
cat /sys/kernel/debug/gpio 
```



#### 5.测试用例

##### 5.1 led-gpio测试用例

```c

```

编译

```c
arm-linux-gnueabihf-gcc led-test.c -o led-test
//拷贝到板子 ./led-test
```



##### 5.2 普通gpio测试用例




##### 5.3 特殊gpio44测试用例