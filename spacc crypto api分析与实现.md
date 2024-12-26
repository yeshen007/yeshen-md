## <center>spacc crypto api分析与实现</center>

###  1. linux crypto api概括

&emsp;&emsp;linux crypto api提供一套统一的api给内核注册和使用加解密、hash和解压缩算法。注册算法是将硬件操作函数(加解密，设置密钥等)设置到crypo框架对应结构体，将暂存上下文信息(除了处理的数据之外的信息如key，iv，加解密模式等)的数据结构设置到crypo框架对应结构体，最后通过内核的函数注册到内核crypto系统。使用crypo框架提供的api处理数据主要有两步：设置上下文信息，处理数据。设置上下文信息一般会先保存在软件的数据结构中，最后在处理数据阶段再从数据结构取出设置到硬件；处理数据主要是设置提交请求，请求包含数据源和数据目的(根据算法类型的不同可能还包含部分上下文信息)，提交的请求通常不会马上使用硬件处理，会先保存在软件维护的队列中，当通过某种方式(通常是中断)知道硬件可用时再从队列取出一个请求给硬件处理。  
&emsp;&emsp;目前只实现aes模式的对称加解密，下文的分析总结以aes对称加解密作为示范。

#### 1.1 主要数据结构

![](draw\cryptoapiarch.drawio.svg)

##### (1) crypto_alg

&emsp;&emsp;表示一个注册到内核crypto框架的**算法**。主要包含算法在整个注册被使用期间都不会改变的属性和回调函数，如名字，blocksize，算法实例初始化函数cra_init等。

##### (2) crypto_tfm

&emsp;&emsp;表示**算法实例**，当要使用某种算法执行加解密任务时，需要创建一个crypto_tfm结构体。除了包含表示算法的固定属性crypto_alg外，crypto_tfm还包括本次任务特定的上下文信息，比如spacc_aes_ctx中的秘钥。

##### (3) crypto_async_request

&emsp;&emsp;表示加解密任务中的一个**请求**(单次子任务)，主要包含crypto_tfm和请求完成回调函数。

##### (4) skcipher_alg

&emsp;&emsp;表示**对称加密算法**，除了包含crypto_alg表示的基础属性和回调函数之外还包含一些对称加密特有的属性和回调函数。

##### (5) crypto_skcipher

&emsp;&emsp;表示**对称加密算法实例**，除了包含crypto_tfm之外还包括该算法类型实例特有的属性。

##### (6) skcipher_request

&emsp;&emsp;表示一个**对称加解密的请求**，除了包含crypto_async_request外还包括对称加密类型相关的属性比如加密长度和源数据，目的数据，初始化向量，以及驱动自定义的结构。

##### (7) spacc_sk_ctx

&emsp;&emsp;表示在一整个加解密任务过程中需要保存的特定驱动数据，在我的实现中保存密钥和其他需要的数据和函数。

##### (8) spacc_sk_reqctx

&emsp;&emsp;表示在一个请求处理过程中需要保存的特定驱动数据，在我的实现中保存加解密的模式。

#### 1.2 算法注册使用流程

##### (1) 注册算法

&emsp;&emsp;通过crypto_register_skcipher(struct skcipher_alg *alg)函数将已经设置好的算法alg注册到内核。

##### (2) 分配设置算法实例

&emsp;&emsp;skcipher = crypto_alloc_skcipher(alg_name, ...)得到一个对称算法alg_name的实例对象，接下来的api调用都需要它。

##### (3) 分配设置请求

&emsp;&emsp;req = skcipher_request_alloc(skcipher, ...)得到一个请求，然后再根据数据源地址、数据目的地址和初始化向量设置请求。

##### (4) 设置秘钥

&emsp;&emsp;crypto_skcipher_setkey(skcipher, key, keylen)设置密钥。

##### (5) 请求处理

&emsp;&emsp;crypto_skcipher_encrypt(req)提交请求，请求通常先放入队列，是否会直接处理看硬件状态和队列状态。

##### (6) 请求完成

&emsp;&emsp;请求完成一般触发中断，通常在中断上半部清中断，在下半部继续处理剩下的请求。



### 2. atmel skcipher分析记录

> atmel-aes.c

#### 2.1 注册算法

```c
static int atmel_aes_probe(struct platform_device *pdev)
{
	struct atmel_aes_dev *aes_dd;
	struct device *dev = &pdev->dev;
	aes_dd = devm_kzalloc(&pdev->dev, sizeof(*aes_dd), GFP_KERNEL);

	aes_dd->dev = dev;
	platform_set_drvdata(pdev, aes_dd);

	tasklet_init(&aes_dd->done_task, atmel_aes_done_task,
					(unsigned long)aes_dd);
	tasklet_init(&aes_dd->queue_task, atmel_aes_queue_task,
					(unsigned long)aes_dd);
	crypto_init_queue(&aes_dd->queue, ATMEL_AES_QUEUE_LENGTH);
	devm_request_irq(&pdev->dev, aes_dd->irq, atmel_aes_irq,
			       IRQF_SHARED, "atmel-aes", aes_dd);

	atmel_aes_register_algs(aes_dd);	//注册算法
}

static struct skcipher_alg aes_algs[] = {
    ...
    {
        .base.cra_name		= "cbc(aes)",
        .base.cra_driver_name	= "atmel-cbc-aes",
        .base.cra_blocksize	= AES_BLOCK_SIZE,
        .base.cra_ctxsize	= sizeof(struct atmel_aes_ctx),		

        .init			= atmel_aes_init_tfm,
        .min_keysize		= AES_MIN_KEY_SIZE,
        .max_keysize		= AES_MAX_KEY_SIZE,
        .setkey			= atmel_aes_setkey,
        .encrypt		= atmel_aes_cbc_encrypt,
        .decrypt		= atmel_aes_cbc_decrypt,
        .ivsize			= AES_BLOCK_SIZE,
    },
    ...
};
```

