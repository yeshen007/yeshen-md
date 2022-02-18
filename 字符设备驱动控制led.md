# <center>字符设备驱动控制led</center>

## 一、驱动代码

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <linux/device.h>
#include <asm/uaccess.h>
#include <asm/io.h>
#include <linux/slab.h>
#include <linux/err.h>



/* 设计一个类型，描述一个led设备的信息 */
struct led_desc {
	unsigned int dev_major;		//设备号
	struct class *cls;
	struct device *dev;		//创建设备文件
	void *reg_virt_base;	//寄存器基地址虚拟
};

/* 主设备号 */
#define LED_MAJOR 230

/* led对应的GPIO物理基地址 */
#define GPIO1_BASE_ADDR		0xff709000
/* 需要映射的地址范围 */
#define GPIO1_SIZE	8

/* 表示一个全局的设备对象 */
struct led_desc *led_dev; 


/* 上传给用户read的值 */
static int kernel_val = 555;

static int led_open(struct inode *inode, struct file *filp)
{
	printk("-----%s-----\n", __FUNCTION__);
	
	return 0;
}

static ssize_t led_read(struct file *filp, char __user *buf, 
									size_t size, loff_t *ppos)
{
	printk("-----%s-----\n", __FUNCTION__);
	int ret;
	/* copy_to_user(void __user *to, const void *from, unsigned long n); */
	ret = copy_to_user(buf, &kernel_val, size); //将kernel中的值上传给用户
	if (ret > 0) {
		printk("copy_to_user error\n");
		return -EFAULT;
	}
	
	return 0;
}

static ssize_t led_write(struct file *filp, const char __user *buf,
								size_t size, loff_t *ppos)
{
	printk("-----%s-----\n", __FUNCTION__);
	int ret;
	int value;
	/* copy_from_user(void *to, const void __user *from, unsigned long n); */
	ret = copy_from_user(&value, buf, size);	//将用户传过来的值写到value

	if (value)     /* 如果value非零，亮灯 */
		writel(readl(led_dev->reg_virt_base) | (0x1<<24), led_dev->reg_virt_base);
	else  		/* 如果value为零，灭灯 */
		writel(readl(led_dev->reg_virt_base) & (~(0x1<<24)), led_dev->reg_virt_base);
	

	printk("__KERN__ : value = %d\n", value);	
	return 0;
}

								
static int led_close(struct inode *inode, struct file *filp)
{
	printk("-----%s-----\n", __FUNCTION__);
	
	return 0;
}


static const struct file_operations led_fops = {
	.open = led_open,
	.read = led_read,
	.write = led_write,
	.release = led_close,
};


static int __init led_init(void)
{
	/* 实例化全局设备对象--分配空间 */
	led_dev = kmalloc(sizeof(struct led_desc), GFP_KERNEL);
	if (led_dev == NULL) {
		printk(KERN_ERR "malloc error\n");
		return -ENOMEM;
	}

	/* 注册设备 */
	int ret;
	led_dev->dev_major = register_chrdev(0, "yeshen_led", &led_fops);  //0代表动态申请设备号
	if (led_dev->dev_major < 0) {
		printk(KERN_ERR "register failed\n");
		ret = -ENODEV;
		goto err_0;
	} 
	

	/* 自动创建设备文件节点 */
	led_dev->cls = class_create(THIS_MODULE, "led_cls");
	if (IS_ERR(led_dev->cls)) {
		printk(KERN_ERR "class create failed\n");
		ret = PTR_ERR(led_dev->cls);
		goto err_1;		
	}
	
	led_dev->dev = device_create(led_dev->cls, NULL, MKDEV(led_dev->dev_major, 0), NULL, "led_dev");		//对应/dev下设备文件的名字
	if (IS_ERR(led_dev->dev)) {
		printk(KERN_ERR "device create failed\n");
		ret = PTR_ERR(led_dev->dev);
		goto err_2;		
	}

	/* 映射物理地址到虚拟地址。返回虚拟地址。参数1：物理地址，参数2：长度 */
	led_dev->reg_virt_base = ioremap(GPIO1_BASE_ADDR, GPIO1_SIZE);
	if (IS_ERR(led_dev->reg_virt_base)) {
		printk(KERN_ERR "ioremap failed\n");	
		ret = -ENOMEM;
		goto err_3;
	}

	/* 初始化gpio53为输出，高电平 */
	u32 value = readl(led_dev->reg_virt_base + 4);
	value |= (0x1<<24);
	writel(value, led_dev->reg_virt_base + 4);

	value = readl(led_dev->reg_virt_base);
	value |= (0x1<<24);
	writel(value, led_dev->reg_virt_base);
	

	
	return 0;

err_3:
	device_destroy(led_dev->cls, MKDEV(led_dev->dev_major, 0));
err_2:	
	class_destroy(led_dev->cls);
err_1:
	unregister_chrdev(led_dev->dev_major, "yeshen_led");
err_0:
	kfree(led_dev);
	return ret;

}


static void __exit led_exit(void)
{
	/* 解除地址映射 */
	iounmap(led_dev->reg_virt_base);
	
	/* 释放自动创建的设备节点 */
	device_destroy(led_dev->cls, MKDEV(led_dev->dev_major, 0));
	class_destroy(led_dev->cls);

	/* 释放设备 */
	unregister_chrdev(led_dev->dev_major, "yeshen_led");

	/* 释放实例化的全局设备分配的内存 */
	kfree(led_dev);

}




module_init(led_init);
module_exit(led_exit);
MODULE_LICENSE("GPL");


```



# 二、应用测试代码

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>



int main(int argc, char *argv[])
{

	int fd;
	int value;

    /* 打开设备文件 */
	fd = open("/dev/led_dev", O_RDWR);
	if (fd < 0){
		perror("open");
		exit(1);
	}

    /* 应该读到555 */
	read(fd, &value, 4);

	printf("__USER__ value = %d\n", value);

	value = 666;
		
    /* 写入非0值应该亮灯 */
	write(fd, &value, 4);

	while (1) {
		
		printf("led off\n");
		sleep(1);
		value = 0;
		write(fd, &value, 4);

		printf("led on\n");
		sleep(1);
		value =1;
		write(fd, &value, 4);
		
	}

	close(fd);

	return 0;

}

```



# 三、测试结果

![](pictures\led.PNG)

​		从上图测试结果可以看出：驱动正常运行。

