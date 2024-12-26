# rbcrypto user guide

### 安装rbcrypto

```c
# 使用rpm安装
# 一般不需要单独安装，这个包已经通过N430x_Basic_SDK一起安装
rpm -i rbcrypto-1.1.6-1.aarch64.rpm

# 查询安装包的详细信息
rpm -qi rbcrypto
Name        : rbcrypto
Version     : 1.1.6
Release     : 1
Architecture: aarch64
Install Date: Fri 16 Aug 2024 05:59:06 PM CST
Group       : Unspecified
Size        : 1314078
License     : BSD
Signature   : (none)
Source RPM  : rbcrypto-1.1.6-1.src.rpm
Build Date  : Tue 30 Jul 2024 03:38:55 PM CST
Build Host  : openEuler
URL         : rbcrypto-1.1.6.tar.gz
Summary     : N430x board crypto library
Description :
crypto library for caisa430 N430x board.
```



### 测试rbcrypto

```c
# 加载驱动模块
# 一般不需要单独加载，除非是第一次安装rbcrypto，否则会在开机默认加载驱动
[root@openEuler ~]# /opt/rb/etc/loadko.sh

# 跑一个rbcrypto包的测试用例
[root@openEuler ~]# /opt/rb/tests/aes_128_cbc_test
Original: This is a test for AES-128-CBC encryption
Encrypted: 0a51438d5ca3a097b4a4c8be5a6eb276a7e437eee03d81c5350f539d7917c62d298a784b68e14e9ad2
```



### 编译使用rbcrypto的程序

​		这个给出一个在openeuler板子本地编译使用rbcrypto库的demo。

```c
# 板子首先提前安装rbcrypto包

# 然后在板子上创建程序目录rbcrypto_demo_dir
mkdir rbcrypto_demo_dir
cd rbcrypto_demo_dir

# 在rbcrypto_demo_dir下创建编辑rbcrypto_demo.cpp文件和CMakeLists.txt文件
# demo.cpp可以参考rbcrypto源码包的arm/tests，比如aes_128_cbc_test.cpp
# CMakeLists.txt如下
cmake_minimum_required(VERSION 3.5)
project(rbcrypto-demo)
set (CMAKE_CXX_STANDARD 11)
include_directories(/opt/rb/include/rbcrypto/)
link_directories(/opt/rb/lib/)
add_executable (rbcrypto_demo ${CMAKE_CURRENT_LIST_DIR}/rbcrypto_demo.cpp)
target_link_libraries (rbcrypto_demo PUBLIC rbcrypto)
    
# 在rbcrypto_demo_dir下创建build目录，然后到build中编译生成rbcrypto_demo
mkdir build
cd build
cmake ..
make

# 测试rbcrypto_demo
./rbcrypto_demo
Original: This is a test for AES-128-CBC encryption
Encrypted: 0a51438d5ca3a097b4a4c8be5a6eb276a7e437eee03d81c5350f539d7917c62d298a784b68e14e9ad2
Decrypted: This is a test for AES-128-CBC encryption
```



### rbcrypto源码包组成

```c
# 下载源码包
git clone git@crdev1.corerain.com:bsp/rbcrypto.git

# 源码包组成
[zye@cr2 rbcrypto]$ tree
.
├── arm
│   ├── CMakeLists.txt
│   ├── include
│   │   └── rbcrypto_api.h
│   ├── src
│   │   └── rbcrypto_api.c
│   └── tests
│       ├── aes_128_cbc_decrypt_file.cpp
│       ├── aes_128_cbc_efuse_test.cpp
│       ├── aes_128_cbc_encrypt_file.cpp
│       ├── aes_128_cbc_test.cpp
│       ├── aes_192_cbc_efuse_test.cpp
│       ├── aes_192_cbc_test.cpp
│       ├── aes_256_cbc_efuse_test.cpp
│       ├── aes_256_cbc_test.cpp
│       ├── aes_process_test.cpp
│       ├── aes_thread_test.cpp
│       ├── CMakeLists.txt
│       ├── sm4_128_cbc_decrypt_file.cpp
│       ├── sm4_128_cbc_encrypt_file.cpp
│       └── sm4_128_cbc_test.cpp
├── README.md
├── scripts
│   ├── loadko.sh
│   ├── pack_deb.sh
│   ├── pack_rpm.sh
│   ├── rbcrypto.spec
│   └── unloadko.sh
└── x86
    └── tests
        ├── aes_128_cbc_decrypt_file.cpp
        ├── aes_128_cbc_encrypt_file.cpp
        ├── aes_128_cbc_test.cpp
        ├── aes_192_cbc_test.cpp
        ├── aes_256_cbc_test.cpp
        ├── sm4_128_cbc_decrypt_file.cpp
        ├── sm4_128_cbc_encrypt_file.cpp
        └── sm4_128_cbc_test.cpp
```

