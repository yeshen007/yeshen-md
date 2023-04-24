## <center>spacc驱动小结</center>

[TOC]

### 一. 传统设备节点方式

#### 1.1 概括

&emsp;&emsp;这种方式是通过应用程序打开设备节点，然后通过ioctl设置密钥和其他相关参数，最后启动作业。这是spacc驱动包提供的方式。  
&emsp;&emsp;spacc驱动包主要包含以下几个内核模块：

- elppdu.ko ：由pdu.c和一些辅助代码编译得到，它给其他驱动模块提供了内存、DMA的分配和操作，资源锁定释放等通用的api。这些api通常是linux内核其他函数的包装，如pdu_io_write32是对writel的包装，PDU_LOCK宏扩展就是spin_lock_irqsave。
- elpmem.ko ：由spacc_mem.c编译得到，是一个平台设备驱动模块，它主要做的首先是创建平台设备，根据在Makefile中配置的PDU_BASE_ADDR和PDU_BASE_IRQ生成资源，读取sapcc硬件寄存器填入pdu_info结构中，将资源和pdu_info添加平台设备并将平台设备注册到平台设备总线。
- elspacc.ko ：由spacc.c和其他辅助代码编译得到，封装和提供了对加密硬件的底层操作，比如作业的管理，中断的管理，密钥上下文的设置等。elspacc.ko 注册的平台驱动会和elpmem.ko匹配，然后提取出平台设备的资源和pdu_info，根据这些数据和对硬件相应的读取设置操作初始化一个spacc_device结构体，该结构体包含除了根据作业需要临时传入设置参数之外的所有的信息。
- elspaccusr.ko ：由spacc_dev.c编译得到，它注册了一个字符设备*/dev/spaccusr*，应用层可以打开操作这个设备节点进行加解密作业，主要是通过调用ioctl并通过传入适当的参数来设置和启动加解密操作。该字符设备的ioctl驱动接口使用elspacc.ko中的函数进行设置和作业。

#### 1.2 启动作业流程

```c
/* 
 * 申请一个作业和一个关联它的上下文,设置密钥,iv,加密模式等
 */
spacc_dev_open
	open("/dev/spaccusr")
		spacc_dev_open
	ioctl(ELP_SPACC_USR_INIT)  	
		spacc_open
		spacc_write_context
		spacc_set_operation

/*
 * 设置数据源地址和目的地址,启动作业,等待作业完成
 */
spacc_dev_process
	ioctl(ELP_SPACC_USR_PROCESS)
		init_completion
		spacc_packet_enqueue_ddt
		wait_for_completion_interruptible
		spacc_packet_dequeue

/*
 * 释放作业和关联的上下文
 */
spacc_dev_close    
	close(fd)
		spacc_dev_release
			spacc_close
```

&emsp;&emsp;以上是启动一个简要作业的流程，其中第一级和第二级缩进属于应用层函数，第三级和第四级属于内核层函数。第一级spacc_dev_xx函数是对第二级直接操作设备节点的一层封装，方便用户的使用。无论是使用spacc_dev_xx还是直接操作设备节点最终都要落实到下面的内核调用流程上。

```c
//设置加解密模式(如AES_CBC),申请一个作业并关联一个上下文和设置回调函数
spacc_open
//将key和iv写入上下文
spacc_write_context
//加解密方向和其他设置
spacc_set_operation
//初始化一个completion
init_completion
//传入src和dst,分配swid关联到作业并设置到硬件,启动作业
spacc_packet_enqueue_ddt
//等待completion
wait_for_completion_interruptible
===========================================================================================
//取出硬件stat fifo队列中的完成作业并调用作业的回调函数
spacc_irq_handler
	spacc_process_irq
		spacc_stat_process			//sapcc_device->irq_cb_stat()
			spacc_pop_jobs
				spacc_pop_packets
					spacc_pop_packets_ex
						complete	//job->cb()
===========================================================================================
//返回作业完成的状态
spacc_packet_dequeue
//释放作业和关联的上下文
spacc_close
```

&emsp;&emsp;以上流程是内核中启动一个任务并等待一个任务完成的过程，其中两条===线之间的部分是在启动作业后在作业完成后触发的中断所做的事。中断函数主要负责清中断，取出stat fifo队列中的完成作业，并调用作业的回调函数，这里的作业回调函数是complete，因此会唤醒用wait_for_completion_interruptible等待的任务，任务唤醒后可以通过spacc_packet_dequeue查看作业完成的状态，最后可以通过spacc_close释放作业和关联的上下文或者可以重新设置数据密钥进行下一轮作业。



### 二. linux crypto api方式