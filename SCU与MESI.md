# <center> SCU与MESI协议</center>

#### 一、A9MPU SCU

​		在A9 MPU和A15 MPU中都有一个监听控制单元(Snoop Control Unit)，简称SCU。这个硬件的功能是为处理器和内存系统管理数据交互，保证处理器操作的数据是最新的，维护缓存一致性。
![](pictures\scu.PNG)		如上图所示，A9 MPU的SCU连接了cpu和acp到L2，acp是一个外设通道，外设通过它来访问soc的公共内存系统。因为L2是A9 MPU外部接的一个缓存，不是L1这种内嵌cpu中的，因此L2和ddr都属于soc的公共内存。当处理器使能了smp模式时，scu开启维护数据一致性，它不维护icache的一致性。
​		当处理器写某处内存时，SCU负责处理和记录好缓存在L1或L2中的数据和状态，比如因为写而更新了本核的L1，则SCU会同步另一核中的L1的数据或者将它失效掉。当一个核读某处内存时，如果在本核的L1没有有效数据，则SCU查看别的核的L1是否是最新的有效数据，如果是直接返回给本核并更新本核的L1，如果没有SCU将读发送到下一级的L2中。
​		外设通过acp通道访问内存时也会通过SCU，但这个只是单向一致性维护，外设可以看到cpu核的最新数据而cpu看不到外设中的最新数据(如果外设也有缓存的话)。以下是通过acp的外设的可能的读写情况：

- acp外设读的时候最新数据在某个核的L1中：从该核的L1中获取最新数据。
- acp外设读的时候最新数据在L2中不在L1中：从L2中获取最新数据。
- acp外设读的时候最新数据即不在L1也不在L2中：从ddr中获取最新数据。
- acp外设写的时候在某个核的L1中缓存着有效数据：将该L1行失效，并驱逐到L2，然后acp外设的数据也会写到L2覆盖掉被驱逐下来的数据，但不会写到L1。接下来如果该核访问该内存时会在L1命中缺失。
- acp外设写的时候在L2中缓存着有效数据，L1中没有：如果是一致性数据，则将数据写入L2，如果不是先驱逐到ddr，再写入L2。
- acp外设写的时候有效数据既不在L1也不在L2中：将数据写入L2。

以上情况稍作补充：有效数据包括一致性数据和非一致性数据，非一致性数据意思是不同的内存位置却映射到同一缓存位置，一致性数据指的是内存位置一致。特别要注意以上的情况实际上除了倒数第2条都没有明确区分有效数据和一致性数据，因此需要根据语境区分。



#### 二、一致性协议MESI

​		一致性意思是所有核都能访问到最新的数据。意味着更改某个核的缓存的数据会让其他核可见，不会让其他核看到的是老的数据。一致性主要有**写失效**和**写广播**两种，MESI协议是**写失效**协议。mesi协议要实现两个机制：**第一**，cpu核心对数据的操作要同步到其他cpu核心。**第二**，如果有多个核心具有同一数据的cache，那么需要类似锁的操作。以上两个机制结合起来就能得到完整的cache一致性。
​		缓存一致性通过某种协议来维护。一般有MOESI或者MESI协议，a9 mpu的scu采用MESI。a9 mpu的SCU标记每个核的缓存行可能的M,E,S,I这四种状态之一。

- **M：代表已修改（Modified）** 

- **E：代表独占（Exclusive）**		

- **S：代表共享（Shared）**			

- **I：代表已失效（Invalid）**		

  **已修改**表示的是脏的缓存行，即该缓存行中的数据比对应内存中的数据新，还没有写回内存。

  **已失效**表示该缓存行已经失效。

  **独占**和**共享**相对比较复杂，是精髓所在。无论是独占还是共享都表示该行是**干净**的，意思是和内存数据是一致的。区别在于独占的情况下该行只加载到当前cpu核的cache，没有加载到别的核中，因此向独占的缓存行中写数据不需要告知其他核；而共享的情况下，该行数据也缓存在其他cpu的cache中，因此不能直接写里面的数据，需要先向其他cpu广播一个请求，要其他缓存该行数据的cpu将自己对应的行标记为无效，然后再修改数据。独占的缓存行在收到来自总线的其他核读取该行数据的请求会把该行标记为共享，因为读取该行的核也要在缓存中加载这行数据。
  
  

**状态机：**

![](pictures\MESI.jpg)

​		图中的**本地读/写**不同于其他的类似于**总线读/发出写回信号**的东西，**本地读/写**意思是本地读或者写，没有关联；而**总线读/发出写回信号**表示先收到总线读，然后发出写回信号，有前后的因果关系。