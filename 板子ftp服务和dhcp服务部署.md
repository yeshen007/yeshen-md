## <center>板子ftp服务和dhcp服务部署</center>

[TOC]

### 一、ftp服务tftpd和vsftpd

#### 1. tftpd

##### 准备工作

&emsp;&emsp;首先在buildroot目录下做好清理配置后make menuconfig，然后选中`Target packages`下的`Show packages that are also provided by busybox`，后文的vsftpd和dhcpd也是这样，因为这些在busybox配置里，buildroot默认使用一个busybox配置文件来生成这些工具但是没有在菜单里显示，通过以上步骤，busybox中的也会在buildroot中显示。

##### buildroot配置编译

```c
Target packages --->
	Networking applications --->
    	tftpd (NEW)
```

##### tftp命令

```c
/* tftp下载远端服务器的tftp文件夹的文件指令 */
tftp -g -r <filename> <serverip>
/* tftp上传文件到服务器的tftp文件夹 */
tftp -p -l <filename> <serverip>

/* 从远端服务器下载文件到本地的内存(uboot命令) */
tftp <addr> <filename>

/*
 * 新版tftp用法
*/
tftp <serverip>
tftp> get <file>
tftp> put <file>
tftp> q
```

##### tftpd服务启动

```c
/* step1 */
通过 ps | grep tftpd 得到如下：
143 root     /usr/sbin/tftpd -c -l -s /var/lib/tftpboot
/* step2 */
在/etc目录下grep "/var/lib/tftpboot" * -nr 得到：
init.d/S80tftpd-hpa:3:OPTIONS="-c -l -s /var/lib/tftpboot"
init.d/S80tftpd-hpa:18: mkdir -p /var/lib/tftpboot
init.d/S80tftpd-hpa:19: chmod 1777 /var/lib/tftpboot
/* step3 */
打开S80tftpd-hpa文件修改/var/lib/tftpboot为自己想要的目录<mydir>
/* step4 */
重启开发板
/* step5 */
在板子的<mydir>下创建文件<myfile>
/* step6 */
再主机连接板子的tftpd服务器测试
tftp <boardip>
tftp> get <file>
tftp> put <file>
tftp> q
```



#### 2. vsftpd

##### buildroot配置编译

```c
Target packages --->
	Networking applications --->
    	vsftpd 
```

##### 服务配置文件vsftpd.conf

```c
#是否允许匿名访问
anonymous_enable=NO
#允许本地用户访问(/etc/passwd中的用户)
local_enable=YES
#允许写入权限，包括修改，删除
write_enable=YES
#本地用户文件上传后的权限
local_umask=022
#设定所有本地用户登陆后的目录。
#如不设此项，则本地用户登陆后位于各自家目录下（如/home/yaho）
local_root=/mnt
#匿名用户上传后权限
anon_umask=022
#允许匿名用户上传
anon_upload_enable=YES
#允许匿名用户建立目录
anon_mkdir_write_enable=YES
#允许匿名用户具有建立目录，上传之外的权限，如重命名，删除
anon_other_write_enable=YES
#允许匿名用户浏览，下载文件
anon_world_readable_only=YES
#匿名用户登录是不需要密码，YES不需要密码；NO需要密码
no_anon_password=NO
#设定匿名用户登陆后所在的目录
anon_root=/mnt
#当使用者转换目录,则会显示该目录下的.message信息
dirmessage_enable=YES
#确保ftp-datad 数据传送使用port 20
connect_from_port_20=YES
#PAM所使用的名称.同userlist_*一样限制用户登陆，
#不同的是userlist_*在进行密码验证之前拒绝用户登陆，
#pam是在密码验证之后拒绝登陆.(提示密码错误) 
#用户列表默认存放在/etc/ftpusers中，一行一个. 
#(可通过/etc/pam.d/vsftpd重定向用户列表存放文件)
pam_service_name=vsftpd
#定义匿名登入的使用者名称。默认值为ftp。
ftp_username=ftp
#支持tcp_wrappers,限制访问(/etc/hosts.allow,/etc/hosts.deny)，
#NO是可以访问
tcp_wrappers=NO
#ftp监听端口，注意是21
listen_port=21
    
#是否启动userlist为通过模式，YES的话只有存在于userlist文件中的用户才能登录ftp（可以理解为userlist是一个白名单），NO的话，白名单失效，和下面一个参数配合使用
userlist_enable=YES
#是否启动userlist为禁止模式，YES表示在userlist中的用户禁止登录ftp（黑名单），NO表示黑名单失效，我们已经让userlist作为一个白名单，所以无需使用黑名单功能
userlist_deny=NO
#指定哪个文件作为userlist文件，我们稍后编辑这个文件
userlist_file=/etc/vsftpd.user_list

#是否限制本地所有用户切换根目录的权限，YES为开启限制，即登录后的用户不能访问ftp根目录以外的目录，当然要限制啦
chroot_local_user=YES
#是否启动限制用户的名单list为允许模式，上面的YES限制了所有用户，可以用这个名单作为白名单，作为例外允许访问ftp根目录以外
chroot_list_enable=YES
#设置哪个文件是list文件，里面的用户将不受限制的去访问ftp根目录以外的目录
chroot_list_file=/etc/vsftpd.chroot_list
#是否开启写模式，开启后可以进行创建文件夹等写入操作
allow_writeable_chroot=YES
```

