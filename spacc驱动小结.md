<center>spacc驱动小结</center>

[TOC]



### 一. 传统设备节点方式

#### 1.1 概括

&emsp;&emsp;设备节点方式是通过应用程序打开设备节点，然后通过ioctl设置密钥和其他相关参数，最后启动作业。**这是spacc sdk提供的方式**，spacc sdk驱动主要包含以下几个内核模块：

- **elppdu.ko** ：由pdu.c和一些辅助代码编译得到，它给其他驱动模块提供了内存、DMA的分配和操作，资源锁定释放等通用的api。这些api通常是linux内核其他函数的包装，如pdu_io_write32是对writel的包装，PDU_LOCK宏扩展就是spin_lock_irqsave。
- **elpmem.ko** ：由spacc_mem.c编译得到，是一个平台设备驱动模块，它主要做的首先是创建平台设备，根据在Makefile中配置的PDU_BASE_ADDR和PDU_BASE_IRQ生成资源，读取sapcc硬件寄存器填入pdu_info结构中，将资源和pdu_info添加平台设备并将平台设备注册到平台设备总线。
- **elspacc.ko** ：由spacc.c和其他辅助代码编译得到，封装和提供了对加密硬件的底层操作，比如作业的管理，中断的管理，密钥上下文的设置等。elspacc.ko 注册的平台驱动会和elpmem.ko匹配，然后提取出平台设备的资源和pdu_info，根据这些数据和对硬件相应的读取设置操作初始化一个spacc_device结构体，该结构体包含除了根据作业需要临时传入设置参数之外的所有需要的数据。
- **elspaccusr.ko** ：由spacc_dev.c编译得到，它注册了一个字符设备*/dev/spaccusr*，应用层可以打开操作这个设备节点进行加解密作业，主要是通过调用ioctl并通过传入适当的参数来设置和启动加解密操作。该字符设备的ioctl驱动接口使用elspacc.ko中的函数进行设置和作业。

&emsp;&emsp;**算法实现者**实现和提供算法，**算法使用者**使用算法进行作业处理。算法使用者主要关注**作业调用流程**，算法实现者既要关注**注册流程**也要关注作业调用流程。

#### 1.2 注册流程

&emsp;&emsp;上面的几个驱动模块的加载顺序是elppdu.ko，elpmem.ko，elspacc.ko，elpspaccusr.ko。  
&emsp;&emsp;**elppdu.ko**没有注册任何设备或者驱动，只提供通用api。  
&emsp;&emsp;**elpmem.ko**注册了一个平台设备，如下面简化后的代码所示：

```c
static int __init pdu_vex_mod_init(void)
{
    int irq_num = get_irq_num(PDU_BASE_IRQ);	//映射物理irq PDU_BASE_IRQ为虚拟irq irq_num
	pdu_info info;
    
    //res[0]是spacc物理基地址, res[1]是spacc虚拟irq
    struct resource res[2];	
    res[0] = (struct resource) {
        .start = PDU_BASE_ADDR,	
        .end   = PDU_BASE_ADDR,
        .flags = IORESOURCE_MEM,
    };
    res[1] = (struct resource) {
        .start = irq_num,
        .end   = irq_num,
        .flags = IORESOURCE_IRQ,
    };

    //读取spacc硬件配置信息到info
    spdu_init(PDU_BASE_ADDR, &info);	

    //注册平台设备,并记录在spacc驱动管理的devices[]数组中
    register_device("spacc", info.spacc_version.project << 16, res, 2, &info);
}

module_init(pdu_vex_mod_init);
```

&emsp;&emsp;如1.1小节所说，这里根据sapcc的物理地址和映射后得到的linux irq号生成两个struct resource，同时读取spacc硬件配置信息到pdu_info结构中，然后通过register_device创建平台设备并注册，register_device简化后代码如下，意思很明了不再细说。

```c
static struct platform_device *devices[MAX_DEV];
static int dev_id;

static void register_device(const char *name, int id,
                            const struct resource *res, unsigned num,
                            pdu_info *info)
{
    struct platform_device_info pdevinfo = {
        .name = name,		//.name = "spacc"
        .id = id,			//.id = project号码
        .res = res,			//.res = spacc物理基地址和spacc虚拟irq
        .num_res = num,		//.num_res = 2
        .data = info,		//.data = pdu_info
        .size_data = sizeof *info,
    };

	//注册进平台总线并记录在devices[]中
    devices[dev_id] = platform_device_register_full(&pdevinfo);	

    dev_id++;	
}
```

&emsp;&emsp;**elspacc.ko**注册了一个平台驱动，如下面简化后代码所示：

```c
static struct platform_driver spacc_driver = {
    .probe  = spacc_probe,
    .driver = {
        .name  = "spacc",		//和上文注册的平台设备名字一样，匹配成功调用spacc_probe
        .owner = THIS_MODULE
    },
};

static int __init spacc_mod_init (void)
{
    platform_driver_register(&spacc_driver);
}

module_init (spacc_mod_init);
```

&emsp;&emsp;注册的平台驱动名字和上文注册的平台设备名字一样，匹配成功调用spacc_probe，该函数简化后代码如下：

