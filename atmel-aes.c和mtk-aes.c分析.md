## <center>atmel和mtk加速器crypto api实现分析记录</center>

### 1. atmel-aes.c

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
	atmel_aes_buff_init(aes_dd);
	atmel_aes_dma_init(aes_dd);
	atmel_aes_register_algs(aes_dd);	//注册算法
}

{
	.base.cra_name		= "cbc(aes)",
	.base.cra_driver_name	= "atmel-cbc-aes",
	.base.cra_blocksize	= AES_BLOCK_SIZE,
	.base.cra_ctxsize	= sizeof(struct atmel_aes_ctx),		//

	.init			= atmel_aes_init_tfm,
	.min_keysize		= AES_MIN_KEY_SIZE,
	.max_keysize		= AES_MAX_KEY_SIZE,
	.setkey			= atmel_aes_setkey,
	.encrypt		= atmel_aes_cbc_encrypt,
	.decrypt		= atmel_aes_cbc_decrypt,
	.ivsize			= AES_BLOCK_SIZE,
},
```

#### 1.2 申请算法对象

```c
crypto_alloc_skcipher
	crypto_skcipher_init_tfm	//crypto_skcipher_type->init_tfm
		atmel_aes_init_tfm	//alg->init

static int atmel_aes_init_tfm(struct crypto_skcipher *tfm)
{
	struct atmel_aes_ctx *ctx = crypto_skcipher_ctx(tfm);

	crypto_skcipher_set_reqsize(tfm, sizeof(struct atmel_aes_reqctx));
	ctx->base.start = atmel_aes_start;

	return 0;
}
```

#### 1.3 设置秘钥

```c
crypto_skcipher_setkey
	atmel_aes_setkey	//cipher->setkey

static int atmel_aes_setkey(struct crypto_skcipher *tfm, const u8 *key,
			   unsigned int keylen)
{
	struct atmel_aes_base_ctx *ctx = crypto_skcipher_ctx(tfm);

	memcpy(ctx->key, key, keylen);
	ctx->keylen = keylen;

	return 0;
}
```

#### 1.4 启动一个加密

```c
skcipher_request_alloc
	kmalloc(sizeof(struct skcipher_request) + crypto_skcipher_reqsize(tfm))
...
skcipher_request_set_crypt(req, src, dst, cryptlen, iv)
    
crypto_skcipher_encrypt
	atmel_aes_cbc_encrypt		//crypto_skcipher_alg(tfm)->encrypt(req);
	
static int atmel_aes_cbc_encrypt(struct skcipher_request *req)
{
	return atmel_aes_crypt(req, AES_FLAGS_CBC | AES_FLAGS_ENCRYPT);
}	

static int atmel_aes_crypt(struct skcipher_request *req, unsigned long mode)
{
	struct crypto_skcipher *skcipher = crypto_skcipher_reqtfm(req);
	struct atmel_aes_base_ctx *ctx = crypto_skcipher_ctx(skcipher);
	struct atmel_aes_reqctx *rctx;
	struct atmel_aes_dev *dd;

	switch (mode & AES_FLAGS_OPMODE_MASK) {
	default:
		ctx->block_size = AES_BLOCK_SIZE;
	ctx->is_aead = false;

	rctx = skcipher_request_ctx(req);
	rctx->mode = mode;

	return atmel_aes_handle_queue(dd, &req->base);
}
    
static int atmel_aes_handle_queue(struct atmel_aes_dev *dd,
				  struct crypto_async_request *new_areq)
{
	struct crypto_async_request *areq;
	struct atmel_aes_base_ctx *ctx;
	bool start_async;
	int err, ret = 0;

	spin_lock_irqsave(&dd->lock, flags);
	if (new_areq)
		ret = crypto_enqueue_request(&dd->queue, new_areq);
	if (dd->flags & AES_FLAGS_BUSY) {
		spin_unlock_irqrestore(&dd->lock, flags);
		return ret;
	}
	areq = crypto_dequeue_request(&dd->queue);
	if (areq)
		dd->flags |= AES_FLAGS_BUSY;
	spin_unlock_irqrestore(&dd->lock, flags);

	if (!areq)
		return ret;

	ctx = crypto_tfm_ctx(areq->tfm);

	dd->areq = areq;
	dd->ctx = ctx;

	err = ctx->start(dd);
}
    
static int atmel_aes_start(struct atmel_aes_dev *dd)
{
	if (use_dma)
		return atmel_aes_dma_start(dd, req->src, req->dst, req->cryptlen,
					   atmel_aes_transfer_complete);
	else
		return atmel_aes_cpu_start(dd, req->src, req->dst, req->cryptlen,
				   atmel_aes_transfer_complete);
}
  
atmel_aes_dma_start
    dd->resume = atmel_aes_transfer_complete;
	启动加密和传输
```

#### 1.5 一个加密完成

```c
atmel_aes_irq
	tasklet_schedule(&aes_dd->done_task);	//atmel_aes_done_task
	
atmel_aes_done_task
	dd->resume	//atmel_aes_transfer_complete
		atmel_aes_complete
		
static inline int atmel_aes_complete(struct atmel_aes_dev *dd, int err)
{
	dd->areq->complete(dd->areq, err);
	tasklet_schedule(&dd->queue_task);		//atmel_aes_queue_task
}		

atmel_aes_queue_task
	atmel_aes_handle_queue
```

#### 1.6总结



### 2. mtk-aes.c

#### 2.1 注册

```c
mtk_crypto_probe
	mtk_cipher_alg_register
		mtk_aes_record_init
			crypto_init_queue(&aes[i]->queue, AES_QUEUE_SIZE)
			tasklet_init(&aes[i]->queue_task, mtk_aes_queue_task,)
			tasklet_init(&aes[i]->done_task, mtk_aes_done_task, ) 
		devm_request_irq(mtk_aes_irq)
		mtk_aes_register_algs
```

#### 2.2 申请算法对象

```c
crypto_alloc_skcipher
	crypto_skcipher_init_tfm	//crypto_skcipher_type->init_tfm
		mtk_aes_init_tfm		//alg->init
		
static int mtk_aes_init_tfm(struct crypto_skcipher *tfm)
{
	struct mtk_aes_ctx *ctx = crypto_skcipher_ctx(tfm);

	crypto_skcipher_set_reqsize(tfm, sizeof(struct mtk_aes_reqctx));
	ctx->base.start = mtk_aes_start;		//
	return 0;
}		
```

#### 2.3 设置秘钥

```c
crypto_skcipher_setkey
	mtk_aes_setkey	//cipher->setkey
		ctx->keylen = SIZE_IN_WORDS(keylen);
		memcpy(ctx->key, key, keylen);
	
```

#### 2.4 启动一个加密

```c
crypto_skcipher_encrypt
	mtk_aes_cbc_encrypt		//crypto_skcipher_alg(tfm)->encrypt(req);
		mtk_aes_crypt
			mtk_aes_handle_queue
				crypto_enqueue_request
				crypto_dequeue_request
					ctx->start()		//mtk_aes_start
						aes->resume = mtk_aes_transfer_complete;
						mtk_aes_dma

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

	return mtk_aes_xmit(cryp, aes);
}
```

