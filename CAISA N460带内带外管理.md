# <center>CAISA N460带内带外管理</center>

## 一、带内带外管理功能

N460支持带内带外管理功能，主要支持芯片温度、板级温度、板级功耗、设备id等信息的读取。
N460带内带外管理框图如下：

![](D:\bsp\docs\yeshen-md\pictures\ioband.png)

N460板卡上有一个温度传感器tsensor和一个电源传感器pmic，这两个传感器作为i2c从设备挂在mcu的i2c总线上。bass通过uart和mcu通信。mcu则作为从设备挂在host金手指的smbus上。host通过smbus发送命令给mcu，mcu再通过i2c读取传感器信息或者通过uart从bass获取信息传给host。



## 二、带内带外信息的读取

|                         | i2c addr | offset | width  |         值的含义         |                             注释                             |
| :---------------------: | :------: | :----: | :----: | :----------------------: | :----------------------------------------------------------: |
|    CHIP0 Vendor name    |   0x13   |  0x01  | 4byte  | bass0芯片厂家名字ascii码 |                    从bass0 soc reg中获取                     |
|    CHIP1 Vendor name    |   0x13   |  0x02  | 4byte  | bass1芯片厂家名字ascii码 |                    从bass1 soc reg中获取                     |
|    CHIP2 Vendor name    |   0x13   |  0x03  | 4byte  | bass2芯片厂家名字ascii码 |                    从bass2 soc reg中获取                     |
|       CHIP0 name        |   0x13   |  0x04  | 4byte  |   bass0芯片名字ascii码   |                    从bass0 soc reg中获取                     |
|       CHIP1 name        |   0x13   |  0x05  | 4byte  |   bass1芯片名字ascii码   |                    从bass1 soc reg中获取                     |
|       CHIP2 name        |   0x13   |  0x06  | 4byte  |   bass2芯片名字ascii码   |                    从bass2 soc reg中获取                     |
|      CHIP0 version      |   0x13   |  0x07  | 4byte  |  bass0芯片版本号ascii码  |                   从bass0 soc efuse中获取                    |
|      CHIP1 version      |   0x13   |  0x08  | 4byte  |  bass1芯片版本号ascii码  |                   从bass1 soc efuse中获取                    |
|      CHIP2 version      |   0x13   |  0x09  | 4byte  |  bass2芯片版本号ascii码  |                   从bass2 soc efuse中获取                    |
|        CHIP0 ID         |   0x13   |  0x0a  | 16byte |       bass0芯片ID        |                   从bass0 soc efuse中获取                    |
|        CHIP1 ID         |   0x13   |  0x0e  | 16byte |       bass1芯片ID        |                   从bass1 soc efuse中获取                    |
|        CHIP2 ID         |   0x13   |  0x12  | 16byte |       bass2芯片ID        |                   从bass2 soc efuse中获取                    |
|     PCIe Vendor ID      |   0x13   |  0x16  | 4byte  |      PCIe Vendor ID      |                  从soc pcie ctrl reg中获取                   |
|     PCIe Device ID      |   0x13   |  0x17  | 4byte  |      PCIe Device ID      |                  从soc pcie ctrl reg中获取                   |
|   PCIe Sub-Vendor ID    |   0x13   |  0x18  | 4byte  |    PCIe Sub-Vendor ID    |                   默认设置成PCIe Vendor ID                   |
|   PCIe Sub-System ID    |   0x13   |  0x19  | 4byte  |    PCIe Sub-System ID    |                   默认设置成PCIe Device ID                   |
|    CHIP0 Temperature    |   0x13   |  0x1a  | 12byte |   bass0芯片tsensor温度   | 前四个字节是bass0的cpu和ddr的平均值，中间四个字节是bass0的ai0的温度，最后四个字节是bass0的ai1的温度 |
|    CHIP1 Temperature    |   0x13   |  0x1d  | 12byte |   bass1芯片tsensor温度   | 前四个字节是bass1的cpu和ddr的平均值，中间四个字节是bass1的ai0的温度，最后四个字节是bass1的ai1的温度 |
|    CHIP2 Temperature    |   0x13   |  0x20  | 12byte |   bass2芯片tsensor温度   | 前四个字节是bass2的cpu和ddr的平均值，中间四个字节是bass2的ai0的温度，最后四个字节是bass2的ai1的温度 |
|    CARD Vendor name     |   0x13   |  0x23  | 4byte  |   板卡厂家名字 ascii码   |                        mcu程序中写死                         |
|    CARD Device name     |   0x13   |  0x24  | 4byte  |   板卡设备名字 ascii码   |                      mcu从串码SN中提取                       |
|     CARD Device SN      |   0x13   |  0x25  | 16byte |          串码SN          |                        烧到mcu flash                         |
|  CARD Hardware Version  |   0x13   |  0x29  | 4byte  |        product id        |                       从soc reg中获取                        |
|  CARD Firmware Version  |   0x13   |  0x2a  | 4byte  |  板卡固件版本号 ascii码  |                        mcu程序中写死                         |
| CARD Manufacturing Time |   0x13   |  0x2b  | 4byte  |       板卡生产时间       |                      mcu从串码SN中提取                       |
|    CARD Temperature     |   0x13   |  0x2c  | 4byte  |         板卡温度         |               mcu通过i2c从板卡tsensor获取温度                |
| CARD Power consumption  |   0x13   |  0x2d  | 4byte  |         板卡功耗         |                    (iout*vout) / 0.9 + 9W                    |