```c
static int __devinit spacc_probe(struct platform_device *pdev)
{
    void *baseaddr;
    struct resource *mem, *irq;
    struct spacc_priv *priv;
    pdu_info info;

    //获取平台设备的spacc物理基地址和虚拟irq
    mem = platform_get_resource(pdev, IORESOURCE_MEM, 0);	
    irq = platform_get_resource(pdev, IORESOURCE_IRQ, 0);	
    
    priv = devm_kzalloc(&pdev->dev, sizeof *priv, GFP_KERNEL);
    baseaddr = pdu_linux_map_regs(&pdev->dev, mem);	//物理基地址映射为虚拟地址
    pdu_get_version(baseaddr, &info);				//读取硬件配置信息填入info

    //初始化spacc_device结构体, 包括作业数组和上下文数组
    spacc_init(baseaddr, &priv->spacc, &info);
    
    //tasklet函数设置为spacc_pop_jobs
    tasklet_init(&priv->pop_jobs, spacc_pop_jobs, (unsigned long)priv);	
    spacc_irq_glbl_disable (&priv->spacc);	//关闭全局irq
    platform_set_drvdata(pdev, priv);		//设置pdev->dev->driver_data = priv;
    priv->max_msg_len = priv->spacc.config.max_msg_size;

    //注册irq处理函数spacc_irq_handler
    devm_request_irq(&pdev->dev, irq->start, spacc_irq_handler, IRQF_SHARED, 
                     dev_name(&pdev->dev), &pdev->dev);

    //设置中断回调函数
    priv->spacc.irq_cb_stat = spacc_stat_process;		
    priv->spacc.irq_cb_cmdx = spacc_cmd_process;		

    //设置两个fifo初始阈值并使能它们
    spacc_irq_stat_enable (&priv->spacc, 1);	
    spacc_irq_cmdx_enable(&priv->spacc, 0, 1);	
    
    spacc_irq_glbl_enable (&priv->spacc);		//使能全局irq
}
```

&emsp;&emsp;spacc_probe首先做的是从elpmem.ko注册的平台设备提取spacc寄存器物理基地址和linux irq号。然后对物理地址映射后重新读取spacc寄存器得到各种数据填充pdu_info，再通过spacc_init初始化一个spacc_device结构体，该结构体包含了启动作业临时传入的参数之外的所有需要的数据，包括硬件的状态信息和软件管理的数据如作业管理和上下文管理数组。最后是中断相关的设置，包括注册irq处理函数spacc_irq_handler，设置tasklet函数spacc_pop_jobs，设置stat和cmd fifo中断回调函数和初始fifo阈值，最后使能中断。  
&emsp;&emsp;stat和cmd fifo是spacc硬件的两个硬件fifo，用来一起配合完成作业处理，当cmd fifo下降到设定的阈值或者stat fifo上升到设定的阈值都会触发irq调用spacc_irq_handler然后调用spacc_process_irq，做些硬件中断相关的操作然后调用上面的回调函数spacc_XX_process，回调函数发起tasklet然后调用到spacc_pop_jobs，spacc_pop_jobs将stat fifo的结果取出调用对应作业的回调函数，具体调用过程参考下文的作业调用流程。

&emsp;&emsp;**elspaccusr.ko**注册了一个字符设备，提供了设备节点*/dev/spaccusr*，如下简化代码所示。应用层通过open该设备节点然后通过ioctl设置作业参数和启动作业，具体用法参考下文的作业调用流程。

```c
static struct file_operations spacc_dev_fops = {
    .owner = THIS_MODULE,
    .open           = spacc_dev_open,
    .release        = spacc_dev_release,
    .unlocked_ioctl = spacc_dev_ioctl,
};

static struct miscdevice spaccdev_device = {
    .minor = MISC_DYNAMIC_MINOR,
    .name = "spaccusr",
    .fops = &spacc_dev_fops,
};

static int __init spacc_dev_init (void)
{
    misc_register (&spaccdev_device);
}

module_init (spacc_dev_init);
```

#### 1.3 作业调用流程

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

&emsp;&emsp;以上是启动一个作业的通用流程，无论是哪种算法这个流程通用，其差别在于传入设置的参数不同。其中第一级和第二级缩进属于应用层函数，第三级和第四级属于内核层函数。第一级spacc_dev_xx函数是对第二级直接操作设备节点的一层封装，方便用户的使用。无论是使用spacc_dev_xx还是直接操作设备节点最终都要落实到下面的内核调用流程上。

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
&emsp;&emsp;这里只讲作业调用的流程，没有详细说明调用的函数的参数设置，这涉及到很多的spacc硬件架构细节和相关加密知识，自行参考spacc手册和相关加密模式资料。



### 二. linux crypto框架方式

#### 2.1 概括

&emsp;&emsp;linux crypto框架方式是通过应用层打开操作一个套接字来间接调用下面框图中的通用内核api进行加解密作业，**目前spacc sdk还没有实现这种方式**。linux crypto框架比较复杂，这里只关注如何注册和使用加密算法。下面框图是注册和使用加密算法的主要数据结构和函数：

