## <center>crypto api实现分析与设计</center>

### 1. atmel加速器分析记录

> atmel-aes.c

#### 1.1 注册

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
				frontend->init_tfm()		//crypto_skcipher_init_tfm
					alg->init		//atmel_aes_init_tfm
						crypto_skcipher_set_reqsize(sizeof(struct atmel_aes_reqctx))
						ctx->base.start = atmel_aes_start	//
```

#### 1.3 分配设置请求对象

```c
skcipher_request_alloc
	kmalloc(sizeof(struct skcipher_request) + crypto_skcipher_reqsize(tfm))
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

	err = ctx->start(aes_dd);	//->
}
    
static int atmel_aes_start(struct atmel_aes_dev *aes_dd)
{
	if (use_dma)
		return atmel_aes_dma_start(aes_dd, req->src, req->dst, req->cryptlen,
					   atmel_aes_transfer_complete);
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
触发中断，清除加速器在处理任务标志，回调请求完成函数，通过atmel_aes_handle_queue(&aes_dd，NULL)取出下一个请求，如果有请求则处理，如果没请求了则返回.
```



### 2. mtk加速器分析记录

> mtk-platform.c mtk-aes.c

#### 2.1 注册

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
				frontend->init_tfm()		//crypto_skcipher_init_tfm
					alg->init		//mtk_aes_init_tfm
						crypto_skcipher_set_reqsize(sizeof(struct mtk_aes_reqctx))
						ctx->base.start = mtk_aes_start	//
```

#### 2.3 分配设置请求对象

```c
skcipher_request_alloc
	kmalloc(sizeof(struct skcipher_request) + crypto_skcipher_reqsize(tfm))
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
			mtk_aes_handle_queue
				crypto_enqueue_request
				crypto_dequeue_request
					ctx->start()		//mtk_aes_start
						aes->resume = mtk_aes_transfer_complete;
						mtk_aes_dma		//->

static int mtk_aes_map(struct mtk_cryp *cryp, struct mtk_aes_rec *aes)
{
	if (aes->src.sg == aes->dst.sg) {
		aes->src.sg_len = dma_map_sg(cryp->dev, aes->src.sg, aes->src.nents,
					     			DMA_BIDIRECTIONAL);
		aes->dst.sg_len = aes->src.sg_len;
	} else {
		aes->src.sg_len = dma_map_sg(cryp->dev, aes->src.sg,
					    		 	aes->src.nents, DMA_TO_DEVICE);
		aes->dst.sg_len = dma_map_sg(cryp->dev, aes->dst.sg,
					     			aes->dst.nents, DMA_FROM_DEVICE);
	}

	return mtk_aes_xmit(cryp, aes);		//->
}
```

#### 2.6 一个加密完成

```c
mtk_aes_irq
	清中断
	tasklet_schedule(&aes->done_task);		//mtk_aes_done_task
		mtk_aes_unmap(cryp, aes);
		aes->resume(cryp, aes);			//mtk_aes_transfer_complete
			mtk_aes_complete
				aes->flags &= ~AES_FLAGS_BUSY
				aes->areq->complete(aes->areq, err);
				tasklet_schedule(&aes->queue_task);		//mtk_aes_queue_task
					mtk_aes_handle_queue(aes->cryp, aes->id, NULL)	//注意NULL
```

#### 2.7 总结

&emsp;&emsp;mtk和atmel实现框架基本是一致的，因为mtk是借鉴atmel的，唯一重要的区别是第5步启动一个加密的最后的dma任务设置启动阶段。atmel使用的是标准的linux dma api，而mtk是手动设置dma，除此之外他们两个基本一样。我们的dma任务设置启动要参考mtk手动设置，因为要使用内核dma api还要花费大量的精力再去对接内核dma框架，可以留到以后考虑。



### 3. spacc加速器crypto api设计

#### 3.1 注册

> 