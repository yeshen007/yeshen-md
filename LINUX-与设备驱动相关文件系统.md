# </center>LINUX 与设备驱动相关文件系统<center>

虚拟文件系统用一个inode关联一个文件

```CQL
struct inode {
	umode_t			i_mode;

	...

	dev_t			i_rdev;

	...
	
	const struct file_operations	*i_fop;	/* former ->i_op->default_file_ops */

	...
	
	struct list_head	i_devices;
	
	union {
		struct pipe_inode_info	*i_pipe;
		struct block_device	*i_bdev;
		struct cdev		*i_cdev;
	};

	...
};

```



```c
struct cdev {
	struct kobject kobj;
	struct module *owner;
	const struct file_operations *ops;
	struct list_head list;
	dev_t dev;
	unsigned int count;
};
```



```c
const struct file_operations def_chr_fops = {
	.open = chrdev_open,
	.llseek = noop_llseek,
};
```

```c
void init_special_inode(struct inode *inode, umode_t mode, dev_t rdev)
{
	inode->i_mode = mode;
	if (S_ISCHR(mode)) {
		inode->i_fop = &def_chr_fops;
		inode->i_rdev = rdev;
	} else if (S_ISBLK(mode)) {
		inode->i_fop = &def_blk_fops;
		inode->i_rdev = rdev;
	} else if (S_ISFIFO(mode))
		inode->i_fop = &def_fifo_fops;
	else if (S_ISSOCK(mode))
		inode->i_fop = &bad_sock_fops;
	else
		printk(KERN_DEBUG "init_special_inode: bogus i_mode (%o) for"
				  " inode %s:%lu\n", mode, inode->i_sb->s_id,
				  inode->i_ino);
}
```

```c
static void init_inode(struct inode *inode, struct treepath *path)
{
	...
    	pathrelse(path);
	if (S_ISREG(inode->i_mode)) {
		inode->i_op = &reiserfs_file_inode_operations;
		inode->i_fop = &reiserfs_file_operations;
		inode->i_mapping->a_ops = &reiserfs_address_space_operations;
	} else if (S_ISDIR(inode->i_mode)) {
		inode->i_op = &reiserfs_dir_inode_operations;
		inode->i_fop = &reiserfs_dir_operations;
	} else if (S_ISLNK(inode->i_mode)) {
		inode->i_op = &reiserfs_symlink_inode_operations;
		inode->i_mapping->a_ops = &reiserfs_address_space_operations;
	} else {
		inode->i_blocks = 0;
		inode->i_op = &reiserfs_special_inode_operations;
		init_special_inode(inode, inode->i_mode, new_decode_dev(rdev));
	}
}    
```





![](pictures\fs.PNG)



![](pictures\fs1.PNG)