```c
/* 
 * 通用基础数据结构和函数
 */
struct crypto_alg			//算法实现
struct crypto_tfm			//算法对象,包含struct crypto_alg指针和上下文指针
struct crypto_async_request	 //请求,包含struct crypto_tfm指针
   
//注册和注销一个算法实现    
int crypto_register_alg(struct crypto_alg* alg)
void crypto_unregister_alg(struct crypto_alg* alg)

//申请和释放一个算法对象
struct crypto_tfm* crypto_alloc_base(const char* alg_name, u32 type, u32 mask)
void crypto_destroy_tfm(void* mem, struct crypto_tfm* tfm)

//等待请求和完成请求, 通常req->data = wait, 而wait包含一个struct completion
int crypto_wait_req(int err, struct crypto_wait* wait)
void crypto_req_done(struct crypto_async_request* req, int err)


/*
 * single-block symmetric key cipher(单块对称加密)相关数据结构和函数
 */
struct cipher_alg		//单块对称加密算法实现,被crypto_alg包含
struct crypto_cipher	//单块对称加密算法对象,包含struct crypto_tfm

//注册和注销一个单块对称加密算法实现
//注册时实现一个struct cipher_alg填充alg->cra_u.cipher, 并且设置alg->cra_type = NULL  
int crypto_register_alg(struct crypto_alg* alg)	
void crypto_unregister_alg(struct crypto_alg* alg)

//单块对称加密使用api
struct crypto_cipher* crypto_alloc_cipher(const char* alg_name, u32 type, u32 mask)
void crypto_free_cipher(struct crypto_cipher* tfm)
int crypto_has_cipher(const char* alg_name, u32 type, u32 mask)
unsigned int crypto_cipher_blocksize(struct crypto_cipher* tfm) 
int crypto_cipher_setkey(struct crypto_cipher* tfm, const u8* key, unsigned int keylen)
void crypto_cipher_encrypt_one(struct crypto_cipher* tfm, u8* dst, const u8* src)
void crypto_cipher_decrypt_one(struct crypto_cipher* tfm, u8* dst, const u8* src)    

/* 
 * muti-block symmetric key cipher(多块对称加密)相关数据结构和函数 
 */ 
struct skcipher_alg    	  //多块对称加密算法实现,包含struct crypto_alg
struct crypto_skcipher    //多块对称加密算法对象,包含struct crypto_tfm
struct skcipher_request   //多块对称加密请求,包含struct crypto_async_request
    
//注册和注销一个多块对称加密算法实现
int crypto_register_skcipher(struct skcipher_alg* alg)		//调用crypto_register_alg
void crypto_unregister_skcipher(struct skcipher_alg* alg)	//调用crypto_unregister_alg
    
//多块对称加密使用api     
struct crypto_skcipher* crypto_alloc_skcipher(const char* alg_name, u32 type, u32 mask)
void crypto_free_skcipher(struct crypto_skcipher* tfm)
int crypto_has_skcipher(const char* alg_name, u32 type, u32 mask)  
unsigned int crypto_skcipher_ivsize(struct crypto_skcipher* tfm)
unsigned int crypto_skcipher_blocksize(struct crypto_skcipher* tfm)
int crypto_skcipher_setkey(struct crypto_skcipher* tfm, const u8* key, unsigned int keylen)
struct crypto_skcipher* crypto_skcipher_reqtfm(struct skcipher_request * req) 
int crypto_skcipher_encrypt(struct skcipher_request* req)
int crypto_skcipher_decrypt(struct skcipher_request* req)  
unsigned int crypto_skcipher_reqsize(struct crypto_skcipher * tfm)    
void skcipher_request_set_tfm(struct skcipher_request * req, struct crypto_skcipher* tfm)
struct skcipher_request* skcipher_request_alloc(struct crypto_skcipher* tfm, gfp_t gfp)
void skcipher_request_free(struct skcipher_request* req)    
void skcipher_request_set_callback(struct skcipher_request* req, u32 flags, 
                                   crypto_completion_t compl, void * data)
void skcipher_request_set_crypt(struct skcipher_request* req, struct scatterlist* src, 
                                struct scatterlist* dst, unsigned int cryptlen, void * iv)   
    
    
/* 
 * hash(包括裸hash和hmac模式hash)相关数据结构和函数 
 */   
struct hash_alg_common	 //通用hash算法实现,包含struct crypto_alg
struct ahash_alg		//异步hash算法实现,包含struct hash_alg_common 
struct shash_alg		//同步hash算法实现,包含struct crypto_alg
struct crypto_ahash		//异步hash算法对象,包含struct crypto_tfm
struct crypto_shash      //同步hash算法对象,包含struct crypto_tfm
struct ahash_request	//异步hash请求,包含struct crypto_async_request
struct shash_desc		//同步hash请求

//注册和注销一个hash算法实现
int crypto_register_ahash(struct ahash_alg* alg)	//同步hash注册,调用crypto_register_alg
int crypto_register_shash(struct shash_alg* alg)	//异步hash注册,调用crypto_register_alg
void crypto_unregister_ahash(struct ahash_alg* alg)	//同步hash注销,调用crypto_register_alg    
void crypto_unregister_shash(struct shash_alg* alg) //异步hash注销,调用crypto_register_alg

//异步hash使用api      
struct crypto_ahash* crypto_alloc_ahash(const char* alg_name, u32 type, u32 mask)
void crypto_free_ahash(struct crypto_ahash* tfm)
unsigned int crypto_ahash_digestsize(struct crypto_ahash* tfm)
unsigned int crypto_ahash_statesize(struct crypto_ahash* tfm)    
struct crypto_ahash* crypto_ahash_reqtfm(struct ahash_request* req)
unsigned int crypto_ahash_reqsize(struct crypto_ahash* tfm)
int crypto_ahash_setkey(struct crypto_ahash* tfm, const u8* key, unsigned int keylen)
int crypto_ahash_finup(struct ahash_request* req)    
int crypto_ahash_final(struct ahash_request* req)
int crypto_ahash_digest(struct ahash_request* req)
int crypto_ahash_export(struct ahash_request* req, void* out)
int crypto_ahash_import(struct ahash_request* req, const void* in)
int crypto_ahash_init(struct ahash_request* req)
void ahash_request_set_tfm(struct ahash_request* req, struct crypto_ahash* tfm)
struct ahash_request* ahash_request_alloc(struct crypto_ahash* tfm, gfp_t gfp)    
void ahash_request_free(struct ahash_request* req)
void ahash_request_set_callback(struct ahash_request* req, u32 flags, 
                                crypto_completion_t compl, void* data) 
void ahash_request_set_crypt(struct ahash_request* req, struct scatterlist* src, 
                             u8* result, unsigned int nbytes)  
    
//同步hash使用api  
struct crypto_shash* crypto_alloc_shash(const char* alg_name, u32 type, u32 mask)
void crypto_free_shash(struct crypto_shash* tfm)
unsigned int crypto_shash_blocksize(struct crypto_shash* tfm)    
unsigned int crypto_shash_digestsize(struct crypto_shash* tfm)
unsigned int crypto_shash_descsize(struct crypto_shash* tfm)
int crypto_shash_setkey(struct crypto_shash* tfm, const u8* key, unsigned int keylen)
int crypto_shash_digest(struct shash_desc* desc, const u8* data, unsigned int len, u8* out)
int crypto_shash_export(struct shash_desc* desc, void* out)
int crypto_shash_import(struct shash_desc* desc, const void* in)
int crypto_shash_init(struct shash_desc* desc)
int crypto_shash_update(struct shash_desc* desc, const u8* data, unsigned int len)
int crypto_shash_final(struct shash_desc* desc, u8* out)
int crypto_shash_finup(struct shash_desc* desc, const u8* data, unsigned int len, u8* out)
    
/*
 * aead(认证加密)相关数据结构和函数 
 */ 
struct aead_alg		//认证加密算法实现,包含struct crypto_alg
struct crypto_aead	//认证加密算法对象,包含struct crypto_tfm
struct aead_request	//认证加密请求,包含struct crypto_async_request
    
//注册和注销一个认证加密算法实现   
int crypto_register_aead(struct aead_alg* alg)		//调用crypto_register_alg
void crypto_unregister_aead(struct aead_alg* alg)    //调用crypto_unregister_alg
    
//认证加密使用api 
struct crypto_aead* crypto_alloc_aead(const char* alg_name, u32 type, u32 mask)   
void crypto_free_aead(struct crypto_aead* tfm) 
unsigned int crypto_aead_ivsize(struct crypto_aead* tfm) 
unsigned int crypto_aead_authsize(struct crypto_aead* tfm) 
unsigned int crypto_aead_blocksize(struct crypto_aead* tfm) 
int crypto_aead_setkey(struct crypto_aead* tfm, const u8* key, unsigned int keylen)  
int crypto_aead_setauthsize(struct crypto_aead* tfm, unsigned int authsize)
int crypto_aead_encrypt(struct aead_request* req) 
int crypto_aead_decrypt(struct aead_request* req) 
unsigned int crypto_aead_reqsize(struct crypto_aead* tfm)
void aead_request_set_tfm(struct aead_request* req, struct crypto_aead* tfm)
struct aead_request* aead_request_alloc(struct crypto_aead* tfm, gfp_t gfp)
void aead_request_free(struct aead_request* req)  
void aead_request_set_callback(struct aead_request* req, u32 flags, 
                               crypto_completion_t compl, void* data) 
void aead_request_set_crypt(struct aead_request* req, struct scatterlist* src, 
                            struct scatterlist* dst, unsigned int cryptlen, u8* iv)  
void aead_request_set_ad(struct aead_request* req, unsigned int assoclen)    
```

