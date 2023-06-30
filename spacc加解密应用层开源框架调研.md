## <center>spacc加解密应用层开源框架调研</center>

[TOC]

### 一. 加解密应用开源框架概述

&emsp;&emsp;linux流行的通用加解密开源框有openssl，gnupg，dm-crypt等。其中openssl是一个最为通用和广泛使用的开源框架，其他的开源框架一般侧重某些应用场景。  
&emsp;&emsp;openssl可以通过af_alg套接字和/dev/crypto设备节点使用linux crypto api框架注册的算法。/dev/crypto方式兼容OpenBSD cryptodev user-space API，需要额外下载安装cryptodev-linux，af_alg套接字方式不需要额外安装，较为方便。openssl提供了一个通用的加解密库libcrypto，一个tls协议库libssl和命令行工具openssl。其中libcrypto就是应用层使用加解密的核心组件，可以单独使用。



### 二. openssl配置编译安装

```c
/* 下载openssl源码 */
从https://www.openssl.org/source/ 下载openssl-3.1.1.tar.gz(经过我反复测试只有这个或更新版本能使能linux-aarch64的afalg引擎)
tar -xvf openssl-3.1.1.tar.gz
cd openssl-3.1.1/ 
    
/* 配置，使能afalg引擎 */
./Configure linux-aarch64 enable-afalgeng --cross-compile-prefix=aarch64-none-linux-gnu- 
 或者
./Configure linux-aarch64 --cross-compile-prefix=aarch64-none-linux-gnu- shared enable-engine enable-dso enable-afalgeng

/* 编译 */
source /opt/arm.env
make -j4

/* 安装 */
//make install DESTDIR=/home/syshaps/workspace/zye/openssl-target
make install DESTDIR=/home/yeshen/workspace/openssl/target

/* 拷贝头文件和库文件到交叉编译工具链目录(参考pkgconfig中的.pc后缀文件) */
//sudo cp -r /home/syshaps/workspace/zye/openssl-target/usr/local/include/openssl /opt/gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu/aarch64-none-linux-gnu/libc/usr/include
//sudo cp -r /home/syshaps/workspace/zye/openssl-target/usr/local/lib/libcrypto.* /opt/gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu/aarch64-none-linux-gnu/libc/usr/lib64
//sudo cp -r /home/syshaps/workspace/zye/openssl-target/usr/local/lib/libssl.* /opt/gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu/aarch64-none-linux-gnu/libc/usr/lib64
cp -r /home/yeshen/workspace/openssl/target/usr/local/include/openssl /home/yeshen/workspace/toolchain/aarch64-none-linux-gnu/libc/usr/include
cp -r /home/yeshen/workspace/openssl/target/usr/local/lib/* /home/yeshen/workspace/toolchain/aarch64-none-linux-gnu/libc/usr/lib


/* 拷贝库文件,引擎文件到目标文件系统(参考pkgconfig中的.pc后缀文件) */
//sudo cp -r /home/syshaps/workspace/zye/openssl-target/usr/local/lib/libcrypto.* target_rootfs/usr/lib
//sudo cp -r /home/syshaps/workspace/zye/openssl-target/usr/local/lib/libssl.* target_rootfs/usr/lib
//cp -r /home/yeshen/workspace/openssl/target/usr/local/bin/* /home/yeshen/workspace/cpio_mnt/usr/bin
cp -r /home/yeshen/workspace/openssl/target/usr/local/lib/* /home/yeshen/workspace/cpio_mnt/usr/local/lib


/* 清除 */
make clean

/* 内核中的配置 */
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- menuconfig

//配置启动xxx硬件加速器  
Cryptographic API -->					//对应crypto/Kconfig
	[*] Hardware crypto devices -->		//对应drivers/crypto/Kconfig
		//对应drivers/crypto/xxx/Kconfig或者drivers/crypto/Kconfig直接子config
		<> Support for xxx hw accelerator
      
      
crypto/Kconfig    
    ....    
    source "drivers/crypto/Kconfig"    
    
drivers/crypto/Kconfig 
    menuconfig CRYPTO_HW
        ...
    if CRYPTO_HW
    source "drivers/crypto/xxx/Kconfig"
    config xxx
    ...
    endif # CRYPTO_HW
```



