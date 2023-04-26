## <center>spacc驱动小结</center>

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
                     dev_name(&pdev->dev), &pdev->dev)

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

&emsp;&emsp;首先关注三个通用的基础数据结构`struct crypto_alg`，`struct crypto_tfm`和`struct crypto_async_request`。`struct crypto_alg`表示一个**算法实现**，一个算法实现可以有多个使用者，每个使用者都有自己的**算法对象**`struct crypto_tfm`，而每个使用者都可以使用多个**请求**`struct crypto_async_request`进行作业处理。因此一个`struct crypto_alg`对应多个`struct crypto_tfm`，而一个`struct crypto_tfm`对应多个`struct crypto_async_request`。  
&emsp;&emsp;然后是通用的算法实现注册函数`crypto_register_alg`和算法对象申请函数`crypto_alloc_base`。算法实现者填充`struct crypto_alg`然后通过`crypto_register_alg`将算法实现注册到内核crypto框架中。算法使用者通过`crypto_alloc_base`获取算法对象，然后基于该算法对象通过对应的算法使用api进行作业。  
&emsp;&emsp;这几个通用的数据结构和函数是特定算法类型的基类，通常不直接使用。比如多块对称加密使用的算法实现是包含`struct crypto_alg`的`struct skcipher_alg`，算法对象是包含`struct crypto_tfm`的`struct crypto_skcipher`，请求是包含`struct crypto_async_request`的`struct skcipher_request`，算法实现注册函数是包含`crypto_register_alg`的`crypto_register_skcipher`，算法对象申请函数是包含`crypto_alloc_base`的`crypto_alloc_skcipher`。  
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
    int (*encrypt)(struct skcipher_request *req);
        
    //将请求skcipher_request中的src的数据解密到dst中，其中iv和解密的数据大小也在请求中指定    
    int (*decrypt)(struct skcipher_request *req); 
        
    //初始化对称加密算法对象. 在通过crypto_alloc_skcipher申请一个对称加密算法对象
    //crypto_skcipher时,首先通过crypto_alloc_tfm申请一个通用的算法对象crypto_tfm,
    //然后就通过contain_of得到包含crypto_tfm的crypto_skcipher的地址, 然后通过这里
    //的init初始化crypto_skcipher
    int (*init)(struct crypto_skcipher *tfm);					
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

&emsp;&emsp;关于应用层如何通过套接字设置参数启动作业可以自行参考网上资料，这里讲解内核层算法使用api的调用流程，因为无论应用层软件是什么形式（比如自己操作设置sockaddr_alg套接字或者使用openssl开源软件）最终都要落实到正确的内核算法使用api的调用流程上。