#### 2.2 分配设置算法实例

```c
crypto_alloc_skcipher
	crypto_alloc_tfm
		crypto_alloc_tfm_node
			crypto_find_alg
			crypto_create_tfm_node
				alloc(sizeof(struct crypto_tfm) 
                      	+ frontend->tfmsize 		//offsetof(struct crypto_skcipher, base)
                      	+ frontend->extsize(alg)	//sizeof(struct atmel_aes_ctx)
				tfm->__crt_alg = alg;		//算法对象绑定算法
				frontend->init_tfm()		//crypto_skcipher_init_tfm
					alg->init		//atmel_aes_init_tfm
						crypto_skcipher_set_reqsize(sizeof(struct atmel_aes_reqctx))
						ctx->base.start = atmel_aes_start	//
```

#### 2.3 分配设置请求

```c
skcipher_request_alloc
	kmalloc(sizeof(struct skcipher_request) + crypto_skcipher_reqsize(tfm))
	skcipher_request_set_tfm(req, tfm)	//请求绑定算法对象
...
skcipher_request_set_crypt(req, src, dst, cryptlen, iv)
```

#### 2.4 设置秘钥

```c
crypto_skcipher_setkey
	cipher->setkey  //atmel_aes_setkey

static int atmel_aes_setkey(struct crypto_skcipher *tfm, const u8 *key,
			   unsigned int keylen)
{
	struct atmel_aes_base_ctx *ctx = crypto_skcipher_ctx(tfm);

	memcpy(ctx->key, key, keylen);
	ctx->keylen = keylen;

	return 0;
}
```

#### 2.5 请求处理

```c
crypto_skcipher_encrypt
	crypto_skcipher_alg(tfm)->encrypt(req)		//atmel_aes_cbc_encrypt
		atmel_aes_crypt(req, AES_FLAGS_CBC | AES_FLAGS_ENCRYPT)
	

static int atmel_aes_crypt(struct skcipher_request *req, unsigned long mode)
{
	struct crypto_skcipher *skcipher = crypto_skcipher_reqtfm(req);
	struct atmel_aes_base_ctx *ctx = crypto_skcipher_ctx(skcipher);
	struct atmel_aes_reqctx *rctx;
	struct atmel_aes_dev *aes_dd;

	switch (mode & AES_FLAGS_OPMODE_MASK) {
	default:
		ctx->block_size = AES_BLOCK_SIZE;
	ctx->is_aead = false;

	rctx = skcipher_request_ctx(req);
	rctx->mode = mode;

	return atmel_aes_handle_queue(aes_dd, &req->base);	//->
}
    
static int atmel_aes_handle_queue(struct atmel_aes_dev *aes_dd,
				  struct crypto_async_request *new_areq)
{
	struct crypto_async_request *areq;
	struct atmel_aes_base_ctx *ctx;
	bool start_async;
	int err, ret = 0;

	spin_lock_irqsave(&dd->lock, flags);
	if (new_areq)
		ret = crypto_enqueue_request(&aes_dd->queue, new_areq);
	if (dd->flags & AES_FLAGS_BUSY) {
		spin_unlock_irqrestore(&aes_dd->lock, flags);
		return ret;
	}
	areq = crypto_dequeue_request(&aes_dd->queue);
	if (areq)
		aes_dd->flags |= AES_FLAGS_BUSY;
	spin_unlock_irqrestore(&dd->lock, flags);

	if (!areq)
		return ret;

	ctx = crypto_tfm_ctx(areq->tfm);

	aes_dd->areq = areq;	
	aes_dd->ctx = ctx;		

	err = ctx->start(aes_dd);	//->atmel_aes_start
}
    
static int atmel_aes_start(struct atmel_aes_dev *aes_dd)
{
	if (use_dma)
		return atmel_aes_dma_start(aes_dd, req->src, req->dst, req->cryptlen,
					   atmel_aes_transfer_complete); //->
	else
		return atmel_aes_cpu_start(aes_dd, req->src, req->dst, req->cryptlen,
				   atmel_aes_transfer_complete);
}
  
atmel_aes_dma_start
    aes_dd->resume = atmel_aes_transfer_complete;	//
	启动加密和传输	//->
```

#### 2.6 请求完成

```c
atmel_aes_irq
	清中断
	tasklet_schedule(&aes_dd->done_task);	//atmel_aes_done_task
		aes_dd->resume		//atmel_aes_transfer_complete
			atmel_aes_complete(&aes_dd, err)
            	aes_dd->flags &= ~AES_FLAGS_BUSY
            	aes_dd->areq->complete(aes_dd->areq, err)
            	tasklet_schedule(&aes_dd->queue_task)	//atmel_aes_queue_task
            		atmel_aes_handle_queue(&aes_dd，NULL)	//注意NULL
```

