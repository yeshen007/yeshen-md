# </center>preloader烧写镜像主流程<center>

## 1.流程

1.初始化GPIO，qspi和buffer。

2.延时最大5秒等待串口传输数据，如果有数据来延时提前结束，进入烧写流程；如果5秒之内没有数据则跳过烧写阶段执行默认preloader启动流程。

3.烧写main.img：通过串口接收main.img烧写到qspi中；

4.烧写fpga镜像：通过串口接收fpga镜像烧写到qspi中；

5.配置启动fpga



## 2.代码

```c
#ifdef SPL_YESHEN_TEST
		/* code here */
		yeshen_mdelay(1000);
		yeshen_gpio_init();		//初始化GPIO
		yeshen_qspi_init();		//初始化qspi
		yeshen_mem_init();		//初始化存放从串口接收数据后存放的内存，清0
		
		yeshen_puts("It's yeshen show time\n");
		led17_on();				//led 17 点亮
		yeshen_mdelay(5000);
		led17_off();			/* 在5秒延时如果没有数据打断，则跳过，不烧写镜像，执行默认加载启动
								   否则执行烧写main.img或fpga镜像，然后启动 */
	
#ifdef LOAD_MAIN
		if (yeshen_flag) {
			yeshen_uart_to_qspi_load_main();	//通过串口接收main.img烧写到qspi中
			
			s32 i = 0;
			/* 烧写完毕后会闪5下led 17 */
			while (i < 5) {
				yeshen_mdelay_nobreak(500);
				led17_blink();
				i++;
			}
			yeshen_gpio_close();
		} 
#endif
	
#ifdef LOAD_FPGA
		if (yeshen_flag) {
			yeshen_uart_to_qspi_load_fpga();	//通过串口接收fpga镜像烧写到qspi中
			
			s32 i = 0, test_y;
			/* 烧写完毕后会闪5下led 17 */
			while (i < 5) {
				yeshen_mdelay_nobreak(500);
				led17_blink();
				i++;
			}
			yeshen_gpio_close();
	
			if (!yeshen_qspi_load_rbf())		/* 从qspi中加载fpga固件到ddr */
				yeshen_puts("load fpga rbf is ok!\n");
			
			if ((test_y = fpgamgr_program_fpga((const unsigned long *)YESHEN_FPGA_RBF_BASE,
														YESHEN_QSPI_FPGA_SIZE)) < 0)		/* 从ddr中配置fpga */
				printf("config fpga failed!return %d\r\n",test_y);
				
			else 
				yeshen_puts("config fpga OK!\r\n"); 
			
		} 
#endif
		
#endif
```

