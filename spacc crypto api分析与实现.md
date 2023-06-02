## <center>spacc crypto api分析与实现</center>

### 1. atmel加速器代码分析记录

> atmel-aes.c

#### 1.1 注册算法

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

#### 1.2 分配设置算法对象

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

#### 1.3 分配设置请求

```c
skcipher_request_alloc
	kmalloc(sizeof(struct skcipher_request) + crypto_skcipher_reqsize(tfm))
	skcipher_request_set_tfm(req, tfm)	//请求绑定算法对象
...
skcipher_request_set_crypt(req, src, dst, cryptlen, iv)
```

#### 1.4 设置秘钥

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

#### 1.5 启动一个加密

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

#### 1.6 一个加密完成

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

#### 1.7总结

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



### 2. mtk加速器代码分析记录

> mtk-platform.c mtk-aes.c

#### 2.1 注册算法

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

#### 2.2 分配设置算法对象

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

#### 2.3 分配设置请求

```c
skcipher_request_alloc
	kmalloc(sizeof(struct skcipher_request) + crypto_skcipher_reqsize(tfm))	//附加请求上下文size
	skcipher_request_set_tfm(req, tfm)	//请求绑定算法对象
...
skcipher_request_set_crypt(req, src, dst, cryptlen, iv)
```

#### 2.4 设置秘钥

```c
crypto_skcipher_setkey
	cipher->setkey 		//mtk_aes_setkey
		ctx->keylen = SIZE_IN_WORDS(keylen);
		memcpy(ctx->key, key, keylen);
```

#### 2.5 启动一个加密

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

#### 2.6 一个加密完成

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

#### 2.7 总结

&emsp;&emsp;mtk和atmel实现框架基本是一致的，因为mtk是借鉴atmel的，唯一重要的区别是第5步启动一个加密的最后的dma任务设置启动阶段。atmel使用的是标准的linux dma api，而mtk是手动设置dma，除此之外他们两个基本一样。我们的dma任务设置启动要参考mtk手动设置，因为要使用内核dma api还要花费大量的精力再去对接内核dma框架，可以留到以后考虑。



### 3. spacc crypto api实现记录

> spacc-platform.c
> spacc-aes.c

#### 3.1 注册算法

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
	spacc_init();							  //spacc硬件初始化
	spacc_irq_glbl_disable(&priv->spacc);		//关闭spacc全局中断
	spacc_cipher_record_alg_register(cryp);		
		spacc_aes_record_alg_register(cryp)
			spacc_aes_record_init(cryp)
				struct spacc_aes_rec *aesrec = &cryp->aes
				aesrec->buf = __get_free_pages
				aesrec->cryp = cryp
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

#### 3.2 分配设置算法对象

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

#### 3.3 分配设置请求

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

#### 3.4 设置秘钥

```c
crypto_skcipher_setkey
	cipher->setkey 		//spacc_aes_setkey
		ctx->keylen = keylen;
		memcpy(ctx->key, key, keylen);
```

#### 3.5 启动一个加密

```c
crypto_skcipher_encrypt(struct skcipher_request *req)
	spacc_aes_cbc_encrypt(req)	
		spacc_aes_crypt(req, AES_FLAGS_ENCRYPT | AES_FLAGS_CBC)	
			ctx->cryp = spacc_aes_find_dev
			rctx->mode = AES_FLAGS_ENCRYPT | AES_FLAGS_CBC
			spacc_aes_handle_queue(ctx->cryp, &req->base)
				crypto_enqueue_request
				crypto_dequeue_request
				aesrec->flags |= AES_FLAGS_BUSY
				aesrec->areq = areq
				aesrec->ctx = ctx
				ctx->start(cryp)	//cryp == ctx->cryp	//spacc_aes_start
					aesrec->flags = (aesrec->flags & AES_FLAGS_BUSY) | rctx->mode
					cryp->which_rec = 0
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
							spacc_set_key_exp	//if !enc_or_dec
							dma_map_sg
							spacc_aes_xmit(cryp, aesrec, len + padlen)	//len=req->cryptlen
								pdu_ddt_init	
								pdu_ddt_add
								spacc_packet_enqueue_ddt
```

#### 3.6 一个加密完成

```c
spacc_crypto_irq(irq, cryp)
	清中断
	tasklet_schedule(&aesrec->done_task)		//spacc_aes_done_task
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
```

#### 3.7 总结

&emsp;&emsp;spacc的aes crypto api整体上的软件框架是参考mtk的，主要区别在于结合了spacc sdk的一些注册和硬件操作。注册过程保留了sdk中对struct spacc_priv以及其子结构spacc_device的设置，因为后续的硬件操作需要。硬件操作按照sdk方式操作，如启动加密作业中调用的spacc_open、spacc_write_context、spacc_set_operation、spacc_set_key_exp、pdu_ddt_init、pdu_ddt_add、spacc_packet_enqueue_ddt；以及加密作业完成中调用的spacc_packet_dequeue、pdu_ddt_free、spacc_close。通过使用这些函数大大减少了重新写代码操作寄存器带来的巨大的工作量，但是这些函数的使用方法要严格按照文档和sdk的说明来使用，最后在遵守了文档和sdk的规范之外还要将这些函数正确的融合进linux crypto子系统之中。