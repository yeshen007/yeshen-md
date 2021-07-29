# </center>newlib库移植<center>

###  编译newlib前需要的工具：

​		编译newlib需要autoconf、automake、textinfo、libtool和gettext，从ftp://ftp.gnu.org下载这些工具的源代码然后再编译安装到主机ubuntu中，这些工具的编译安装方法都一样，下面给出autoconf的编译安装步骤：

1. 从ftp://ftp.gnu.org下载autoconf.tar.gz
2. 在当前目录/home/hn/WJ_TEST解压autoconf.tar.gz  ： tar xvf autoconf.tar.gz
3. 在当前目录创建编译目录和安装目录 : mkdir autoconf-build && mkdir autoconf-install 
4. 配置： cd autoconf-build && ../autoconf/configure --prefix=/home/hn/WJ_TEST/autoconf-install
5. 编译和安装：make -j4 && make install
6. 路径导出到环境变量：export PATH=/home/hn/WJ_TEST/autoconf-install/bin/:$PATH 或者修改/etc/profile

注释：这里所使用的工具版本分别是：autoconf-2.64、automake-1.11.2、gettext-0.14.5、libtool-2.2.6b、textinfo-4.13。



###  编译安装newlib：

使用newlib-1.20.0版本编译。

1. 从ftp://ftp.gnu.org下载newlib-1.20.0.tar.gz
2. 在当前目录/home/hn/WJ_TEST解压newlib-1.20.0.tar.gz  ： tar xvf newlib-1.20.0.tar.gz
3. 在当前目录创建编译目录和安装目录 : mkdir newlib-build && mkdir newlib-install
4. 配置：cd newlib-build && ../newlib-1.20.0/configure --with-newlib --target=arm-none-eabi --prefix=/home/hn/WJ_TEST/newlib-install
5. 编译和安装：make -j4 && make install
6. 替换： 将/home/hn/WJ_TEST/newlib-install/arm-none-eabi/libz中的libc.a替换掉C:\Program Files (x86)\CodeSourcery\Sourcery_CodeBench_Lite_for_ARM_EABI\arm-none-eabi\lib中原来的libc.a
7. 然后从新编译工程烧写到板子重新运行，这时就是使用新库了。
8. 如果想修改库的源代码后编译，那么先在编译目录中make clean，然后删除安装目录中的内容，然后修改源代码再保存，最后编译和安装。

