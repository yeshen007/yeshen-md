## N430 Box上下电流程

## 1. 上电 powerup

#### 1.0 12V上电后

1. MCU需要检测各级电源稳定
2. MCU控制LED灯亮起(参考英码科技产品)

#### 1.1 用户按power按键

1.  用户触发power按键(MCU GPIO)
2.  MCU 检测到power按键电平被拉低一段时间
3.  MCU通过MCU GPIO控制12V次级电源上电
4.  MCU需要检测各级电源稳定再sleep 1s
5.  MCU通过GPIO控制Bass POR
6.  Linux正常启动后给出指示灯

#### 1.2 wifi远程控制上电

1.  用户通过wifi远程通知MCU上电
2.  MCU接收到用户远程下发的指令
3.  MCU软件进入上电流程
4.  MCU通过MCU GPIO控制12V次级电源上电
5.  MCU需要检测各级电源稳定再sleep 1s
6.  MCU通过GPIO控制Bass POR
7.  Linux正常启动后给出指示灯

## 2. 下电 poweroff

#### 2.1 用户输入poweroff命令

1.  用户输入poweroff命令
2.  Bass Linux进入poweroff流程
3.  BL31进入poweroff流程，指示灯灭掉
4.  BL31通过通信协议通知MCU，Bass已经完成poweroff流程
5.  MCU 控制POR拉低
6.  MCU控制掉电 

#### 2.2 用户按power按键

1.  用户触发power按键(MCU GPIO)
2.  MCU 检测到power按键电平被拉低一段时间
3.  MCU通过通信协议通知Bass Linux关机
4.  Bass Linux收到关机指令
5.  Bass Linux进入poweroff流程，余下见2.1流程

#### 2.3 wifi远程控制下电

1.  用户通过wifi远程通知MCU下电
2.  MCU通过通信协议通知Bass Linux关机
3.  Bass Linux收到关机指令
4.  Bass Linux进入poweroff流程，余下见2.1流程

## 3. RESET

#### 3.1 Bass Linux中的软件触发software reset

1.  Bass进入 sortware reset & wdt 进入重启流程（异常流程）
2.  再此器件的喂狗怎么做？

#### 3.2 用户按下reset按键

1.  用户按下reset按键(MCU GPIO)
2.  POR down & led灭
3.  MCU 控制12v断电
4.  MCU控制12v上电
5.  POR UP & led 亮

## 4. 重启 reboot

#### 4.1 reboot

1.  用户输入reboot，进入常规重启流程（会上下电）
2.  reboot需要多长时间，MCU timeout设置多少比较合理？

#### 4.2 reboot recovery

1.  用户命令行输入reboot recovery，重启需要进入recovery mode
2.  recovery mode如何给MCU喂狗？

## 5. 超时 timeout

1.  Bass Linux未按时发送心跳给MCU
2.  MCU检测到Bass心跳超时
3.  POR down & led灭
4.  MCU复位Bass芯片
5.  POR UP & led 亮