​		rbcrypto源码包的根目录有arm目录、README.md文件、scripts目录、x86目录。各部分描述如下：

- arm目录：包含rbcrypto api实现的源文件、头文件，使用rbcrypto api的测试用例源文件，以及编译rbcrypto库和这些测试用例的CMakeLists.txt。
- README.md文件：说明文档。
- scripts目录：在编译了rbcrypto包后，通过这个目录的脚本文件打包rbcrypto rpm包或者deb包。
- x86目录：使用openssl开源库api的测试用例，严格来说不属于rbcrypto，给用户在x86上参考使用。



### rbcrypto api描述

​		通过将linux套接字api封装成对用户易用的rbcryptoapi，应用程序只需一个api接口即可完成加解密任务。api格式如下：

```c
/* api原型 */
int rbcrypto_operate(struct crypto_info *info);

/* 功能 */
使用硬件spacc对数据加密或解密

/* 参数 */
包含所有必要的加解密信息的struct crypto_info结构体的指针

/* 返回值 */
成功返回0，失败返回非0
```

​		其中最关键的是struct crypto_info结构体，该结构体和各成员详细说明如下：

```c
struct crypto_info {
	int mode;
	uint8_t	*src;
	size_t srclen;
	uint8_t *dst;
	uint8_t *key;
	size_t keylen;
	uint8_t *iv;
	size_t ivlen;
	int isencrypt;
	int isefuse;
	int efusegroup;	
	int iszerocopy;
};

/* mode */
（必填项）算法模式，参考rbcrypto_api.h中的宏定义，比如CRYPTO_MODE_AES_CBC表示aes cbc模式。

/* src */
（必填项）待加密或解密的源数据地址。

/* srclen */
（必填项）待加密或解密的数据大小。如果srclen是block（算法的块大小，比如aes和sm4是16字节）对齐的，那么src和dst指向的内存大小要大于或等于srclen; 如果srclen不是block对齐的，那么src和dst指向的内存大小要大于或等于srclen向上对齐block后的长度；对于解密，srclen必须是block对齐的。

/* dst */
（必填项）加密或解密后存放结果的地址。

/* key */
（非必填项）密钥的地址。对于非efuse加解密是比填项；对于efuse加解密可以忽略。

/* keylen */
（必填项）密钥的长度。对于aes算法有128bit,192bit,256bit 3种，对于sm4只有128bit。但这里是按字节来填的，128bit填16，192bit填24，256bit填32。

/* iv */
（必填项）初始化向量块地址。

/* ivlen */
（必填项）初始化向量块大小，等于算法块大小。

/* isencrypt */
（必填项）加密填1，解密填0

/* isefuse */
（必填项）使用efuse加解密设置成1，不使用efuse加解密设置成0

/* efusegroup */
（非必填项）表示efuse中的第几组key。不使用efuse加解密时可以忽略；使用efuse加解密时必填，其中128bit有6组，即可以填1，2，3，4，5，6，192bit和256bit都只有3组，可以填1，2，3，192bit的每一组是256bit每一组的前192bit。

/* iszerocopy */
（必填项）表示使用内核的零拷贝特性。目前暂不开放，统一设置成0
```

**使用加密api注意：**如果待加密的数据长度没有按算法块(对于aes和sm4是16字节)对齐，用户可以自己按照PKCS7规则事先填充，或者让rbcrypto api填充，如果用户自行填充则传给rbcrypto api的长度是填充后的长度，如果用户不填充则传给rbcrypto api的长度是填充前的长度；无论是自行填充还是让rbcrypto api填充，传给rbcrypto api的src和dst指针指向的空间必须满足大于等于填充后的长度的空间大小。

**使用解密api注意：**如果待解密的数据长度没有按算法块(对于aes和sm4是16字节)对齐，用户不要自行填充，但是src中指向的待加密的数据一定是包含原来加密未对齐数据得到的多余部分；无论是否对齐，传给rbcrypto api的src和dst指针指向的空间必须满足大于等于对齐后的长度的空间大小。



### x86平台加解密接口使用范例

