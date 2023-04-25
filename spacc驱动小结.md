## <center>spacc驱动小结</center>

[TOC]



### 一. 传统设备节点方式

#### 1.1 概括

&emsp;&emsp;设备节点方式是通过应用程序打开设备节点，然后通过ioctl设置密钥和其他相关参数，最后启动作业。这是spacc驱动包提供的方式，spacc驱动包主要包含以下几个内核模块：

- **elppdu.ko** ：由pdu.c和一些辅助代码编译得到，它给其他驱动模块提供了内存、DMA的分配和操作，资源锁定释放等通用的api。这些api通常是linux内核其他函数的包装，如pdu_io_write32是对writel的包装，PDU_LOCK宏扩展就是spin_lock_irqsave。
- **elpmem.ko** ：由spacc_mem.c编译得到，是一个平台设备驱动模块，它主要做的首先是创建平台设备，根据在Makefile中配置的PDU_BASE_ADDR和PDU_BASE_IRQ生成资源，读取sapcc硬件寄存器填入pdu_info结构中，将资源和pdu_info添加平台设备并将平台设备注册到平台设备总线。
- **elspacc.ko** ：由spacc.c和其他辅助代码编译得到，封装和提供了对加密硬件的底层操作，比如作业的管理，中断的管理，密钥上下文的设置等。elspacc.ko 注册的平台驱动会和elpmem.ko匹配，然后提取出平台设备的资源和pdu_info，根据这些数据和对硬件相应的读取设置操作初始化一个spacc_device结构体，该结构体包含除了根据作业需要临时传入设置参数之外的所有的信息。
- **elspaccusr.ko** ：由spacc_dev.c编译得到，它注册了一个字符设备*/dev/spaccusr*，应用层可以打开操作这个设备节点进行加解密作业，主要是通过调用ioctl并通过传入适当的参数来设置和启动加解密操作。该字符设备的ioctl驱动接口使用elspacc.ko中的函数进行设置和作业。

&emsp;&emsp;**实现者**实现和提供算法，**使用者**使用算法进行作业处理，实现者主要关注**注册流程**，使用者主要关注**作业调用流程**，如下文所述。

#### 1.2 注册流程



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

#### 2.1 概括

&emsp;&emsp;crypto api方式是通过应用层打开操作一个套接字来间接调用下面框图中的通用内核api进行加解密作业。linux crypto框架比较复杂，这里只关注如何注册和使用加密算法。下面框图是注册和使用加密算法的主要数据结构和函数：

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

//使用单块对称加密的主要算法流程api
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
    
//使用多块对称加密的主要算法流程api     
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
 * hash(包括裸hash和hmac模式hash)
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

//使用异步hash的主要算法流程api      
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
    
//使用同步hash的主要算法流程api  
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
 * aead(认证加密)
 */ 
struct aead_alg		//认证加密算法实现,包含struct crypto_alg
struct crypto_aead	//认证加密算法对象,包含struct crypto_tfm
struct aead_request	//认证加密请求,包含struct crypto_async_request
    
//注册和注销一个认证加密算法实现   
int crypto_register_aead(struct aead_alg* alg)		//调用crypto_register_alg
void crypto_unregister_aead(struct aead_alg* alg)    //调用crypto_unregister_alg
    
//使用认证加密的主要算法流程api 
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
&emsp;&emsp;然后是通用的算法实现注册函数`crypto_register_alg`和算法对象申请函数`crypto_alloc_base`。算法实现者填充`struct crypto_alg`然后通过`crypto_register_alg`将算法实现注册到内核crypto框架中。算法使用者通过`crypto_alloc_base`获取算法对象，然后基于该算法对象通过对应的算法流程api进行作业。  
&emsp;&emsp;这几个通用的数据结构和函数是特定算法类型的基类，通常不直接使用。比如多块对称加密使用的算法实现是包含`struct crypto_alg`的`struct skcipher_alg`，算法对象是包含`struct crypto_tfm`的`struct crypto_skcipher`，请求是包含`struct crypto_async_request`的`struct skcipher_request`，算法实现注册函数是包含`crypto_register_alg`的`crypto_register_skcipher`，算法对象申请函数是包含`crypto_alloc_base`的`crypto_alloc_skcipher`。





#### 2.2 注册流程



#### 2.3 作业调用流程