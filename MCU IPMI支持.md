# 460 MCU IPMI支持

## ipmi架构

![](D:\bsp\docs\yeshen-md\pictures\ipmi-ipmb.PNG)

​		如上台所示是ipmi架构图，其中核心是服务器上的BMC(baseboard management controller)。然后通过多种接口和总线连接多种设备。其中远程host通过ICMB，LAN等连接方式使用ipmitool，openipmi等软件和BMC通信；本地host(服务器上的cpu)通过System Interface(KCS,SMIC,BT,SSIF等)接口使用ipmi软件和BMC通信；BMC还通过IPMB总线和其他总线或接口连接其他设备。所有这些部件和连接方式都属于IPMI，无论在哪种物理接口传输数据，都是使用ipmi规范的request/response形式的ipmi message格式，只是对于不同物理接口的ipmi message格式可能有调整。



## 460 host和mcu通信

### host和mcu通信流程概述

​		对于460来说重点要了解的是IPMB及其message，当然也要了解host和BMC的ipmi message格式。因为服务器监控或者设置请求是从host发起的，host先发送一条包含ipmb message的ipmi message给BMC，然后BMC再从这条ipmi message中提取ipmb message转发到IPMB上，IPMB上的SMC(Satellite Management Controller)即460 mcu收到后解析这条message，然后作相应的处理比如读取温度，然后将结果响应通过ipmb message发送给BMC的Receive Message Queue，然后host通过命令从BMC的Receive Message Queue读出来。
​		以上是host和mcu通信的大致流程，下面详细讲host和mcu通信的消息协议格式。host和mcu通信基于ipmb接口的消息格式如下。
![](D:\bsp\docs\yeshen-md\pictures\ipmb.PNG)

​													                                          图1

### host 发消息给mcu

​		host使用BMC作为IPMB控制器，通过host和BMC的接口向IPMB发送消息，这是通过使用Send Message命令将消息写入IPMB（通道0）来完成的。host软件负责提供IPMB消息的所有字段，包括请求者和响应者的从地址和校验和。下图显示了使用Send Message命令通过host和BMC接口向从地址52h LUN 00b发送Set Event Receiver命令的示例。示例命令将事件接收器地址设置为20h = BMC(20h就是BMC的地址)。

![](D:\bsp\docs\yeshen-md\pictures\host2mcu.PNG)

​																						图2

粗体字段是Send Message命令中携带的IPMB消息的字节，对应图1中的Request。rsSA字段占一个字节，52h表示消息的接收者为52h地址的设备；NetFN/rsLUN字段占一个字节，其中最高6位NetFN是04h，表示Sensor/Event请求，最低两位rsLUN为00b表示responser的LUN(除了BMC为10b其他都是00b)；check 1字段占一个字节，表示图中IPMB消息第一行的checksum；rqSA字段占一个字节，20h表示发送者的地址，即BMC；rqSeq/rqLUN字段占一个字节，其中最高6位rqSeq表示序列号，最低两位rqLUN 10b表示BMC SMS LUN，这将指示responser将对Set Event Receiver命令的响应发送到BMC的Receive Message Queue；Cmd字段占一个字节，00h表示Set Event Receiver命令；接下来的event receiver slave address和event receiver LUN字段是Set Event Receiver命令的数据，这里总共占两个字节，但不同命令占的字节数不一样，对应图1的Request消息格式中的data bytes；最后check 2表示图中ipmb消息第二行和第三行的checksum，占一个字节。

### mcu发消息给host

​		SMC通过设置rqLUN为10b将IPMB消息放置到BMC的Receive Message Queue中，然后host软件可以使用 Get Message命令来从Receive Message Queue取出IPMB消息。如果这些IPMB消息是SMC主动发起的而不是一个response消息，那么host从Receive Message Queue取出IPMB消息后然后通过Send Message命令返回一个response。
​		对Set Event Receiver命令的IPMB响应仅包含IPMB消息的数据部分中的一个完成代码字节。假设完成代码为00h = OK，则Receive Message Queue最终将结束一个包含以下内容的响应消息：

![](D:\bsp\docs\yeshen-md\pictures\impbresponse.PNG)

请注意，这是整个IPMB响应消息，去掉了前导从地址（前导从地址不需要存储该地址，因为已知它是BMC从地址= 20h）。然后，对Get Message命令的响应将如下所示。粗体字段显示来自Receive Message Queue的响应的数据部分。

![](D:\bsp\docs\yeshen-md\pictures\getmessageresponse.PNG)