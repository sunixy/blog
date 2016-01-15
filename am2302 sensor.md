# 单片机与AM2302温湿度传感器通信优化
AM2302温湿度传感器采用单总线方式与MCU通信，这就要求MCU有一定的处理速度,
才能正确解析收到的AM2302发送过来的数据。

## MCU处理AM2302数据的方式
AM2302一次传送40位数据给MCU。数据位0由50微妙低电平加26微妙高电平组成。
数据位1有50微妙低电平加70微妙高电平组成。这种编码方式有点象NEC的红外传输协议。


另外AM2302需要由MCU发起启动信号。所以针对这种单线协议，虽然可以采用电平
变化中断+计数器，或输入捕捉来解析40位数据位。但这就需要切换端口的输入输出
配置及控制相应外设的介入时机。

本文介绍的方法采用简单的端口读+延时操作来解析40位数据位。

```cpp
    if (data_port == 1)
        delay_us(30);
    if (data_port == 1)
        //bit = 1
    else
        //bit = 0
```

起始信号通过把端口改为输出，然后通过写端口+延时来实现。
```cpp
    //改变data_port为输出
    data_port = 0;
    delay_us(1000);
    data_port = 1;
    delay_us(20);
    //改变data_port为输入
```
## 数据读取函数实现
根据上述协议的描述，很容易抽象出如下函数：
```cpp
static unsigned char am2302_read_byte(void)
{
    unsigned char i = 0;
    unsigned char data = 0;

    for (i = 0; i < 8; i++)
    {
        //50us low
        while (0 == data_port)
        {
        }
        delay_us(40);
        if (0 == data_port)
        {
            continue;
        }
        else
        {
            data += (0x80U >> i);
            while (1 == data_port)
            {
            }
        }
    }
    return data;
}
```
通过调用am2302_read_byte() 5次，把40位数据读取出来。
```cpp
    humidity_hign = am2302_read_byte();
    humidity_low = am2302_read_byte();
    temperature_high = am2302_read_byte();
    temperature_low = am2302_read_byte();
    checksum = am2302_read_byte();
```

## 为什么上面的函数不能使用了
在某些应用场景下，为了降低功耗，需要把MCU的工作频率降到尽可能的低。
如果在系统时钟很低的情况,指令周期就成为需要考虑的关键因素。


这里拿PIC单片机举例，如果系统时钟为1M Hz，则它的指令周期为4微妙,
(指令周期为系统时钟的4倍)。
这个时候如果使用上面提到的函数调用的方法，将无法得到正确的数据。
因为加上函数调用的开销，当am2302_read_byte()进行电平判断的时候，
很可能已经错过了起始电平，导致解析不正确。另外当判断是数据位1的时候，
```cpp
    data += (0x80U >> i);
    while (1 == data_port)
    {
    }
```
理论上上面的操作要在40~50微妙的时间内完成，大概是10~12个汇编指令。
但目前上面的操作会转换成很多汇编指令，耗费过多的时间，导致后续数据位解析不正确。

## 解决方案
+ 简单的方案，继续使用上面的函数，但需要在调用之前提高系统时钟，缩短指令周期即可。
但对功耗上有些许影响，但基本影响不会太大。这里比较要命的是你提高了系统之中，依赖
系统时钟的外设都要重新设置，例如定时器。当完成温湿度的读取，又要全部切换回来。
+ 考验功力的方案，有没有可能优化上面的数据读取函数，减少生成的汇编指令，使它能够
在1 MHz的系统时钟下，完成数据读取？

## 如何改进
+  利用空间换时间的思路，取消函数调用，把里面的逻辑展开。这里可以利用宏函数实现。

```cpp
#define am2302_read_byte(data)  \
                am2302_read_bit(data) \
                am2302_read_bit(data) \
                am2302_read_bit(data) \
                am2302_read_bit(data) \
                am2302_read_bit(data) \
                am2302_read_bit(data) \
                am2302_read_bit(data) \
                am2302_read_bit(data)
```
+ 优化数据位1的实现逻辑，把移位操作转换成定值的赋值操作。

```cpp
#define am2302_read_byte(data)  \
                am2302_read_bit(data, 0x80) \
                am2302_read_bit(data, 0x40) \
                am2302_read_bit(data, 0x20) \
                am2302_read_bit(data, 0x10) \
                am2302_read_bit(data, 0x08) \
                am2302_read_bit(data, 0x04) \
                am2302_read_bit(data, 0x02) \
                am2302_read_bit(data, 0x01)

#define am2302_read_bit(data, bitmask)    \
                    while (0 == am2302_data_PORT) \
                    { \
                    } \
                    __delay_us(26); \
                    if (1 == am2302_data_PORT) \
                    { \
                        NOP(); \
                        NOP();\
                        if (1 == am2302_data_PORT) \
                        { \
                            data += bitmask; \
                            while (1 == am2302_data_PORT) \
                            { \
                            } \
                        } \
                    }
```

+ 但在实际的调试过程中，发现有时候还是无法完整的解析数据。特别是当数据位1特别多的
时候，往往不能够正确解析。这时候就需要仔细的分析数据位1的生成汇编代码。

```cpp
    movlb	0	; select bank0
    btfss	12,0	;volatile //对I/O进行判断，相当于if (1 == am2302_data_PORT)
    goto	l435    //I/O不是1，跳转到下一个数据位判断逻辑
    movlw	128     //I/O是1，对数据进行加1操作，这里使用了两条指令
    addwf	_g_th,f
l435:	
```

+ 使用|=替换+=，把两条指令的加1操作变成一条指令。

```cpp
    if (1 == am2302_data_PORT) \
    { \
        data |= bitmask; \
        while (1 == am2302_data_PORT) \
        { \
        } \
    } \
```

它生成的汇编代码变成：

```cpp
    movlb	0	; select bank0
    btfss	12,0	;volatile
    goto	l435
    bsf	_g_th,7
l435:
```

##结束

使用上述方案，可以使PIC单片机在1 MHz的系统时钟下，与AM2302进行单线通信。