#### 2.7总结

```c
//1.注册
分配struct atmel_aes_dev结构体aes_dd，然后将它和平台设备互相包含，初始化和设置两个tasklet的函数为：atmel_aes_done_task和atmel_aes_queue_task，初始化一个crypto_queue，注册中断函数atmel_aes_irq. 其中atmel_aes_irq->atmel_aes_done_task->atmel_aes_queue_task.

//2.分配设置算法对象
分配sizeof(struct crypto_skcipher)加上算法上下文大小sizeof(struct atmel_aes_ctx). 然后设置对称加密算法对象struct crypto_skcipher的reqsize和base.__crt_ctx.base.start为atmel_aes_start.

//3.分配设置请求对象
分配sizeof(struct skcipher_request)加上对称加密请求上下文大小sizeof(struct atmel_aes_reqctx). 然后设置请求对象中的src、dst、cryptlen、iv.

//4.设置秘钥
将秘钥和秘钥长度设置到算法上下文struct atmel_aes_base_ctx(其实就是struct atmel_aes_ctx)中.

//5.启动一个加密
将请求放入crypto_queue，如果加速器有在处理任务则直接返回，如果没有在处理任务则从crypto_queue取出一个请求，并标记加速器在处理任务，然后将请求的成员取出映射和设置，启动硬件加密.
    
//6.一个加密完成
触发中断，清中断，清除加速器在处理任务标志，回调请求完成函数，通过atmel_aes_handle_queue(&aes_dd，NULL)取出下一个请求，如果有请求则处理，如果没请求了则返回.
```



### 3. mtk skcipher分析记录

> mtk-platform.c mtk-aes.c

#### 3.1 注册算法