&emsp;&emsp;首先关注三个通用的基础数据结构struct crypto_alg，struct crypto_tfm和struct crypto_async_request。struct crypto_alg表示一个**算法实现**，一个算法实现可以有多个使用者，每个使用者都有自己的**算法对象**struct crypto_tfm，而每个使用者都可以使用多个**请求**struct crypto_async_request进行作业处理。因此一个struct crypto_alg可以对应多个struct crypto_tfm，而一个struct crypto_tfm还对应可以多个struct crypto_async_request。  
&emsp;&emsp;然后是通用的算法实现注册函数crypto_register_alg和算法对象申请函数crypto_alloc_base。算法实现者填充struct crypto_alg然后通过crypto_register_alg将算法实现注册到内核crypto框架中。算法使用者通过crypto_alloc_base获取算法对象，然后基于该算法对象通过对应的算法使用api进行作业。  
&emsp;&emsp;这几个通用的数据结构和函数是特定算法类型的基类，通常不直接使用。比如多块对称加密使用的算法实现是包含struct crypto_alg的struct skcipher_alg，算法对象是包含struct crypto_tfm的struct crypto_skcipher，请求是包含struct crypto_async_request的struct skcipher_request，算法实现注册函数是包含crypto_register_alg的crypto_register_skcipher，算法对象申请函数是包含crypto_alloc_base的crypto_alloc_skcipher。  
&emsp;&emsp;在上面框图中，除了通用的数据结构和函数还列举了几种典型的算法类型的数据结构和函数，它们都是对通用数据结构和函数的包装。这几种类型分别为**单块对称加密**(single-block symmetric key cipher)，**多块对称加密**(muti-block symmetric key cipher)，**哈希**(hash)，**认证加密**(aead)。在这里我们只讨论这几种模式，因为这几种是spacc支持的且常用的。其中单块对称加密是同步的，多块对称加密和认证加密是异步的，而哈希具有同步和异步两种模式。接下来以多块对称加密来讲解注册流程和作业调用流程，其他模式差不多，有些许区别，自行查看相关代码和文档。

