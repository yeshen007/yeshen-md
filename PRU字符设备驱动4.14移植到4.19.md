# <center>PRU字符设备驱动4.14移植到4.19</center>



**设备树节点获取修改**

​		5728的4.14内核和4.19内核的设备树稍有不同，4.14内核的pru核设备节点的绝对路径是/ocp/pruss_soc_bus@4b2a6004/pruss@0/pru@34000，而4.19内核的pru核设备节点的绝对路径是/ocp/pruss-soc-bus@4b2a6004/pruss@4b280000/pru@4b2b4000，因此驱动中获取设备节点的代码也从prunode = of_find_node_by_path("/ocp/pruss_soc_bus@4b2a6004/pruss@0/pru@34000")修改为prunode = of_find_node_by_path("/ocp/pruss-soc-bus@4b2a6004/pruss@4b280000/pru@4b2b4000")。



**拷贝pru固件**

​		在pru字符设备驱动中定义了如下的固件路径：

```c
static char fw_path_2_0[] = "/home/root/pruss2_pru0.bin";
static char fw_path_2_1[] = "/home/root/pruss2_pru1.bin";
```

​		因此要将这两个pru固件拷贝到4.19内核的根文件系统的/home/root目录下。



**卸载ti的相关pru驱动**

​		4.19内核启动时会加载ti的一些不需要的pru驱动，而4.14内核不会，但有几个pru驱动不能卸载，如pru中断控制器驱动和pruss驱动。为了避免每次都要手工输入大量命令卸载这些驱动，因此制作了一个脚本rm_ti_mod.sh，只要在加载pru字符设置驱动前运行即可，脚本如下：

```c		
rmmod ti_prueth
rmmod pru_rproc
rmmod omap_remoteproc
rmmod remoteproc
```

​		