​		x86平台通过openssl开源库api使用软件加解密，而不是使用rbcrypto api。openssl加密库的EVP api是最为流行和常用的加解密api。下面是一个使用EVP api作aes-cbc加解密的完整例子，详细说明请参考openssl官网或者网上教程。

```c
#include <openssl/evp.h>  
#include <openssl/rand.h>  
#include <string.h>  
#include <stdio.h>  

#define AES_BLOCK_SIZE 16
  
void handleErrors(void) {  
    fprintf(stderr, "An error occurred\n");  
    exit(EXIT_FAILURE);  
}  
  
int main(void) {  
    // 待加密的数据  
    unsigned char aes_input[] = "This is a test for AES-128-CBC encryption";  
    size_t aes_inputlen = strlen((char *)aes_input);  
  
    // 密钥和初始化向量（IV）  
    unsigned char aes_key[16]; // AES-128 需要 16 字节的密钥  
    unsigned char aes_iv[AES_BLOCK_SIZE]; // IV 大小为 AES 块大小  
  
    // 加密后的数据  
    unsigned char enc_out[aes_inputlen + AES_BLOCK_SIZE];  
    int enc_outlen;  
  
    // 解密后的数据  
    unsigned char dec_out[aes_inputlen + AES_BLOCK_SIZE];  
    int dec_outlen;  
  
    // 初始化密钥和IV  
    if (!RAND_bytes(aes_key, sizeof(aes_key)))  
        handleErrors();  
    if (!RAND_bytes(aes_iv, sizeof(aes_iv)))  
        handleErrors();  
  
    // 初始化加密操作  
    EVP_CIPHER_CTX *ctx_enc = EVP_CIPHER_CTX_new();  
    EVP_EncryptInit_ex(ctx_enc, EVP_aes_128_cbc(), NULL, aes_key, aes_iv);  
  
    // 执行加密  
	int enc_outlen_tmp = 0;
    if (1 != EVP_EncryptUpdate(ctx_enc, enc_out, &enc_outlen, aes_input, aes_inputlen))  
        handleErrors();  
    if (1 != EVP_EncryptFinal_ex(ctx_enc, enc_out + enc_outlen, &enc_outlen_tmp))  
        handleErrors();  
    enc_outlen += enc_outlen_tmp;  
  
    // 清理加密上下文  
    EVP_CIPHER_CTX_free(ctx_enc);  
  
    // 初始化解密操作  
    EVP_CIPHER_CTX *ctx_dec = EVP_CIPHER_CTX_new();  
    EVP_DecryptInit_ex(ctx_dec, EVP_aes_128_cbc(), NULL, aes_key, aes_iv);  
  
    // 执行解密  
	int dec_outlen_tmp = 0;
    if (1 != EVP_DecryptUpdate(ctx_dec, dec_out, &dec_outlen, enc_out, enc_outlen))  
        handleErrors();  
    if (1 != EVP_DecryptFinal_ex(ctx_dec, dec_out + dec_outlen, &dec_outlen_tmp))  
        handleErrors();  
    dec_outlen += dec_outlen_tmp;  
  
    // 清理解密上下文  
    EVP_CIPHER_CTX_free(ctx_dec);  
  
    // 输出结果  
    printf("Original: %s\n", aes_input);  
    printf("Encrypted: ");  
    for (int i = 0; i < enc_outlen; i++)  
        printf("%02x", enc_out[i]);  
    printf("\n");  
    printf("Decrypted: %s\n", dec_out);  
  
    return 0;  
}
```

在编译上面代码时需要链接openssl加密库：

```c
gcc evp_aes_example.c -o evp_aes_example -lcrypto
```

运行结果：

```c
yeshen@ubuntu:~/workspace/app-demo/host$ ./evp_aes_example 
Original: This is a test for AES-128-CBC encryption
Encrypted: 7db87f3053d57f8a329f10e3d5697651efe181247e44eab772c3c4d9f21dc76db5c997d7b8026df199ea492fcd456034
Decrypted: This is a test for AES-128-CBC encryption
```

evp api加解密步骤：

```c
1.准备好待加密/解密的数据，初始化好密钥和初始化向量iv。
2.使用EVP_CIPHER_CTX_new和EVP_EncryptInit_ex初始化加密/解密操作。
3.使用EVP_EncryptUpdate和EVP_DecryptFinal_ex执行加密/解密. 
4.使用EVP_CIPHER_CTX_free清理加密/解密上下文。
```