#### 2.2 注册流程

&emsp;&emsp;多块对称加密算法注册由算法实现者填充好struct skcipher_alg结构体后通过crypto_register_skcipher发起，如下简化代码所示：

```c
int crypto_register_skcipher(struct skcipher_alg *alg)
{
    struct crypto_alg *base = &alg->base;
    skcipher_prepare_alg(alg);
    return crypto_register_alg(base);
}

static int skcipher_prepare_alg(struct skcipher_alg *alg)
{
	struct crypto_alg *base = &alg->base;
	base->cra_type = &crypto_skcipher_type;
	base->cra_flags &= ~CRYPTO_ALG_TYPE_MASK;
	base->cra_flags |= CRYPTO_ALG_TYPE_SKCIPHER;
}

int crypto_register_alg(struct crypto_alg *alg)
{
    __crypto_register_alg(alg);
}

static struct crypto_larval *__crypto_register_alg(struct crypto_alg *alg)
{
    list_add(&alg->cra_list, &crypto_alg_list);
}
```

&emsp;&emsp;可以看到crypto_register_skcipher所做的事并不复杂，主要是设置struct skcipher_alg包含的通用算法实现结构体struct crypto_alg的类型然后注册到一个链接了所有注册的算法实现的链表crypto_alg_list上。重点是调用crypto_register_skcipher前如何填充struct skcipher_alg。可以参考`mtk-aes.c`中的cbc(aes)加密算法实现：

```c
static struct skcipher_alg aes_algs[] = {
    {
        .base.cra_name		= "cbc(aes)",
        .base.cra_driver_name	= "cbc-aes-mtk",
        .base.cra_priority	= 400,
        .base.cra_flags		= CRYPTO_ALG_ASYNC,
        .base.cra_blocksize	= AES_BLOCK_SIZE,
        .base.cra_ctxsize	= sizeof(struct mtk_aes_ctx),
        .base.cra_alignmask	= 0xf,
        .base.cra_module	= THIS_MODULE,

        .min_keysize		= AES_MIN_KEY_SIZE,
        .max_keysize		= AES_MAX_KEY_SIZE,
        .setkey			= mtk_aes_setkey,
        .encrypt		= mtk_aes_cbc_encrypt,
        .decrypt		= mtk_aes_cbc_decrypt,
        .ivsize			= AES_BLOCK_SIZE,
        .init			= mtk_aes_init_tfm,
    },
    ...
};    
```

&emsp;&emsp;这里就不再展开讲里面的函数的具体实现，因为这是硬件相关的，每个硬件都有所不同。下面按照该示例代码的顺序解释出现的每个数据成员和函数的通用含义，这是每个对称加密硬件实现都要遵守的。

