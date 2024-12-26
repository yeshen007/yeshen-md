# PPE运算限制

​		ppe运算主要由pp_core的计算单元从pp_core的存储单元pp_dbuf中取数进行计算，因此pp_dbuf的容量影响了每次ppe运算所能处理的数据量大小，超过了pp_dbuf的数据要blocking，即分块处理，pp_dbuf的容量从文档得到的配置是65536x32bit。每个算子ppe输入和输出数据需对齐到 1024bit，即输入和输出数据量是 1024bit 的整数倍。除此之外，每个ppe算子可能还会有一些特定的限制。

|   参数    |                含义                 |
| :-------: | :---------------------------------: |
|     H     |              输入高度               |
|     W     |              输入宽度               |
|     C     |             输入通道数              |
|    Ho     |              输出高度               |
|    Wo     |              输出宽度               |
|    Co     |             输出通道数              |
| PPDB_SIZE | pp_dbuf size(当前配置：65536x32bit) |



## yuv2rgba

### blocking限制

input 超过PPDB_SIZE时blocking，blocking方向h/w

### blocking策略

- 按w方向blocking：
- 按h方向blocking：
- 按hw组合blocking：

### 算子自身限制

h和w要四字节对齐。



## rgba2yuv

### blocking限制

input 超过PPDB_SIZE时blocking，blocking方向h/w

### 算子自身限制

h和w要四字节对齐。



## padding

### blocking限制

input h\*w\*c超过PPDB_SIZE时blocking，blocking方向h/w/c

### 算子自身限制

无



## crop

### blocking限制

input h\*w\*c超过PPDB_SIZE时blocking，blocking方向h/w/c

### 算子自身限制

output size：Wo\*Ho\*Co%16=0




## resize

### blocking限制

input h\*w\*c超过PPDB_SIZE时blocking，blocking方向h

### 算子自身限制

input size：

- UINT8 NN/Bilinear：W*ceil(C/4) <= PPDB_SIZE/2, C==3 or C%4==0
- UINT8 Cubic：W*ceil(C/4) <= PPDB_SIZE/4, C==3 or C%4==0
- FP32 NN/Bilinear：W*C<= PPDB_SIZE/2
- FP32 Cubic：W*C<= PPDB_SIZE/4

output size：

- NN/Bilinear：Wo <= 8192
- Cubic：Wo <= 4096



