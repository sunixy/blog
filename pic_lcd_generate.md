#PIC单片机驱动LCD断码屏代码生成工具
PIC 16系列单片机部分型号都带有LCD（断码）驱动模块，
在编写这部分代码的过程中发现对驱动管脚赋值逻辑有很大的重复部分。
所以写了一个宏函数，自动生成驱动逻辑。

##断码屏与驱动管脚对应关系
断码屏的一位一般由8段构成：A,B,C,D,E,F,G,DP。

构成对应图形：
```cpp
     AAAAAA
    F      B
    F      B
    F      B
     GGGGGG
    E      C
    E      C
    E      C
     DDDDDD  DP
```
每段对应PIC单片机的一个驱动管脚。而这个管脚由PIC单片机LCD驱动模块的LCDDATAX寄存器
控制。通过PIC的寄存器映射头文件可以表示为COMxSEGy。

##头文件进行管脚配置
首先定义每一位的各个段：

```cpp
#define LCD_A1          SEGx1COMy1
#define LCD_B1          SEGx2COMy2
#define LCD_C1          SEGx3COMy3
#define LCD_D1          SEGx4COMy4
#define LCD_E1          SEGx5COMy5
#define LCD_F1          SEGx6COMy6
#define LCD_G1          SEGx7COMy7
#define LCD_DP1         SEGx8COMy8
```
不同的数字由各段的亮暗来表示。例如第一位数字0表示为：
```cpp
LCD_A1 = 1;
LCD_B1 = 1;
LCD_C1 = 1;
LCD_D1 = 1;
LCD_E1 = 1;
LCD_F1 = 1;
LCD_G1 = 0;
```
之后还需要定义数字位图，例如数字1和0定义为：
```cpp
#define LCD_DIGIT_0     (LCD_SEG_A|LCD_SEG_B|LCD_SEG_C|LCD_SEG_D|LCD_SEG_E|LCD_SEG_F)
#define LCD_DIGIT_1     (LCD_SEG_B|LCD_SEG_C)
```

##函数自动生成
之后对断码屏的每一位显示来说，就是对相应管脚的0,1赋值。利用定义好的
命名规则可以利用宏函数来自动生成对应的显示函数。

```cpp
#define SEG_VALUE(seg, num, character) \
    if (character&LCD_SEG_##seg) \
    { \
        LCD_##seg##num = 1; \
    } \
    else \
    { \
        LCD_##seg##num = 0; \
    }

#define SEG_GENERATE(num) \
    SEG_VALUE(A, num, character) \
    SEG_VALUE(B, num, character) \
    SEG_VALUE(C, num, character) \
    SEG_VALUE(D, num, character) \
    SEG_VALUE(E, num, character) \
    SEG_VALUE(F, num, character) \
    SEG_VALUE(G, num, character) \


#define generate_begin(page)  \
void lcd_##page##_display_value(unsigned char column, unsigned char character) \
{ \
    switch (column) \
    {

#define generate_end() \
    default: \
        break; \
    } \
}

#define gnerate_column(col, num) \
    case col: \
    SEG_GENERATE(num); \
    break;

 /*define generate micro-define*/
/*need add generate_column logic*/
#define generate_page1_lcd_display_value() \
    generate_begin(page1) \
    gnerate_column(LCD_COLUMN_1, 1) \
    gnerate_column(LCD_COLUMN_2, 2) \
    gnerate_column(LCD_COLUMN_3, 3) \
    gnerate_column(LCD_COLUMN_4, 4) \
    generate_end()

/*void lcd_page1_display_value(unsigned char column, unsigned char character)*/
generate_page1_lcd_display_value()
```
##结束
生成显示函数以后可以像如下调用：
```cpp
lcd_page1_display_value(LCD_COLUMN_1, LCD_DIGIT_0);
```
在这里只是贴出了一个梗概，其中很多细节并没有写到，后续会考虑在github
上贴出完整代码。
如果有不明白的地方，尽可以提出。