```c
struct skcipher_alg {
    struct crypto_alg {
        //通用算法名字，可以和其他算法实现一样
        char cra_name[CRYPTO_MAX_ALG_NAME];
        
        //独特算法名字, 不可以和其他算法实现的独特或者通用算法名字一样，否则注册不成功
        char cra_driver_name[CRYPTO_MAX_ALG_NAME];
        
        //算法实现的优先值，当通过通用算法名字申请一个算法对象时，如果存在多个通用算法名字
        //一样的算法实现，则选择优先值最高的算法实现生成算法对象
        int cra_priority;
        
        //标志位，可以表示算法实现的模式类型，比如CRYPTO_ALG_ASYNC表示异步算法，在上文可以看到
        //crypto_register_skcipher注册过程中还会或上CRYPTO_ALG_TYPE_SKCIPHER表示对称加密
        u32 cra_flags;
        
        //最小加密长度单元，对于块加密算法是一块的大小，对于流加密是一字节
        unsigned int cra_blocksize;
        
        //软件上下文的大小，这是通过crypto_alloc_base分配一个算法对象时额外申请的内存大小
        unsigned int cra_ctxsize;
        
        //输入和输出数据的buf的对齐掩码，如果没按要求对齐内核会在作业前自动搬运到对齐的地方
        unsigned int cra_alignmask;
        
        //注册该算法实现的内核模块，设置为THIS_MODULE
        struct module *cra_module;
    }；		

	unsigned int min_keysize;	//该算法支持的最小密钥长度
	unsigned int max_keysize;	//该算法支持的最大密钥长度	
	unsigned int ivsize;    	//初始化向量的长度
	
    //将密钥设置到硬件或者软件上下文中
    int (*setkey)(struct crypto_skcipher *tfm, const u8 *key, unsigned int keylen);	
        
    //将请求skcipher_request中的src的数据加密到dst中，其中iv和加密的数据大小也在请求中指定
    //关于请求的信息参考下面struct skcipher_request结构
    int (*encrypt)(struct skcipher_request *req);
        
    //将请求skcipher_request中的src的数据解密到dst中，其中iv和解密的数据大小也在请求中指定    
    int (*decrypt)(struct skcipher_request *req); 
        
    //初始化对称加密算法对象. 在通过crypto_alloc_skcipher申请一个对称加密算法对象
    //crypto_skcipher时,首先通过crypto_alloc_tfm申请一个通用的算法对象crypto_tfm,
    //然后就通过contain_of得到包含crypto_tfm的crypto_skcipher的地址, 然后通过这里
    //的init初始化crypto_skcipher
    int (*init)(struct crypto_skcipher *tfm);					
}; 

struct skcipher_request {
	unsigned int cryptlen;		//该请求需要处理的数据大小
	u8 *iv;					   //初始化向量	
	struct scatterlist *src;    //数据源地址
	struct scatterlist *dst;    //数据目的地址
	struct crypto_async_request base;
	void *__ctx[] CRYPTO_MINALIGN_ATTR;
};
```

&emsp;&emsp;通过对skcipher_alg中的这些成员的通用含义的理解再结合对自己的加密硬件的掌握和参考别的加密硬件的实现最终可以将自己的加密硬件注册到内核crypto框架给用户使用。

#### 2.3 作业调用流程

&emsp;&emsp;使用内核crypto框架的好处就是无论硬件有什么差别用户的调用流程都一样。应用层通过操作特定类型的套接字来间接在内核调用2.1小节里的算法使用api来启动作业。对称加密使用的套接字如下：

```c
struct sockaddr_alg sa = {
    .salg_family = AF_ALG,
    .salg_type = "skcipher", 
    .salg_name = "cbc(aes)" 
};
```

&emsp;&emsp;关于应用层如何通过套接字设置参数启动作业可以自行参考网上资料，这里讲解内核层算法使用api的调用流程，因为无论应用层软件是什么形式（比如自己操作设置sockaddr_alg套接字或者使用openssl开源软件）最终都要落实到正确的内核算法使用api的调用流程上。我们通过以下一个示例来展示对称加解密的调用流程模板。

```c
//封装结构
struct skcipher_def {
    struct scatterlist sg;
    struct crypto_skcipher *tfm;
    struct skcipher_request *req;
    struct crypto_wait wait;
};

//启动加密然后等待请求完成 
static unsigned int test_skcipher_encdec(struct skcipher_def *sk, int enc)
{
    if (enc)	//加密
        //首先调用crypto_skcipher_encrypt启动异步加密请求处理，最终会调用
        //算法实现skcipher_alg的encrypt函数，它通常是将请求放入一个队列就返回，
        //然后调用wait_for_completion睡眠直到请求完成调用回调函数crypto_req_done唤醒
        crypto_wait_req(crypto_skcipher_encrypt(sk->req), &sk->wait);
    else		//解密
        crypto_wait_req(crypto_skcipher_decrypt(sk->req), &sk->wait);
}

//初始化和启动加密操作
static int test_skcipher(void)
{
    struct skcipher_def sk;
    struct crypto_skcipher *skcipher = NULL;
    struct skcipher_request *req = NULL;
    char *scratchpad = NULL;
    char *ivdata = NULL;
    unsigned char key[32];

    //申请一个对称加密算法对象
    skcipher = crypto_alloc_skcipher("cbc-aes-aesni", 0, 0);

    //申请一个对称加密请求
    req = skcipher_request_alloc(skcipher, GFP_KERNEL);

	//设置请求回调函数crypto_req_done和相关数据
    skcipher_request_set_callback(req, CRYPTO_TFM_REQ_MAY_BACKLOG,
                      crypto_req_done,
                      &sk.wait);

    get_random_bytes(&key, 32);		//随机256位密钥
    
    //调用skcipher的算法实现的setkey函数将密钥设置到硬件或软件数据结构中
    crypto_skcipher_setkey(skcipher, key, 32);

    ivdata = kmalloc(16, GFP_KERNEL);
    get_random_bytes(ivdata, 16);	//随机的16字节(单块加密的数据大小)初始化向量

    scratchpad = kmalloc(16, GFP_KERNEL);
    get_random_bytes(scratchpad, 16);	//随机的16字节待加密的数据

    sk.tfm = skcipher;
    sk.req = req;

    sg_init_one(&sk.sg, scratchpad, 16);	//将加密数据buf设置到scatterlist
    
    //将数据源地址，目的地址，iv，和需要加密的长度设置到请求中
    skcipher_request_set_crypt(req, &sk.sg, &sk.sg, 16, ivdata);
    
    crypto_init_wait(&sk.wait);		//初始化completion

    //启动加密然后等待请求完成 
    test_skcipher_encdec(&sk, 1);
}
```