### 三. openssl测试用例

```c
/* 
 * 1.初始化 OpenSSL 库。
 * 2.获取您要使用的加密类型的 EVP_CIPHER 结构体实例（例如 EVP_aes_256_cbc()）。
 * 3.创建一个 EVP_CIPHER_CTX 上下文对象并将其初始化为加密模式。
 * 4.调用 EVP_EncryptInit_ex() 以将密钥和IV设置到上下文对象中，然后调用 EVP_EncryptUpdate()和
 *   EVP_EncryptFinal_ex() 来执行加密操作并获取输出。
 * 5.将输出传递给 AF_ALG 套接字接口。
 */
#include <stdio.h>
#include <stdlib.h>
#include <openssl/evp.h>
#include <sys/socket.h>
#include <linux/if_alg.h>
#include <string.h>
#include <unistd.h>

int main() {
    unsigned char *msg = "hello world";
    int msg_len = strlen(msg);

    /* 将您的加密密钥和 IV 存储在这里 */
    unsigned char key[EVP_MAX_KEY_LENGTH] = ...;
    unsigned char iv[EVP_MAX_IV_LENGTH] = ...;

    /* 根据需要设置算法、密钥长度和 IV 长度 */
    EVP_CIPHER *cipher = EVP_aes_256_cbc();
    int key_len = EVP_CIPHER_key_length(cipher);
    int iv_len = EVP_CIPHER_iv_length(cipher);
    unsigned char *encrypted = malloc(msg_len + EVP_MAX_BLOCK_LENGTH);
    unsigned char *decrypted = malloc(msg_len + EVP_MAX_BLOCK_LENGTH);

    if (encrypted == NULL || decrypted == NULL) {
        perror("malloc");
        exit(EXIT_FAILURE);
    }

    /* 初始化 OpenSSL 库 */
    OpenSSL_add_all_algorithms();

    /* 创建 AF_ALG 套接字 */
    int sockfd = socket(AF_ALG, SOCK_SEQPACKET, 0);
    if (sockfd == -1) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    /* 设置套接字类型为加密 */
    struct sockaddr_alg sa = {
        .salg_family = AF_ALG,
        .salg_type = "skcipher",
        .salg_name = "my_algorithm",
    };
    if (bind(sockfd, (struct sockaddr *)&sa, sizeof(sa)) == -1) {
        perror("bind");
        exit(EXIT_FAILURE);
    }

    /* 创建 OpenSSL 上下文对象并设置密钥和 IV */
    EVP_CIPHER_CTX *ctx = EVP_CIPHER_CTX_new();
    if (!ctx) {
        perror("EVP_CIPHER_CTX_new");
        exit(EXIT_FAILURE);
    }

    if (EVP_EncryptInit_ex(ctx, cipher, NULL, key, iv) != 1) {
        perror("EVP_EncryptInit_ex");
        exit(EXIT_FAILURE);
    }

    /* 执行加密操作 */
    int out_len;
    if (EVP_EncryptUpdate(ctx, encrypted, &out_len, msg, msg_len) != 1) {
        perror("EVP_EncryptUpdate");
        exit(EXIT_FAILURE);
    }
    int tmp_len;
    if (EVP_EncryptFinal_ex(ctx, encrypted + out_len, &tmp_len) != 1) {
        perror("EVP_EncryptFinal_ex");
        exit(EXIT_FAILURE);
    }
    out_len += tmp_len;

    /* 将加密数据写入 AF_ALG 套接字 */
    if (send(sockfd, encrypted, out_len, 0) != out_len) {
        perror("send");
        exit(EXIT_FAILURE);
    }

    /* 从 AF_ALG 套接字中读取解密数据 */
    int n = recv(sockfd, decrypted, msg_len, 0);
    if (n == -1) {
        perror("recv");
        exit(EXIT_FAILURE);
    }

    /* 打印解密结果 */
    printf("%s\n", decrypted);

    /* 释放资源并退出 */
    EVP_CIPHER_CTX_free(ctx);
    free(encrypted);
    free(decrypted);

    close(sockfd);

    return 0;
}

/* 编译 */
aarch64-none-linux-gnu-gcc xxx.c -o xxx -lcrypto 
```



