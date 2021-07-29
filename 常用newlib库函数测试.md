# </center>常用newlib库函数测试<center>

​		由于newlib库函数数量庞大，无法一一测试验证，这里提取几个裸机中常用的库函数进行测试验证，这几个函数分别是memset、memcpy、memcmp、strlen。

### memset测试

![](E:\叶神文档\Markdown及其pdf\pictures\2\1.0.PNG)

![](E:\叶神文档\Markdown及其pdf\pictures\2\1.1.PNG)

上图展示的是通过memset将地址0x20000000处开始的8字节设置成0x0c，从第2图的内存显式中可以看到确实该处8字节都变成了0x0c。因此memset测试没有问题。

### memcpy测试

![](E:\叶神文档\Markdown及其pdf\pictures\2\2.0.PNG)

![](E:\叶神文档\Markdown及其pdf\pictures\2\2.1.PNG)

![](E:\叶神文档\Markdown及其pdf\pictures\2\2.2.PNG)

上图展示的是用memcpy将p指向的0x20000000处的8字节内容复制到pp指向的0x20100000处。从图中可以看到p处的8字节内容为0xDFFDDCB5,0x0E7CDFED。未复制前pp处的8字节内容为0xDFFDF4B5,0xBE78FFFD。复制后pp处的8字节内容变为0xDFFDDCB5,0x0E7CDFED，和p处的内容一样。因此memcpy测试没有问题。

### memcmp测试

![](E:\叶神文档\Markdown及其pdf\pictures\2\3.0.PNG)

![](E:\叶神文档\Markdown及其pdf\pictures\2\3.1.PNG)

![](E:\叶神文档\Markdown及其pdf\pictures\2\3.2.PNG)

上图分别将p和pp指向的字符串用memcmp进行比较，第一图返回值为-32,因为aBcd中的B比abcd中的b小32，第二图返回值为32，因为abcd中的a比Abcd中的A大32，第三图返回值为0，因为比较的字符串一样。因此memcmp测试没有问题。

### strlen测试

![](E:\叶神文档\Markdown及其pdf\pictures\2\4.0.PNG)

上图展示的是用strlen计算一个字符串“abcdefg”的长度，从结果看到返回值为7,刚好为该字符串的长度。因此strlen测试没有问题。