&emsp;&emsp;调用流程上面的代码注释已经写的很清晰，可以在内核模块测试代码中直接调用test_skcipher发起初始化和启动加密作业。在test_skcipher中首先通过crypto_alloc_skcipher申请算法对象；然后通过skcipher_request_alloc对该算法对象申请一个请求；然后通过skcipher_request_set_callback设置请求回调函数；然后通过crypto_skcipher_setkey将密钥设置到硬件中；然后通过skcipher_request_set_crypt将数据源地址和目的地址，iv和加密数据长度设置到请求中；然后通过test_skcipher_encdec中调用的crypto_skcipher_encrypt启动异步作业请求处理，最终会调用到skcipher_alg的硬件实现的encrypt，通常是将请求放入某个队列然后返回；然后通过crypto_wait_req中调用的wait_for_completion睡眠等待请求完成，最后请求完成后自动调用回调函数crypto_req_done接着调用complete唤醒该加密任务进程。



### 三.  其他

#### 大数据量加密方案

&emsp;&emsp;由于加密数据可能很大，而加密硬件spacc一次性能处理的数据不超过1m，因此需要探讨大数据量如何分块加密的方式，其中关系到应用层和驱动层的分工和协作，以下给出了四种粗略方案，其中第一种到第三种使用传统设备节点方式，第四种使用linux crypto方式。

##### 第一种 

```c
+----------+----------+----------+----------+----------+
|   job1   |    job2  |    job3  |    job4  |   job5   |          用户态
+----------+----------+----------+----------+----------+
    |   ^      |  ^    	  |	 ^		  |  ^ 	    |   ^ 	 			
    V   |      V  |       V  |        V  |      V   |
+----------+----------+----------+----------+----------+
|   wait   |   wait   |   wait   |    wait  |   wait   |          内核态
+----------+----------+----------+----------+----------+

```

&emsp;&emsp;这种方案是spacc sdk目前提供的方式。因为spacc加密硬件一次能处理的数据不超过1M，因此每一个作业处理的数据大小不能超过1M。应用层每发起一个作业请求不会马上返回，而是阻塞在内核态直到收到作业完成中断被唤醒才返回用户态，然后接着发起下一个作业请求。使用这种方式的好处是驱动几乎不用改动，能很快使用起来。缺点是后面的数据请求需要等到前面的数据处理完才能发起。

##### 第二种

```c 
+----------+----------+----------+----------+----------+
|   job1   |    job2  |    job3  |    job4  |   job5   |          用户态
+----------+----------+----------+----------+----------+
    |   ^      |  ^    	  |	 ^		 |  ^ 	    |  ^					
    V   |      V  |       V  |       V  |       V  |
+----------+----------+----------+----------+----------+
|  nowait  |  nowait  |  nowait  |  nowait  |   wait   |          内核态
+----------+----------+----------+----------+----------+
```

&emsp;&emsp;第二种方案如图所示，不是每个作业都要阻塞等待上一个作业完成才能发起。驱动可以跟踪spacc中的fifo，当用户态通过作业提交接口ioctl进入内核态时，如果发现fifo中的作业个数没有到达驱动设定的值，可以直接将作业放入fifo然后直接返回。如果fifo中的作业个数已经达到驱动设置的值，则阻塞睡眠在内核，直到某个之前提交的作业完成触发中断，在中断中将阻塞在内核的该任务唤醒。此时继续查看fifo数量，此时fifo作业数量会少于设定的最大值，因此可以将作业放入fifo然后返回用户态。其实第一种方案是这一种方案的一个特例，只是fifo作业数量最大值是1。使用这种方案比第一种方案有更好的性能，需要对驱动作些中断方面的修改。

##### 第三种

```c
+------------------------------------------------------+
|                         data                         |          用户态
+------------------------------------------------------+
                      |          ^   
                      V          |
+----------+----------+----------+----------+----------+
|   job1   |    job2  |    job3  |    job4  |   job5   |          内核态
+----------+----------+----------+----------+----------+
```

&emsp;&emsp;第三种方案是应用层将数据一次性全部发给内核，然后内核再分块进行处理，处理完后再返回给应用层。这种方案的好处是应用层几乎不用处理，只需要提供源数据地址和目的数据地址。但是驱动改动量比较大，并且将大量数据一次传送给内核会对内核造成很大的负担。可以通过预留加解密的内存给应用层单独使用，内核的内存管理子系统不会使用它，此时可以传入物理地址避免从用户态拷贝到内核态，加解密数据越大需要预留的内存也越大，系统其他地方能使用的内存资源就越小。

