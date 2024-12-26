## <center>r5f软件说明</center>

[TOC]



### 一、目录结构

```c
[zye@cr2 r5f]$ ls
asm  build  CMakeLists.txt  docs  include  README.md  scripts  src  tests  TODO.md
```

&emsp;&emsp;asm是启动汇编代码，tests中的是汇编启动后进入的main裸机应用程序，tests的每个子目录对应一个应用程序。应用程序需要的头文件和一些函数在include和src中。



### 二、编译

&emsp;&emsp;编译方法根据README.md中的指令来进行：

```c
mkdir build
cd build
cmake3 .. -DCMAKE_TOOLCHAIN_FILE=../scripts/cmake/cc.cmake
make
```

&emsp;&emsp;编译完后就在r5f/build目录下得到了对应r5f/tests目录源代码编写的应用程序。比如r5f/build/tensor_engine/tensor_engine.bin对应的是r5f/tests/tensor_engine目录源代码所编译的裸机应用程序。通过a55启动tensor_engine.bin即可把tensor_engine.bin在r5f上跑起来。

&emsp;&emsp;**如何新加一个应用程序：**

```c
1.在r5f/tests/ 下新建一个子目录demo和主源文件demo.c，主源文件即main函数的源文件，主源文件前缀必须和子目录同名。
2.在子目录r5f/tests/demo下创建CMakeLists.txt，内容为：
include(${CMAKE_CURRENT_SOURCE_DIR}/../../scripts/cmake/CMakeLists.txt)
3.在父目录r5f/tests的CMakeLists.txt后面加上add_subdirectory(demo)
```



### 三、启动流程

&emsp;&emsp;a55端的linux应用程序通过te/ve驱动将r5f裸机应用程序加载到atcm，复位启动r5f，然后r5f就开始从加载的裸机应用程序第一条指令开始执行。对应的代码是asm/start.S中的__start:标号处的b reset_handler指令。启动汇编主要做的是设置好c语言内存环境栈、data和bss段，然后就跳到裸机应用程序的main中运行c代码对应的指令。



### 四、内存和中断资源

#### 内存资源

&emsp;&emsp;r5f的内存资源排布和使用可以参考链接脚本文件r5f/src/gcc.ld：

```c
/*
    memory layout of R5F:
    ATCM: 16KB 0x00000-0x03FFF size 0x4000
    B0TCM 32KB 0x10000-0x17FFF size 0x8000
    B1TCM 32KB 0x18000-0x1FFFF size 0x8000
*/

/*
    memory usage layout of ATCM:
    ROM:  13.5KB 0x00000-0x035FF size 0x3600 [vector & text]
    RAM    0.5KB 0x03600-0x037FF size 0x0200 [data]
    stack    2KB 0x03800-0x03FFF size 0x0800 [stack]
*/

...

/* memory usage layout of stack:
 * 除了irq模式都会切换到system/user模式下,其他模式的栈没啥用
 */
USER_SP  = 0x00003c00;	/* 3800-3c00(1kb) */
IRQ_SP   = 0x00003e00;	/* 3c00-3e00(512b)*/
FIQ_SP   = 0x00003e40;	/* 3e00-3e40(64b) */
SVC_SP   = 0x00003e80;	/* 3e40-3e80(64b) */
ABORT_SP = 0x00003ec0;	/* 3e80-3ec0(64b) */
UNDEF_SP = 0x00003f00;	/* 3ec0-3f00(64b) */

...
```

&emsp;&emsp;从该文件中可以知道r5f有一块16kb的ATCM内存，地址为0x00000-0x03FFF，两块32kb的btcm内存，地址分别为0x10000-0x17FFF(B0TCM)和0x18000-0x1FFFF(B1TCM)。其中ATCM是裸机程序代码运行所在的内存，函数，变量和栈都在上面，而两个BTCM是存放r5f和a55通信数据或者调试信息的内存。因此ATCM不可以随意读写，而BTCM可以根据需要按规定读写。

#### 中断资源

&emsp;&emsp;中断分为两大类，分别是4个分别代表一个通道的mailbox中断和32个其他中断，之所以分为两大类是以为mailbox中断和这32个其他中断不共用寄存器，所以中断来了分开处理，先处理mailbox再处理其他32个中断。这些中断和寄存器详细信息可参考**R5F Subsystem user guide v1.1.2.1.pdf**。
&emsp;&emsp;在r5f代码中mailbox的处理如下：

```c
1.a55通过某个通道发送mailbox信息给r5f。
2.r5f触发中断最终进入到中断处理函数irq_c_handler。
3.irq_c_handler首先调用mailbox_isr，先通过check_mailbox_interrupt读取mailbox的中断状态寄存器，然后通过该寄存器的值轮询0-3通道，如果某个通道确实来了中断则通过clear_mailbox_interrupt清掉该通道的mailbox中断最后通过send_msg_to_a55给a55回一个消息。
4.应用程序比如tensor_engine.bin在轮询等待mailbox消息，因为r5f在收到a55mailbox消息进入中断处理函数的mailbox_isr中将触发中断的通道的标记位置1，所以此时tensor_engine.bin会进入循环体调用te_msg_handler处理a55发送的命令，如处理算子或者内存拷贝。
```

&emsp;&emsp;其他32个中断处理如下：

```c
1.首先应用程序通过irq_vect_register注册某个中断的回调函数，这些中断的编号在interrupt.h中。
2.然后在做好准备工作后通过unmask_r5f_intr打开某个中断，对于定时器在unmask_r5f_intr之后还要unmask_interrupt才能打开定时器中断。
3.相应事件触发了注册的中断，进入中断处理函数irq_c_handler。
4.irq_c_handler还是先调用mailbox_isr，但是发现mailbox的中断状态寄存器没有置位,所以退出mailbox_isr进入下一步（可能有mailbox和所注册的中断都有的特殊情况，那就先处理mailbox再处理其他），先通过get_r5f_intr_state读取这32个中断的状态寄存器，然后从0-31号中断轮询处理，当处理到了所注册的中断，就会通过irq_cb调用到注册的回调函数。
5.通常在应用程序主函数中如果不再使用某个中断，则通过mask_r5f_intr屏蔽它，对应定时器还要mask_interrupt，最后通过irq_vect_unregister注销中断回调函数（其实可以不做，因为下次注册会自动覆盖，只是这样比较规范）。
```