```c
mtk_crypto_probe
	struct mtk_cryp *cryp = devm_kzalloc()
	cryp->dev = &pdev->dev
	mtk_cipher_alg_register
		mtk_aes_record_init
			crypto_init_queue(&aes[i]->queue, AES_QUEUE_SIZE)
			tasklet_init(&aes[i]->queue_task, mtk_aes_queue_task)
			tasklet_init(&aes[i]->done_task, mtk_aes_done_task) 
		devm_request_irq(mtk_aes_irq)
		mtk_aes_register_algs
	platform_set_drvdata(pdev, cryp)
        
static struct skcipher_alg aes_algs[] = {
{
	.base.cra_name		= "cbc(aes)",
	.base.cra_driver_name	= "cbc-aes-mtk",
	.base.cra_priority	= 400,
	.base.cra_flags		= CRYPTO_ALG_ASYNC,
	.base.cra_blocksize	= AES_BLOCK_SIZE,
	.base.cra_ctxsize	= sizeof(struct mtk_aes_ctx),		//
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

#### 3.2 分配设置算法实例

```c
crypto_alloc_skcipher
	crypto_alloc_tfm
		crypto_alloc_tfm_node
			crypto_find_alg
			crypto_create_tfm_node
				alloc(sizeof(struct crypto_tfm) 
                      	+ frontend->tfmsize 		//offsetof(struct crypto_skcipher, base)
                      	+ frontend->extsize(alg)	//sizeof(struct mtk_aes_ctx)
				tfm->__crt_alg = alg;		//算法对象绑定算法
				frontend->init_tfm()		//crypto_skcipher_init_tfm
					alg->init		//mtk_aes_init_tfm
						crypto_skcipher_set_reqsize(sizeof(struct mtk_aes_reqctx))
						ctx->base.start = mtk_aes_start	//设置start回调
```

#### 3.3 分配设置请求

```c
skcipher_request_alloc
	kmalloc(sizeof(struct skcipher_request) + crypto_skcipher_reqsize(tfm))	//附加请求上下文size
	skcipher_request_set_tfm(req, tfm)	//请求绑定算法对象
...
skcipher_request_set_crypt(req, src, dst, cryptlen, iv)
```

#### 3.4 设置秘钥

```c
crypto_skcipher_setkey
	cipher->setkey 		//mtk_aes_setkey
		ctx->keylen = SIZE_IN_WORDS(keylen);
		memcpy(ctx->key, key, keylen);
```

#### 3.5 请求处理

```c
crypto_skcipher_encrypt
	mtk_aes_cbc_encrypt		//crypto_skcipher_alg(tfm)->encrypt(req);
		mtk_aes_crypt
			rctx->mode = AES_FLAGS_ENCRYPT | AES_FLAGS_CBC
			mtk_aes_handle_queue
				crypto_enqueue_request
				crypto_dequeue_request
				aes->flags |= AES_FLAGS_BUSY	//如果能dequeue出来一个
				aes->areq = areq
				aes->ctx = ctx
				ctx->start()		//回调mtk_aes_start
					aes->flags = (aes->flags & AES_FLAGS_BUSY) | rctx->mode
					aes->resume = mtk_aes_transfer_complete	//设置resume回调
					mtk_aes_dma(cryp, aes, req->src, req->dst, req->cryptlen)	
						aes->total = req->cryptlen
						aes->src.sg = req->src
						aes->dst.sg = req->dst
						aes->real_dst = req->dst
						mtk_aes_check_aligned
						mtk_aes_info_init
						mtk_aes_map
							dma_map_sg
							mtk_aes_xmit
```

#### 3.6 请求完成

```c
mtk_aes_irq
	清中断
	tasklet_schedule(&aes->done_task);		//mtk_aes_done_task
		mtk_aes_unmap(cryp, aes);
		aes->resume(cryp, aes);			//回调mtk_aes_transfer_complete
			mtk_aes_complete
				aes->flags &= ~AES_FLAGS_BUSY
				aes->areq->complete(aes->areq, err);
				tasklet_schedule(&aes->queue_task);		//mtk_aes_queue_task
					mtk_aes_handle_queue(aes->cryp, aes->id, NULL)	//注意NULL
```

#### 3.7 总结

&emsp;&emsp;mtk和atmel实现框架基本是一致的，因为mtk是借鉴atmel的，唯一重要的区别是第5步启动一个加密的最后的dma任务设置启动阶段。atmel使用的是标准的linux dma api，而mtk是手动设置dma，除此之外他们两个基本一样。我们的dma任务设置启动要参考mtk手动设置，因为要使用内核dma api还要花费大量的精力再去对接内核dma框架，可以留到以后考虑。



### 4. spacc cryptoapi驱动实现

> spacc-platform.c
> spacc-aes.c

#### 4.1 注册算法

&emsp;&emsp;基于原有的spacc sdk驱动包的基础上注册，但是有相应的修改。具体如下：先将elppdu.ko(pdu.c ...)、elpmem.ko(spacc_mem.c)不做修改加载，然后将elspacc.ko(spacc.c)做修改后加载，它去掉了注册所做的事，只保留EXPORT_SYMBOL导出的函数，并增加spacc_init和spacc_fini的EXPORT_SYMBOL，因为后续加载的模块spacc-crypto.ko要使用。用重新编写的spacc-platform.c替换掉elspacc.ko(spacc.c)所做的注册的事，spacc-platform.c有很多直接拷贝的spacc.c也有一些要重新实现。比如获取平台设备资源和设置struct spacc_priv、struct spacc_device全部照搬（但不一定全部使用），tasklet和irq要重新实现，而且重新定义一个struct spacc_cryp，用来包含struct spacc_priv和新的内容。接下来的注册就从spacc-platform.c讲起。

```c
static struct platform_driver spacc_crypto_driver = {
	.probe = spacc_crypto_probe,
	.remove = spacc_crypto_remove,
	.driver = {
		.name = "spacc",	//和spacc_mem.c中注册的平台设备匹配然后调用spacc_crypto_probe
		.owner = THIS_MODULE
 	},
};
module_platform_driver(spacc_crypto_driver);


spacc_crypto_probe(struct platform_device *pdev)
	struct spacc_priv *priv = devm_kzalloc();
	struct spacc_cryp *cryp = devm_kzalloc();	
	cryp->dev = &pdev->dev;				
	cryp->priv = priv;					
	platform_set_drvdata(pdev, cryp);
	cryp->base = baseaddr;						//虚拟基地址
	cryp->irq = irq;							//虚拟irq
	cryp->which_rec = NO_ATIVE;					
	spacc_init();							  //spacc硬件初始化
	spacc_irq_glbl_disable(&priv->spacc);		//关闭spacc全局中断
	spacc_cipher_record_alg_register(cryp);		
		spacc_aes_record_alg_register(cryp)
			spacc_aes_record_init(cryp)
				struct spacc_aes_rec *aesrec = &cryp->aes
				aesrec->buf = __get_free_pages
				aesrec->cryp = cryp
				spin_lock_init(&aesrec->lock);
				crypto_init_queue(&aes->queue, AES_QUEUE_SIZE);
				tasklet_init(&aes->done_task, spacc_aes_done_task);
				tasklet_init(&aes->queue_task, spacc_aes_queue_task);
			spacc_aes_register_algs
				crypto_register_skcipher(spacc_aes_algs);             
	devm_request_irq(spacc_crypto_irq);
	spacc_irq_cmdx_disable(&priv->spacc, 0);	//关闭cmd0中断
	spacc_irq_stat_wd_disable(&priv->spacc);	//关闭stat watchdog中断
	spacc_irq_stat_enable(&priv->spacc, 1);		//设置stat_cnt为1同时使能stat fifo中断
	spacc_irq_glbl_enable(&priv->spacc);		//使能spacc全局中断,其实只有stat fifo中断	


static struct skcipher_alg spacc_aes_algs[] = {
    {
        .base.cra_name		= "cbc(aes)",
        .base.cra_driver_name	= "spacc-cbc-aes",
        .base.cra_priority	= 400,
        .base.cra_flags		= CRYPTO_ALG_ASYNC,
        .base.cra_blocksize	= AES_BLOCK_SIZE,
        .base.cra_ctxsize	= sizeof(struct spacc_aes_ctx),
        .base.cra_alignmask	= 0xf,
        .base.cra_module	= THIS_MODULE,

        .min_keysize		= AES_MIN_KEY_SIZE,
        .max_keysize		= AES_MAX_KEY_SIZE,
        .setkey			= spacc_aes_setkey,
        .encrypt		= spacc_aes_cbc_encrypt,
        .decrypt		= spacc_aes_cbc_decrypt,
        .ivsize			= AES_BLOCK_SIZE,
        .init			= spacc_aes_init_tfm,
    },  
};
```

#### 4.2 分配设置算法实例

```c
//分配对象通过回调自动触发设置算法对象
crypto_alloc_skcipher
	crypto_alloc_tfm
		crypto_alloc_tfm_node
			alg = crypto_find_alg
			crypto_create_tfm_node
				tfm = alloc(sizeof(struct crypto_tfm) 
                      	+ frontend->tfmsize 		//offsetof(struct crypto_skcipher, base)
                      	+ frontend->extsize(alg)	//sizeof(struct spacc_aes_ctx)
				tfm->__crt_alg = alg			//绑定算法
				frontend->init_tfm()			  //crypto_skcipher_init_tfm
					alg->init					//spacc_aes_init_tfm
						skcipher->reqsize = sizeof(struct spacc_aes_reqctx)
						ctx->base.start = spacc_aes_start	//设置start回调
```

#### 4.3 分配设置请求

```c
//分配请求
skcipher_request_alloc(struct crypto_skcipher *tfm)
	req = kmalloc(sizeof(struct skcipher_request) + tfm->reqsize)	
	skcipher_request_set_tfm(req, tfm)	
    	req->base.tfm = crypto_skcipher_tfm(tfm)
    
//设置请求
skcipher_request_set_crypt(req, src, dst, cryptlen, iv)
skcipher_request_set_callback(req, flags, callbackfunc, callbackdata)    
```

#### 4.4 设置秘钥

```c
crypto_skcipher_setkey
	cipher->setkey 		//spacc_aes_setkey
		ctx->keylen = keylen;
		memcpy(ctx->key, key, keylen);
```

#### 4.5 请求处理

```c
crypto_skcipher_encrypt(struct skcipher_request *req)
	spacc_aes_cbc_encrypt(req)	
		spacc_aes_crypt(req, AES_FLAGS_ENCRYPT | AES_FLAGS_CBC)	
			ctx->cryp = spacc_aes_find_dev					//
			rctx->mode = AES_FLAGS_ENCRYPT | AES_FLAGS_CBC
			spacc_aes_handle_queue(ctx->cryp, &req->base)
				crypto_enqueue_request		// if newreq
				areq = crypto_dequeue_request	
				aesrec->flags |= AES_FLAGS_BUSY	//if areq exist
				aesrec->areq = areq
				aesrec->ctx = ctx
				ctx->start(cryp)	//cryp == ctx->cryp	//spacc_aes_start
					aesrec->flags = (aesrec->flags & AES_FLAGS_BUSY) | rctx->mode
					cryp->which_rec = AES_ATIVE
					aesrec->aes->resume = spacc_aes_transfer_complete;	//设置resume回调
					spacc_aes_dma(cryp, req->src, req->dst, req->cryptlen)
						aesrec->total =  req->cryptlen
						aesrec->src.sg = req->src
						aesrec->dst.sg = req->dst
						aesrec->real_dst = req->dst
						spacc_aes_check_aligned
						sg_copy_to_buffer	// if !src_aligned
						sg_init_table		// if !src_aligned || !dst_aligned
						sg_set_buf			// if !src_aligned || !dst_aligned
						spacc_aes_map_xmit(cryp, aesrec, req->cryptlen + padlen)		    
							enc_mode = rctx->mode & AES_FLAGS_CIPHER_MSK
							enc_or_dec = !!(AES_FLAGS_ENCRYPT | rctx->mode)
							spacc_open
							spacc_write_context	
							spacc_set_operation
							spacc_set_key_exp	//if dec
							dma_map_sg
							spacc_aes_xmit(cryp, aesrec, len + padlen)	//len=req->cryptlen
								pdu_ddt_init	
								pdu_ddt_add
								spacc_packet_enqueue_ddt
```

#### 4.6 请求完成

```c
spacc_crypto_irq(irq, cryp)
	清中断
	tasklet_schedule(&aesrec->done_task)		//spacc_aes_done_task
		spacc_pop_packets
		spacc_packet_dequeue
		pdu_ddt_free
		dma_unmap_sg
		spacc_aes_restore_sg	// if aligned
		sg_copy_from_buffer		// if !dst_aligned
		spacc_close
		aesrec->ctx->resume(cryp)	//spacc_aes_transfer_complete
			spacc_aes_complete(cryp, 0)
				aesrec->flags &= ~AES_FLAGS_BUSY
				aesrec->areq->complete(aesrec->areq, err)
				tasklet_schedule(&aesrec->queue_task) 		//spacc_aes_queue_task
					spacc_aes_handle_queue(cryp, NULL) 
	cryp->which_rec = NO_ATIVE
```

#### 4.7 总结

&emsp;&emsp;spacc的aes crypto api整体上的软件框架是参考mtk的，主要区别在于结合了spacc sdk的一些注册和硬件操作。注册过程保留了sdk中对struct spacc_priv以及其子结构spacc_device的设置，因为后续的硬件操作需要。硬件操作按照sdk方式操作，如启动加密作业中调用的spacc_open、spacc_write_context、spacc_set_operation、spacc_set_key_exp、pdu_ddt_init、pdu_ddt_add、spacc_packet_enqueue_ddt；以及加密作业完成中调用的spacc_packet_dequeue、pdu_ddt_free、spacc_close。通过使用这些函数大大减少了重新写代码操作寄存器带来的巨大的工作量，但是这些函数的使用方法要严格按照文档和sdk的说明来使用，最后在遵守了文档和sdk的规范之外还要将这些函数正确的融合进linux crypto子系统之中。



### 5.  spacc cryptoapi驱动编译

&emsp;&emsp;spacc cryptoapi驱动目前只有spacc-platform.c、spacc-platform.h、spacc-aes.c三个代码源文件，以后会根据功能新增或改动。这些文件会编译得到一个spacc-crypto.ko驱动模块。这三个源文件是加入spacc sdk中和其他模块以及测试程序一起编译的。下面就是讲如何将spacc cryptoapi文件加入sdk和编译的过程。

#### 5.1 下载和修改sdk

&emsp;&emsp;首先从公司gitlab下载sdk到247中，比如我的路径是/home/zye/spacc。如下：

```c
[zye@cr2 spacc]$ pwd
/home/zye/spacc

[zye@cr2 spacc]$ ls
bin  doc  Makefile  Makefile.barespacc  README.TXT  src
```

&emsp;&emsp;然后要根据自己下载的路径修改以下两个地方：

```c
//   src/pdu/linux/kernel/Makefile
#KERNELDIR ?= XXXXXXXXXXXXXXXXXXXXXXXXXX
KERNELDIR ?= /home/zye/kernel

//   src/core/kernel/Makefile
#KERNELDIR ?= XXXXXXXXXXXXXXXXXXXXXXXXXX
KERNELDIR ?= /home/zye/kernel
```

#### 5.2 将spacc cryptoapi合入sdk

&emsp;&emsp;首先spacc-platform.c、spacc-platform.h、spacc-aes.c放入src/core/kernel。   
&emsp;&emsp;然后修改src/pdu/linux/kernel/spacc_mem.c如下，更改spacc访问的ddr地址重映射和增加spacc时钟复位。

```c
...
#define SPACC_REMAP_SET 	  0x4
//#define SPACC_REMAP_SET 	  0xc
...
/* 复位设置spacc时钟 */
void __iomem *prcm_map = NULL;
u32 reg;

prcm_map = ioremap(0x020a0000, 0x10000);
if (prcm_map == NULL) {
    pr_err("Can't map prcm.\n");
    return -1;
}

reg = pdu_io_read32(prcm_map + 0xd0);
reg |= (1<<26);
pdu_io_write32(prcm_map + 0xd0, reg);

reg = pdu_io_read32(prcm_map + 0x06e0);
reg &= ~(1<<3);
pdu_io_write32(prcm_map + 0x06e0, reg);

iounmap(prcm_map);
pr_info("Spacc clock reset done.\n"); 

rc = spdu_init(vex_baseaddr, &info);	
...
```

&emsp;&emsp;然后修改src/core/kernel/spacc.c如下，将必要的工作放到spacc-platform.c中，同时要对其他模块导出spacc_init和spacc_fini这样spacc-platform.c才能使用。

```c
static int __init spacc_mod_init (void)
{
   int err = 0;
#if 0
   err = misc_register(&spaccirq_device);
   if (err) {
      return err;
   }

   err = platform_driver_register(&spacc_driver);
   if (err) {
      misc_deregister(&spaccirq_device);
   }
#endif   
   return err;
}

static void __exit spacc_mod_exit (void)
{
#if 0
   misc_deregister (&spaccirq_device);
   platform_driver_unregister(&spacc_driver);
#endif   
}

...

EXPORT_SYMBOL (spacc_ctx_request);
EXPORT_SYMBOL (spacc_ctx_release);

//新加给spacc-crypto
EXPORT_SYMBOL (spacc_init);
EXPORT_SYMBOL (spacc_fini);

module_param(spacc_endian, int, 0);
MODULE_PARM_DESC(spacc_endian, "Endianess of data transfers (0==little)");
EXPORT_SYMBOL(spacc_endian);

...
```


&emsp;&emsp;然后修改src/core/kernel/Makefile如下：

```c
...
#obj-m += elpspacccrypt.o
#elpspacccrypt-objs := cryptoapi.o cryptoapi-ahash.o  cryptoapi-aead.o

//这两行是新加
obj-m += spacc-crypto.o
spacc-crypto-objs := spacc-platform.o spacc-aes.o

ccflags-$(CONFIG_SPACC_DEBUG) := -DDEBUG
...
```

&emsp;&emsp;然后修改bin/load.sh和/bin/unload.sh如下：

```c
//   bin/load.sh
#!/bin/bash
insmod elppdu.ko
insmod elpmem.ko
insmod elpspacc.ko
insmod spacc-crypto.ko

//   bin/unload.sh
#!/bin/bash
for f in spacc-crypto.ko elpspacc.ko elpmem.ko elppdu.ko; do
   if [ -e $f ]; then rmmod $f; fi
done
```

&emsp;&emsp;最后就可以编译和拷贝了：

```c
//编译
source /opt/arm.env
cd /home/zye/spacc
make clean
make

//拷贝编译得到的模块和测试程序到根文件系统cpio镜像中
将/home/zye/spacc/bin目录的编译得到的文件拷贝到解压的cpio的usr/local/spacc中
```



### 6.  spacc 应用层访问方式

&emsp;&emsp;内核cryptoapi框架不仅可为内核提供加解密服务，还可通过特定接口为用户态提供相应的加解密服务。为此内核专门为其实现了一种socket协议af_alg，该协议采用算法类型和算法名作为网络地址，且使用标准的socket编程接口。

#### 6.1 直接操作af_alg协议套接字使用cryptoapi

```c
struct sockaddr_alg sa = {
    .salg_family = AF_ALG,
    .salg_type = "skcipher",			//算法类型
    .salg_name = "spacc-cbc-aes"		//算法名
};
tfmfd = socket(AF_ALG, SOCK_SEQPACKET, 0);	//__sys_socket
	//alg_create(sock->ops = &alg_proto_ops)
bind(tfmfd, (struct sockaddr *)&sa, sizeof(sa));	//__sys_bind		//type:algif_type_skcipher
	//alg_bind->skcipher_bind->crypto_alloc_skcipher
setsockopt(tfmfd, SOL_ALG, ALG_SET_KEY, key, keylen);	//__sys_setsockopt
	//alg_setsockopt->alg_setkey->crypto_skcipher_setkey
opfd = accept(tfmfd, NULL, 0);	//__sys_accept4
	//alg_accept->af_alg_accept(sock->ops = &algif_skcipher_ops)
...		
sendmsg(opfd, &msg, 0);		//__sys_sendmsg
	//sock_sendmsg->skcipher_sendmsg
read(opfd, output, srclen);	//ksys_read
	//sock_read_iter->sock_recvmsg->skcipher_recvmsg->alloc request
close(opfd);	//ksys_close
close(tfmfd);

//socket_file_ops
//PIPE_DEF_BUFFERS
//alloc_pipe_info
```

&emsp;&emsp;通过套接字接口会调用到前文的内核cryptoapi接口，比如bind对应的是内核分配设置算法实例接口skcipher_request_alloc。

#### 6.2 通过开源软件库的api使用cryptoapi

```c
/* openssl开源库的常用高级加密evp接口的模版 */

ENGINE_load_builtin_engines();
ENGINE_load_dynamic();
ENGINE *engine = ENGINE_by_id("afalg");
ENGINE_init(engine);
ENGINE_set_default(engine, ENGINE_METHOD_ALL);		
ENGINE_register_all_complete();

EVP_CIPHER *cipher = EVP_get_cipherbyname("AES-128-CBC");	//openssl的名字和注册内核的不一样
EVP_CIPHER_CTX *ctx = EVP_CIPHER_CTX_new();
EVP_EncryptInit_ex(ctx, cipher, NULL, key, iv);
EVP_EncryptUpdate(ctx, output_encrypt, &outlen, input, input_len);
EVP_EncryptFinal_ex(ctx, output_encrypt + outlen, &outlen);
EVP_CIPHER_CTX_free(ctx);

ENGINE_finish(engine);
ENGINE_free(engine);
```

&emsp;&emsp;这种方法本质上也是使用了af_alg协议套接字，因为这是应用层访问内核cryptoapi的必通之路，只是开源库封装了使用户看不到，但是经过测试和openssl afalg引擎源代码e_afalg.c的确认，**openssl afalg引擎目前只能对接cbc(aes)模式**。

#### 6.3 通过sdk api使用设备节点不使用cryptoapi

```c
spacc_dev_open(fd, cipher_mode, hash_mode, encrypt, icvmode, icv_len, aad_copy, 
                   ckey, ckeylen, civ, civlen, hkey, hkeylen, hiv, hivlen)；             
spacc_dev_process(fd, new_iv, iv_offset, pre_aad, post_aad, src_offset, dst_offset, 
                  src, src_len, dst, dst_len)；
spacc_dev_close(fd);           
```

&emsp;&emsp;本质是通过使用sdk的字符设备驱动导出的字符设备来直接操作spacc，没有经过cryptoapi框架，软件通路相对cryptoapi更少，但是应用层的访问没有标准化。

### 7. hash分析记录

#### 7.1 mtk

```c
mtk_hash_alg_register(cryp)
    list_add_tail(&cryp->sha_list, &mtk_sha.dev_list);

ahashtfm = crypto_alloc_ahash(algname)
	crypto_ahash_init_tfm(tfm)				//frontend->init_tfm
    	ahashtfm->init = mtk_sha_init		//alg->init
    	ahashtfm->update = mtk_sha_update	//alg->update
    	...
	mtk_sha_cra_init(tfm)			//alg->cra_init(tfm)
    	crypto_ahash_set_reqsize	//设置ahashtfm->reqsize
	
ahashreq = ahash_request_alloc(ahashtfm)    
    ahash_request_set_tfm(ahashreq, ahashtfm)
  
//init    
crypto_ahash_init(ahashreq)	
    mtk_sha_init(ahashreq)			//ahashtfm->init
		rctx->xx = yy
		
//update
crypto_ahash_update(ahashreq)  
    mtk_sha_update(ahashreq)		//ahashtfm->update
		rctx->xx = yy
		mtk_sha_enqueue(ahashreq, SHA_OP_UPDATE)
    
//final
crypto_ahash_final(ahashreq)
    mtk_sha_final(ahashreq)		//ahashtfm->final
    	rctx->flags |= SHA_FLAGS_FINUP
    	mtk_sha_enqueue(ahashreq, SHA_OP_FINAL)
    
//finup(optional)
crypto_ahash_finup(ahashreq)  
    
//digest(optional)
crypto_ahash_digest(ahashreq)   
```

#### 7.2 atmel

```c
//分配设置算法实例
crypto_alloc_ahash
	crypto_alloc_tfm
		crypto_alloc_tfm_node
			crypto_find_alg
			crypto_create_tfm_node
				alloc(sizeof(struct crypto_tfm) 
                      	+ frontend->tfmsize 		//offsetof(struct crypto_ahash, base)
                      	+ frontend->extsize(alg)	//sizeof(struct atmel_sha_ctx)
				tfm->__crt_alg = alg;		//
				frontend->init_tfm()		//crypto_ahash_init_tfm
                     crypto_ahash_init_tfm(tfm)
                         ahashtfm->init = atmel_sha_init		//alg->init
                         ahashtfm->update = atmel_sha_update	//alg->update
                         ...
                     atmel_sha_cra_init(tfm)			//alg->cra_init(tfm)
                         struct atmel_sha_ctx *ctx = crypto_tfm_ctx(tfm);
                         crypto_ahash_set_reqsize(__crypto_ahash_cast(tfm),
                                 sizeof(struct atmel_sha_reqctx));	//
                         ctx->start = atmel_sha_start;			//                   
                      
//分配设置请求
req = ahash_request_alloc(tfm)
	kmalloc(sizeof(struct ahash_request) + crypto_ahash_reqsize(tfm))
	ahash_request_set_tfm(req, tfm)	//请求绑定算法对象
...
ahash_request_set_crypt(req, src, result, nbytes)   
                      
//init
crypto_ahash_init(req)
	atmel_sha_init(req)	//tfm->init
         ctx->dd = dd 
         ctx->flags = 0
         ctx->flags |= SHA_FLAGS_SHAX
         ctx->block_size = SHAX_BLOCK_SIZE
		ctx->bufcnt = 0
		ctx->digcnt[0] = 0
		ctx->digcnt[1] = 0
		ctx->buflen = SHA_BUFFER_LEN
		return 0;

//update
crypto_ahash_update(req)
	atmel_sha_update(req)		//tfm->update
         ctx->total = req->nbytes		//
         ctx->sg = req->src
         ctx->offset = 0
         //如果buffer还够空间存放本次req则存放到buffer就返回             
		if (ctx->bufcnt + ctx->total < ctx->buflen) {
			atmel_sha_append_sg(ctx)	
			return 0;
		}
         //否则             
		atmel_sha_enqueue(req, SHA_OP_UPDATE)	
         	ctx->op = SHA_OP_UPDATE		//
         	atmel_sha_handle_queue
         		atmel_sha_start
         			dd->resume = atmel_sha_done		//
         			err = atmel_sha_update_req(dd)		
         			if (!err)
                     	 atmel_sha_finish_req(req, err)	
                                        
//final
crypto_ahash_final(ahashreq)  
	atmel_sha_final(ahashreq)
		ctx->flags |= SHA_FLAGS_FINUP				//
		if (ctx->flags & SHA_FLAGS_ERROR)
			return 0;
		if (ctx->flags & SHA_FLAGS_PAD)
             return atmel_sha_finish(req);        
		atmel_sha_enqueue(req, SHA_OP_FINAL)
			ctx->op = SHA_OP_FINAL		//
			atmel_sha_handle_queue
				atmel_sha_start
					dd->resume = atmel_sha_done
					err = atmel_sha_final_req(dd)
					if (!err)
						atmel_sha_finish_req(req, err);		
                      
//finup(update + final)
crypto_ahash_finup(ahashreq)  
	atmel_sha_finup(ahashreq) 
		ctx->flags |= SHA_FLAGS_FINUP		//
		err1 = atmel_sha_update(req)
         if (err1 == -EINPROGRESS ||
             (err1 == -EBUSY && (ahash_request_flags(req) &
                     CRYPTO_TFM_REQ_MAY_BACKLOG)))
			return err1;  
		err2 = atmel_sha_final(req)
		return err1 ?: err2;
                      
//digest(init + update + final)
crypto_ahash_digest(ahashreq)
	atmel_sha_init(req) ?: atmel_sha_finup(req)          
```

#### 7.3 直接操作af_alg协议套接字使用cryptoapi

```c
struct sockaddr_alg sa = {
    .salg_family = AF_ALG,
    .salg_type = "skcipher",			//算法类型
    .salg_name = "hmac(sha1)"		//算法名
};
tfmfd = socket(AF_ALG, SOCK_SEQPACKET, 0);	
	//alg_create
		//sock->ops = &alg_proto_ops
		//sock_init_data(sock, sk)
bind(tfmfd, (struct sockaddr *)&sa, sizeof(sa));	//type:algif_type_hash
	//alg_bind
		//type = algif_type_hash
		//private = hash_bind->crypto_alloc_ahash
		//ask->type = type
		//ask->private = private
setsockopt(tfmfd, SOL_ALG, ALG_SET_KEY, key, keylen);
	//alg_setsockopt->hash_setkey->crypto_ahash_setkey
opfd = accept(tfmfd, NULL, 0);
	//alg_accept
		//af_alg_accept
			//newsock->ops = &algif_skcipher_ops
			//hash_accept_parent
				//newask->private = ctx;
				//ahash_request_set_tfm
				//ahash_request_set_callback
write(opfd, &msg, 0);
	//sock_write_iter->sock_sendmsg->hash_sendmsg
read(opfd, output, srclen);
	//sock_read_iter->sock_recvmsg->hash_recvmsg
close(opfd);
close(tfmfd);
```