##### 第四种

&emsp;&emsp;下面是网上的一个例子，用这种方式应用软件移植性高，只要硬件在内核crypto框架中正确的注册都可以按照下面的方式使用，其中对于大数据量处理也需要自行分块处理，如代码中for循环使用sendmsg来发送数据给硬件加密，每次sendmsg发送的大小不能超过硬件的最大加密长度。在分块使用sendmsg发送完之后可以使用一个recv一次性接收完处理后得到的数据。可能存在一些开源软件封装了这些接口让用户更容易使用，也许不需要用户自己分块，但是对内核实现来说是一样的。目前spacc sdk还未实现对内核crypto框架的支持，需要熟悉crypto框架和spacc然后实现算法注册进内核才能使用。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <linux/if_alg.h>
#include <unistd.h>
#include <fcntl.h>

#define BLOCK_SIZE 16 

int main() {
    // 初始化加密数据和密钥
    char *plaintext = "Hello, world! This is a test of the long plaintext. It is longer than one block and needs to be split into multiple blocks for encryption.";
    char *key = "0123456789abcdef"; 
    size_t plaintext_len = strlen(plaintext);
    char iv[BLOCK_SIZE] = {0}; // 初始向量为空

    // 创建socket并绑定算法类型为"skcipher"
    int fd = socket(AF_ALG, SOCK_SEQPACKET, 0);
    struct sockaddr_alg sa = {
        .salg_family = AF_ALG,
        .salg_type = "skcipher",
        .salg_name = "cbc(aes)"
    };
    bind(fd, (struct sockaddr *)&sa, sizeof(sa)) 

    // 设置加密参数
    struct msghdr msg = {0};
    struct cmsghdr *cmsg;
    char cbuf[CMSG_SPACE(sizeof(struct af_alg_iv))] = {0};

    msg.msg_control = cbuf;
    msg.msg_controllen = sizeof(cbuf);

    struct iovec iov[2];
    iov[0].iov_base = iv;
    iov[0].iov_len = BLOCK_SIZE;
    iov[1].iov_base = malloc(BLOCK_SIZE * sizeof(char));
    iov[1].iov_len = BLOCK_SIZE;

    msg.msg_iov = iov;
    msg.msg_iovlen = 2;

    cmsg = CMSG_FIRSTHDR(&msg);
    cmsg->cmsg_level = SOL_ALG;
    cmsg->cmsg_type = ALG_SET_IV;
    cmsg->cmsg_len = CMSG_LEN(sizeof(struct af_alg_iv));
    struct af_alg_iv *alg_iv = (struct af_alg_iv *)CMSG_DATA(cmsg);
    alg_iv->ivlen = BLOCK_SIZE;

    // 设置加密会话
    int tfmfd = accept(fd, NULL, 0);
    if (tfmfd == -1) {
        perror("accept");
        exit(EXIT_FAILURE);
    }
    if (setsockopt(tfmfd, SOL_ALG, ALG_SET_KEY, key, 16) == -1) {
        perror("setsockopt set key");
        exit(EXIT_FAILURE);
    }

    // 分次加密数据
    size_t i = 0;
    for (i = 0; i < plaintext_len / BLOCK_SIZE; i++) { 
        memcpy(iov[1].iov_base, plaintext + i * BLOCK_SIZE, BLOCK_SIZE);
        sendmsg(tfmfd, &msg, 0);
    }
    if (plaintext_len % BLOCK_SIZE != 0) {
        iov[1].iov_len = BLOCK_SIZE;
        memset(iov[1].iov_base, 0, BLOCK_SIZE);
        memcpy(iov[1].iov_base, plaintext + i * BLOCK_SIZE, plaintext_len % BLOCK_SIZE);
        sendmsg(tfmfd, &msg, 0);
    }

    // 一次接收加密后的数据
    size_t ciphertext_len = (plaintext_len / BLOCK_SIZE + 1) * BLOCK_SIZE; 
    char *ciphertext = calloc(1, ciphertext_len * sizeof(char));
    ssize_t res = recv(tfmfd, ciphertext, ciphertext_len, MSG_WAITALL);

    printf("Plaintext: %s\n", plaintext);
    printf("Ciphertext: ");
    for (int i=0; i<res; i++) {
        printf("%02x", (unsigned char)ciphertext[i]);
    }
    printf("\n");

    close(tfmfd);
    close(fd);

    free(iov[1].iov_base);
    free(ciphertext);
    return 0;
}
```

##### 对比

&emsp;&emsp;简单对比一下这四种方案。从应用层来说，第一种和第二种使用方便性差不多，用户需要操作设备节点对数据进行分块处理数据，第三种使用最简单，不需要分块数据，第四种也需要分块数据，但是通用性和移植性强，可以对接其他开源软件和库，而前面三种无法直接移植到其他硬件ip上去。从内核层实现来说，第一种到第四种实现改动越来越大，第一种不需要改动可以直接使用，第二种主要修改驱动中断相关内容，第三种除了中断相关还有重新设计实现与用户的接口，第四种需要实现加密算法注册进内核crypto子系统。


#### 开源框架


##### openssl


##### uadk