### 四. openssl和spacc支持的密码和散列算法

&emsp;&emsp;openssl支持很多密码和散列算法，就算没有硬件支持openssl也可以使用它支持的算法处理数据。对于openssl不支持的某些算法如果硬件支持的话也可以使用openssl接口使用注册到内核crypto api框架的硬件算法，参考上面的测试用例。也就是说，如果硬件提供了用户需要的算法，那openssl即使不支持也可以通过统一的api使用，如果硬件不提供用户需要的算法而openssl支持的话，openssl也可以使用该算法处理用户的数据，只不过是用软件来处理，速度偏慢。spacc主要支持对称加密和散列，不支持非对称加密。下面列出openssl和spacc支持的密码和散列算法列表。

#### openssl支持的密码算法

```c
/* 该命令显示的是openssl支持的密码算法 */
$ openssl list-cipher-algorithms	

AES-128-CBC
AES-128-CBC-HMAC-SHA1
AES-128-CBC-HMAC-SHA256
AES-128-CFB
AES-128-CFB1
AES-128-CFB8
AES-128-CTR
AES-128-ECB
AES-128-OFB
AES-128-XTS
AES-192-CBC
AES-192-CFB
AES-192-CFB1
AES-192-CFB8
AES-192-CTR
AES-192-ECB
AES-192-OFB
AES-256-CBC
AES-256-CBC-HMAC-SHA1
AES-256-CBC-HMAC-SHA256
AES-256-CFB
AES-256-CFB1
AES-256-CFB8
AES-256-CTR
AES-256-ECB
AES-256-OFB
AES-256-XTS
AES128 => AES-128-CBC
AES192 => AES-192-CBC
AES256 => AES-256-CBC
BF => BF-CBC
BF-CBC
BF-CFB
BF-ECB
BF-OFB
CAMELLIA-128-CBC
CAMELLIA-128-CFB
CAMELLIA-128-CFB1
CAMELLIA-128-CFB8
CAMELLIA-128-ECB
CAMELLIA-128-OFB
CAMELLIA-192-CBC
CAMELLIA-192-CFB
CAMELLIA-192-CFB1
CAMELLIA-192-CFB8
CAMELLIA-192-ECB
CAMELLIA-192-OFB
CAMELLIA-256-CBC
CAMELLIA-256-CFB
CAMELLIA-256-CFB1
CAMELLIA-256-CFB8
CAMELLIA-256-ECB
CAMELLIA-256-OFB
CAMELLIA128 => CAMELLIA-128-CBC
CAMELLIA192 => CAMELLIA-192-CBC
CAMELLIA256 => CAMELLIA-256-CBC
CAST => CAST5-CBC
CAST-cbc => CAST5-CBC
CAST5-CBC
CAST5-CFB
CAST5-ECB
CAST5-OFB
DES => DES-CBC
DES-CBC
DES-CFB
DES-CFB1
DES-CFB8
DES-ECB
DES-EDE
DES-EDE-CBC
DES-EDE-CFB
DES-EDE-OFB
DES-EDE3
DES-EDE3-CBC
DES-EDE3-CFB
DES-EDE3-CFB1
DES-EDE3-CFB8
DES-EDE3-OFB
DES-OFB
DES3 => DES-EDE3-CBC
DESX => DESX-CBC
DESX-CBC
IDEA => IDEA-CBC
IDEA-CBC
IDEA-CFB
IDEA-ECB
IDEA-OFB
RC2 => RC2-CBC
RC2-40-CBC
RC2-64-CBC
RC2-CBC
RC2-CFB
RC2-ECB
RC2-OFB
RC4
RC4-40
RC4-HMAC-MD5
RC5 => RC5-CBC
RC5-CBC
RC5-CFB
RC5-ECB
RC5-OFB
SEED => SEED-CBC
SEED-CBC
SEED-CFB
SEED-ECB
SEED-OFB
AES-128-CBC
AES-128-CBC-HMAC-SHA1
AES-128-CBC-HMAC-SHA256
id-aes128-CCM
AES-128-CFB
AES-128-CFB1
AES-128-CFB8
AES-128-CTR
AES-128-ECB
id-aes128-GCM
AES-128-OFB
AES-128-XTS
AES-192-CBC
id-aes192-CCM
AES-192-CFB
AES-192-CFB1
AES-192-CFB8
AES-192-CTR
AES-192-ECB
id-aes192-GCM
AES-192-OFB
AES-256-CBC
AES-256-CBC-HMAC-SHA1
AES-256-CBC-HMAC-SHA256
id-aes256-CCM
AES-256-CFB
AES-256-CFB1
AES-256-CFB8
AES-256-CTR
AES-256-ECB
id-aes256-GCM
AES-256-OFB
AES-256-XTS
aes128 => AES-128-CBC
aes192 => AES-192-CBC
aes256 => AES-256-CBC
bf => BF-CBC
BF-CBC
BF-CFB
BF-ECB
BF-OFB
blowfish => BF-CBC
CAMELLIA-128-CBC
CAMELLIA-128-CFB
CAMELLIA-128-CFB1
CAMELLIA-128-CFB8
CAMELLIA-128-ECB
CAMELLIA-128-OFB
CAMELLIA-192-CBC
CAMELLIA-192-CFB
CAMELLIA-192-CFB1
CAMELLIA-192-CFB8
CAMELLIA-192-ECB
CAMELLIA-192-OFB
CAMELLIA-256-CBC
CAMELLIA-256-CFB
CAMELLIA-256-CFB1
CAMELLIA-256-CFB8
CAMELLIA-256-ECB
CAMELLIA-256-OFB
camellia128 => CAMELLIA-128-CBC
camellia192 => CAMELLIA-192-CBC
camellia256 => CAMELLIA-256-CBC
cast => CAST5-CBC
cast-cbc => CAST5-CBC
CAST5-CBC
CAST5-CFB
CAST5-ECB
CAST5-OFB
des => DES-CBC
DES-CBC
DES-CFB
DES-CFB1
DES-CFB8
DES-ECB
DES-EDE
DES-EDE-CBC
DES-EDE-CFB
DES-EDE-OFB
DES-EDE3
DES-EDE3-CBC
DES-EDE3-CFB
DES-EDE3-CFB1
DES-EDE3-CFB8
DES-EDE3-OFB
DES-OFB
des3 => DES-EDE3-CBC
desx => DESX-CBC
DESX-CBC
id-aes128-CCM
id-aes128-GCM
id-aes128-wrap
id-aes128-wrap-pad
id-aes192-CCM
id-aes192-GCM
id-aes192-wrap
id-aes192-wrap-pad
id-aes256-CCM
id-aes256-GCM
id-aes256-wrap
id-aes256-wrap-pad
id-smime-alg-CMS3DESwrap
idea => IDEA-CBC
IDEA-CBC
IDEA-CFB
IDEA-ECB
IDEA-OFB
rc2 => RC2-CBC
RC2-40-CBC
RC2-64-CBC
RC2-CBC
RC2-CFB
RC2-ECB
RC2-OFB
RC4
RC4-40
RC4-HMAC-MD5
rc5 => RC5-CBC
RC5-CBC
RC5-CFB
RC5-ECB
RC5-OFB
seed => SEED-CBC
SEED-CBC
SEED-CFB
SEED-ECB
SEED-OFB
```

