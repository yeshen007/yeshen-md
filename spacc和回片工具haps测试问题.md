spacc 4g问题

```c
sg_page

page_to_phys(page)	
    (page - vmemmap)
    	(page - VMEMMAP_START + (memstart_addr >> PAGE_SHIFT)) << PAGE_SHIFT	

1.在spacc_sk_dma结构体中增加一个成员保存ddr 4g域，spacc_sk_record_init通过dma zone中分配在域0中的内存yu0buf，在spacc_crypto_probe中调用remap默认映射域0。
2.check函数检查src和dst所有的sg是不是都在一个ddr 4g域中，如果是就保存这个域，如果不是就设置成-1。
3.在check函数的父函数即spacc_sk_dma中检查src和dst的域分别处理以下九中情况。


x>0
y>0
z>=0    
x != y

//s1		!copy_src !copy_dst
src = z, dst = z	
	bass_read_remap_spacc(z);
	bass_write_remap_spacc(z);
  
//s2    	!copy_src copy_dst
src = 0, dst = y	
	bass_read_remap_spacc(0);
	bass_write_remap_spacc(0);
	//real_dst = dst
	dst = yu0buf
	//处理完后spacc_sk_done_task中sg_copy_from_buffer(real_dst, yu0buf)
	
//s3		copy_src !copy_dst
src = x, dst = 0
	bass_read_remap_spacc(0);
	bass_write_remap_spacc(0);
	copy_to_yu0buf(src)
	src = yu0buf
	
//s4		copy_src copy_dst
src = x, dst = y
	bass_read_remap_spacc(0);
	bass_write_remap_spacc(0);
	copy_to_yu0buf(src)
    src = yu0buf
    //real_dst = dst
    dst = yu0buf
    //处理完后spacc_sk_done_task中sg_copy_from_buffer(real_dst, yu0buf)
    
//s5		copy_src !copy_dst
src = -1, dst = 0	
	bass_read_remap_spacc(0); 
	bass_write_remap_spacc(0);
	copy_to_yu0buf(src)
	src = yu0buf
        
//s6    	copy_src copy_dst
src = -1, dst = y	
	bass_read_remap_spacc(0);
	bass_write_remap_spacc(0);
	copy_to_yu0buf(src) 
	src = yu0buf
	//real_dst = dst
	dst = yu0buf
	//处理完后spacc_sk_done_task中sg_copy_from_buffer(real_dst, yu0buf)
		
//s7		!copy_src copy_dst
src = 0, dst = -1
	bass_read_remap_spacc(0);
	bass_write_remap_spacc(0);
	//real_dst = dst
	dst = yu0buf
	//处理完后spacc_sk_done_task中sg_copy_from_buffer(real_dst, yu0buf)
        
//s8		copy_src copy_dst
src = x, dst = -1	
	bass_read_remap_spacc(0);
	bass_write_remap_spacc(0);
	copy_to_yu0buf(src) 
	src = yu0buf
	//real_dst = dst
	dst = yu0buf
	//处理完后spacc_sk_done_task中sg_copy_from_buffer(real_dst, yu0buf)
    
//s9    	copy_src copy_dst
src = -1, dst = -1
	bass_read_remap_spacc(0);
	bass_write_remap_spacc(0);
	copy_to_yu0buf(src) 
	src = yu0buf
	//real_dst = dst
	dst = yu0buf
	//处理完后spacc_sk_done_task中sg_copy_from_buffer(real_dst, yu0buf)  

额外需要解决：    
zone划分并划分起码两个夸4g的zone，在init.c的zone_sizes_init中
设备树内存划分设置
dma_map_sg/page_to_phys得到的物理地址不在设备树memory范围:在dma_map_sg调用链中的page_to_phys替换为spacc_page_to_phys
```



回片工具haps测试问题

```c
/* haps回片工具测试现象 */
注意：bass ip测试前先用prcm.sh使能和复位lsp和对应ip。还有下面的一些bug现象可能是工程砍掉了部分资源，或者没有更新没有跟上文档导致的。

bass dro: 寄存器不能正常读写，所有寄存器读出来都是0，并且写任何值之后读出来还是0。

bass tsensor: 寄存器不能正常读写，所有寄存器读出来都是0，并且写任何值之后读出来还是0。

bass pwm: 寄存器可以正常读写，是否能正常工作需要回片后用示波器检测。

bass rng: 寄存器可以正常读写，不能正常工作。设置seed寄存器后启动种子生成，stats寄存器一直读不到种子生成完成状态，读出随机数结果寄存器的值一直是0。

bass timer: 通道0正常工作，通道1-15无法工作，且访问寄存器会产生异常。

bass wdt: wdt0 single-expiration模式可以正常工作，Dual-expiration模式不起作用，和single-expiration模式现象一样第一次计数到了就产生复位。wdt1和wdt2可以正常访问寄存器，但是single-expiration模式和Dual-expiration模式都不会产生复位，计数到了重新导入继续递减。

bass efuse: 正常工作。

x86 spi efuse: 未测。

r5f mailbox: 基本正常，但是和sw中断有冲突，r5f收到sw中断会导致mailbox发送不到a55。

r5f wfi: 正常工作。

r5f wfe: 部分异常，现象是一次事件会穿两个wfe。

r5f timer: 正常工作。

r5f wdt: 计时模式正常工作，复位模式不能工作(文档中也提到删除了复位功能)。文档中说锁寄存器编译从0x18改到0x118是错的，实际上还是0x18,可能工程比较老没有更新,回片后再看看。

r5f cpu clock: 调频后运行速度没变，可能是haps频率固定。
```