##### 登录操作

```c
/* 首先定义可访问的用户和可访问ftp目录以外的用户 */
cat vsftpd.user_list
	root
	yezheng
	ftp
cat vsftpd.chroot_list
	root
	yezheng
	
/* 板子启动后，在linux主机模拟ftp客户端：ftp <boardip>
 * 然后输入板子文件chroot_list中其中一个账号和密码，
 * 看到的根目录和板子上的一样，可以cd切换目录访问别的文件。
 */
hanglory@HanGlory:~/yezheng/git/bak/ftptest$ ftp 192.168.34.129
Connected to 192.168.34.129.
220 (vsFTPd 3.0.3)
Name (192.168.34.129:hanglory): root
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 0        0              17 Jan 01  2000 ye.txt
226 Directory send OK.
ftp> pwd
257 "/mnt" is the current directory
ftp> cd ..
250 Directory successfully changed.
ftp> cd mnt
250 Directory successfully changed.
ftp> get ye.txt
local: ye.txt remote: ye.txt
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for ye.txt (17 bytes).
226 Transfer complete.
17 bytes received in 0.00 secs (12.6440 kB/s)
ftp> quit
221 Goodbye.
hanglory@HanGlory:~/yezheng/git/bak/ftptest$ ls
ye.txt
hanglory@HanGlory:~/yezheng/git/bak/ftptest$ cat ye.txt
yezheng ftp test
hanglory@HanGlory:~/yezheng/git/bak/ftptest$

/* 
 * 使用不在文件chroot_list却在user_list中的ftp登录，
 * 看到的根目录是板子的/mnt(local_root),并且无法切换到外面
 * 访问板子/mnt外面的文件
 */
hanglory@HanGlory:~/yezheng/git/bak/ftptest$ ftp 192.168.34.129
Connected to 192.168.34.129.
220 (vsFTPd 3.0.3)
Name (192.168.34.129:hanglory): ftp
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> pwd
257 "/" is the current directory
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 0        0              17 Jan 01  2000 ye.txt
226 Directory send OK.
ftp> cd ..
250 Directory successfully changed.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 0        0              17 Jan 01  2000 ye.txt
226 Directory send OK.
ftp>
```



### 二、dhcp服务dhcpd

##### buildroot配置编译

```c
Target packages --->
	Networking applications --->
    	dhcp (ISC) 
```

&emsp;&emsp;上面可以配置dhcpd和dhclient，dhcpd是dhcp服务器程序，dhclient是dhcp客户端程序，还有另外一个dhcp客户端程序是dhcpcd，配置路径和dhcp (ISC)同级目录，busybox中**还有一个udhcpc客户端和udhcpd服务器，我们板子启动时默认使用udhcpc来获取ip，但从现象来看它不是获取dhcpd中设置的ip，后续继续研究**。dhcpd和dhclient的配置文件在/etc/dhcp中的dhcpd.conf和dhclient.conf，dhcpcd的配置文件在/etc中的dhcpcd.conf。

##### dhcp服务配置

```
# /etc/network/interfaces 
auto eth0
iface eth0 inet dhcp

# /etc/dhcp/dhcpd.conf
subnet 192.168.0.0 netmask 255.255.0.0 {
range 192.168.34.49 192.168.34.53;
option routers 192.168.3.254;
option subnet-mask 255.255.0.0;
option broadcast-address 192.168.255.255;
option domain-name-servers 202.96.128.166, 114.114.114.114;
}

```

##### dhcp客户端命令

```c
dhclient -r
dhclient ethx 或者 dhclient
dhclient -s <dhcpserverip>
```