#### openssl支持的散列算法

```c
/* 该命令显示的是openssl支持的散列算法 */
$ openssl list-message-digest-algorithms

DSA
DSA-SHA
DSA-SHA1 => DSA
DSA-SHA1-old => DSA-SHA1
DSS1 => DSA-SHA1
MD4
MD5
RIPEMD160
RSA-MD4 => MD4
RSA-MD5 => MD5
RSA-RIPEMD160 => RIPEMD160
RSA-SHA => SHA
RSA-SHA1 => SHA1
RSA-SHA1-2 => RSA-SHA1
RSA-SHA224 => SHA224
RSA-SHA256 => SHA256
RSA-SHA384 => SHA384
RSA-SHA512 => SHA512
SHA
SHA1
SHA224
SHA256
SHA384
SHA512
DSA
DSA-SHA
dsaWithSHA1 => DSA
dss1 => DSA-SHA1
ecdsa-with-SHA1
MD4
md4WithRSAEncryption => MD4
MD5
md5WithRSAEncryption => MD5
ripemd => RIPEMD160
RIPEMD160
ripemd160WithRSA => RIPEMD160
rmd160 => RIPEMD160
SHA
SHA1
sha1WithRSAEncryption => SHA1
SHA224
sha224WithRSAEncryption => SHA224
SHA256
sha256WithRSAEncryption => SHA256
SHA384
sha384WithRSAEncryption => SHA384
SHA512
sha512WithRSAEncryption => SHA512
shaWithRSAEncryption => SHA
ssl2-md5 => MD5
ssl3-md5 => MD5
ssl3-sha1 => SHA1
whirlpool
```

