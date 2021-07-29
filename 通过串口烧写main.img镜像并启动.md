# </center>通过串口烧写main.img镜像并启动<center>

## 1.流程

1.初始化GPIO，qspi和buffer。

2.延时最大5秒等待串口传输数据，如果有数据来延时提前结束，进入烧写流程；如果5秒之内没有数据则跳过烧写阶段执行默认preloader启动流程。

3.烧写main.img：通过串口接收main.img烧写到qspi中；

4.从qspi中读取main.img镜像到加载地址；

5.从加载地址启动main.img



## 2.代码

```c
#ifdef SPL_YESHEN_TEST
	/* code here */
	yeshen_mdelay(2000);
	yeshen_puts("It's yeshen show time\n");
	
	yeshen_gpio_init(); //初始化GPIO
	yeshen_qspi_init(); //初始化qspi
	yeshen_mem_init();	//初始化存放从串口接收数据后存放的内存，清0
	
	led17_on(); 		//led 17 点亮
	yeshen_mdelay(5000);
	led17_off();		//在5秒延时如果没有数据打断，则跳过，不烧写镜像，执行默认加载启动
						//否则执行烧写main.img或fpga镜像，然后启动
		
			
#ifdef LOAD_MAIN		
	if (yeshen_flag) {
		yeshen_puts("begin to load main form uart\n");
		yeshen_uart_to_qspi_load_main();		//通过串口接收main.img烧写到qspi中
		
		/* 烧写完毕后会闪5下led 17 */
		s32 i = 0;
		while (i < 8) {
			yeshen_mdelay_nobreak(500);
			led17_blink();
			i++;
		}
		yeshen_gpio_close();
	} 
#endif
				
#endif

...
    
	spl_spi_load_image();		/* 从qspi加载镜像 */

...
    
	jump_to_image_no_args(&spl_image);		/* 启动镜像 */

```



## 3.现象

![](E:\叶神文档\Markdown及其pdf\pictures\通过串口烧写main.img镜像并启动1.PNG)

<center>图1 串口工具发送窗口</center>

![](E:\叶神文档\Markdown及其pdf\pictures\通过串口烧写main.img镜像并启动2.PNG)

<center>图2 </center>

将main.img通过串口工具发送到soc，从图1可以看出发送的文件大小为239464字节大小，图二为soc的打印调试信息，接收到的数据也是239464字节大小，没有丢失，并且接收完毕后正常启动main.img，使main.img控制的led灯正常闪烁。