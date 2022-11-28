## <center>arm-linux多核设置</center>

```c
/* step 1
 * 设置全局变量smp_ops为platsmp.c中定义的socfpga_smp_ops
 * 调用socfpga_smp_init_cpus来根据硬件和软件配置来设置possible位图和ipi函数
 */
start_kernel()
    setup_arch()
    	if (is_smp()) {
			smp_set_ops(mdesc->smp);		
				smp_ops = *ops;          	 //platsmp.c中定义socfpga_smp_ops 
			smp_init_cpus();			
         		 smp_ops.smp_init_cpus()	//socfpga_smp_init_cpus
		}

/* step 2 
 * 设置present位图,不会超过possible,是能scu,启动present位图中所有的核
 * 注意amp中副核已经启动,所以smp_init()中啥也没做
 * 在smp中会启动副核并且设置online位图
 */
start_kernel()
    rest_init()
    	kernel_init()
    		kernel_init_freeable()
    			//设置不超过possible的present位图
    			//调用socfpga_smp_prepare_cpus使能scu
    			smp_prepare_cpus(setup_max_cpus)	
    			//启动present中所有的核
    			smp_init()
    				for_each_present_cpu(cpu) {
    					if (!cpu_online(cpu))	
        					cpu_up(cpu)	
    							_cpu_up(cpu, 0)
                                       __cpu_up(cpu, idle)
					}
    				
/* step 2 last substep
 * 在amp中多了amp_init()这一个步骤,设置amp共享内存中的函数给bm用,而boot_secondary直接返回
 * 在smp中副核复位后会跑secondary_startup,最后会进入secondary_start_kernel调用
 * set_cpu_online(cpu, true)设置online位图
 */
__cpu_up(cpu, idle)
    //共享内存区域填入linux的函数,然后bm可以直接使用
	amp_init();	
		asp = (struct amp_share_param *)AMP_SHARE_PARAM_START;
         asp->sta.printk_fn = (u32)printk;
         asp->sta.spinlock_trylock_fn = (u32)_raw_spin_trylock;
         asp->sta.spinlock_lock_fn  = (u32)_raw_spin_lock;
         asp->sta.spinlock_unlock_fn = (u32)_raw_spin_unlock;
         asp->sta._down_trylock_fn = (u32)down_trylock; 
         asp->sta._up_fn			= (u32)up;
	boot_secondary(cpu, idle);	
		smp_ops.smp_boot_secondary(cpu, idle)	//socfpga_boot_secondary
			return -1						//直接返回,因为uboot已经启动副核跑bm了
		
```