#### spacc支持的密码算法

```c
CRYPTO_MODE_CHACHA20_STREAM 	   CHACHA20 mode
CRYPTO_MODE_CHACHA20_POLY1305 	   CHACHA20/POLY1305 AEAD mode
CRYPTO_MODE_NULL 				  NULL cipher
CRYPTO_MODE_RC4_40 				  RC4 40-bit key
CRYPTO_MODE_RC4_128 			  RC4 128-bit key
CRYPTO_MODE_RC4_KS 				  RC4 and the RC4 state is manually loaded in the RC4 key 
								context (this allows non-standard key size support)
    
CRYPTO_MODE_AES_ECB 		      AES ECB mode
CRYPTO_MODE_AES_CBC 		      AES CBC mode
CRYPTO_MODE_AES_CTR 		      AES CTR mode
CRYPTO_MODE_AES_CCM 		      AES CCM mode
CRYPTO_MODE_AES_GCM 		      AES GCM mode (96-bit IV)
CRYPTO_MODE_AES_F8 			      AES F8 mode
CRYPTO_MODE_AES_XTS 		      AES XTS mode (with or without CTS)
CRYPTO_MODE_AES_CFB 		      AES CFB (s=128) mode
CRYPTO_MODE_AES_OFB 		      AES OFB (s=128) mode
CRYPTO_MODE_AES_CS1 		      NIST CBC-CS mode 1
CRYPTO_MODE_AES_CS2 		      NIST CBC-CS mode 2
CRYPTO_MODE_AES_CS3 		      NIST CBC-CS mode 3
CRYPTO_MODE_MULTI2_ECB 		      MULTI2 ECB mode
CRYPTO_MODE_MULTI2_CBC 		      MULTI2 CBC mode
CRYPTO_MODE_MULTI2_OFB 		      MULTI2 OFB mode
CRYPTO_MODE_MULTI2_CFB 		      MULTI2 CFB mode
CRYPTO_MODE_3DES_CBC 		      Triple DES CBC mode
CRYPTO_MODE_3DES_ECB 		      Triple DES ECB mode
CRYPTO_MODE_DES_CBC 		      DES CBC mode
CRYPTO_MODE_DES_ECB 		      DES ECB mode
CRYPTO_MODE_KASUMI_ECB 		      KASUMI ECB mode
CRYPTO_MODE_KASUMI_F8 		      KASUMI F8 mode
CRYPTO_MODE_SNOW3G_UEA2 	      SNOW3G UEA2 encryption mode
CRYPTO_MODE_ZUC_UEA3 		      ZUC UEA3 encryption mode
CRYPTO_MODE_SM4_ECB 		      SM4 ECB mode
CRYPTO_MODE_SM4_CBC 		      SM4 CBC mode
CRYPTO_MODE_SM4 CFB 		      SM4 CFB mode
CRYPTO_MODE_SM4_OFB 		      SM4 OFB mode
CRYPTO_MODE_SM4_CFB 		      SM4 CFB mode
CRYPTO_MODE_SM4_CTR 		      SM4 CTR mode
CRYPTO_MODE_SM4_CS1 		      SM4 CS1 mode
CRYPTO_MODE_SM4_CS2 		      SM4 CS2 mode
CRYPTO_MODE_SM4_CS3 		      SM4 CS3 mode
CRYPTO_MODE_SM4_CCM 		      SM4 CCM mode
CRYPTO_MODE_SM4_GCM 		      SM4 GCM mode
CRYPTO_MODE_SM4_F8 			      SM4 F8 mode
CRYPTO_MODE_SM4_XTS 		      SM4 XTS mode
```

#### spacc支持的散列算法

```c
CRYPTO_MODE_HASH_MD5            MD5 hash mode
CRYPTO_MODE_HMAC_MD5            MD5 HMAC mode
CRYPTO_MODE_HASH_SHA1           SHA-1 hash mode
CRYPTO_MODE_HMAC_SHA1           SHA-1 HMAC mode
CRYPTO_MODE_HASH_SHA224         SHA-224 hash mode (not to be confused with SHA-512_224)
CRYPTO_MODE_HMAC_SHA224         SHA-224 HMAC mode
CRYPTO_MODE_HASH_SHA256         SHA-256 hash mode
CRYPTO_MODE_HMAC_SHA256         SHA-256 HMAC mode
CRYPTO_MODE_HASH_SHA384         SHA-384 hash mode
CRYPTO_MODE_HMAC_SHA384         SHA-384 HMAC mode
CRYPTO_MODE_HASH_SHA512         SHA-512 hash mode
CRYPTO_MODE_HMAC_SHA512         SHA-512 HMAC mode
CRYPTO_MODE_HASH_SHA512_224     SHA-512_224 hash mode
CRYPTO_MODE_HMAC_SHA512_224     SHA-512_224 HMAC mode
CRYPTO_MODE_HASH_SHA512_256     SHA-512_256 hash mode
CRYPTO_MODE_HMAC_SHA512_256     SHA-512_256 HMAC mode
CRYPTO_MODE_HASH_SHA3_224       SHA3-224 hash mode
CRYPTO_MODE_HASH_SHA3_256       SHA3-224 hash mode
CRYPTO_MODE_HASH_SHA3_384       SHA3-224 hash mode
CRYPTO_MODE_HASH_SHA3_512       SHA3-224 hash mode
CRYPTO_MODE_HASH_SHAKE128       SHAKE128 hash mode
CRYPTO_MODE_HASH_SHAKE256       SHAKE256 hash mode
CRYPTO_MODE_HASH_CSHAKE128      CSHAKE128 hash mode
CRYPTO_MODE_HASH_CSHAKE256      CSHAKE256 hash mode
CRYPTO_MODE_MAC_POLY1305        POLY1305 MAC mode
CRYPTO_MODE_MAC_XCBC            AES XCBC MAC mode
CRYPTO_MODE_MAC_CMAC            AES CMAC MAC mode
CRYPTO_MODE_MAC_KASUMI_F9       KASUMI F9 MAC mode
CRYPTO_MODE_MAC_KMAC128         KMAC128 MAC mode
CRYPTO_MODE_MAC_KMAC256         KMAC256 MAC mode
CRYPTO_MODE_MAC_KMACXOF128      KMACXOF128 MAC mode
CRYPTO_MODE_MAC_KMACXOF256      KMACXOF256 MAC mode
CRYPTO_MODE_MAC_SNOW3G_UIA2     SNOW3G UIA2 MAC mode
CRYPTO_MODE_MAC_ZUC_UIA3        ZUC UIA3 MAC mode
CRYPTO_MODE_SSLMAC_MD5          SSL 3.0 MAC based on MD5
CRYPTO_MODE_SSLMAC_SHA1         SSL 3.0 MAC based on SHA1
CRYPTO_MODE_HASH_CRC32          CRC32 HASH (not cryptographically secure)
CRYPTO_MODE_MAC_MICHAEL         Michael-MIC Algorithm
CRYPTO_MODE_HASH_SM3            SM3 hash mode
CRYPTO_MODE_MAC_SM4_XCBC        SM4 XCBC MAC mode
CRYPTO_MODE_MAC_SM4_CMAC        SM4 CMAC MAC mode
```

