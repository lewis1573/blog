---
date: 2021-03-02
description: "含有处理器的系统"
featured_image: "/images/Pope-Edouard-de-Beaumont-1844.jpg"
tags: ["芯片验证"]
title: "System Verilog与 C语言统一验证环境（一）"
---

# 背景
在SOC的设计中，大部分含有处理器，比如ARM、RISC-V以及各种DSP等等，这些处理器IP基本都有成熟的编译工具链，可以方便的编译运行C代码，如果验证平台能够利用设计中的处理器本身执行C的用例，可以减少很多平台搭建的工作，同时因为C语言相对来说更加容易上手，代码在不同验证环境，包括硬件仿真平台的可复用性也更好。这里的统一验证环境指的就是在SystemVerilog及UVM搭建的验证平台中集成DUT中处理器的编译执行流程，让验证平台同时支持SystemVerilog和C语言的用例。

# 实现
SystemVerilog的验证平台在simv运行后会有打印log，一般会通过log检查case的运行结果，如果C的用例也能够打印到SV的log中，就可以实现基本的统一验证环境，但是默认情况下使用C的printf函数并不会像我们预期的那样打印，因为printf是C语言的标准库，是由开源的C lib库或者处理器厂家实现的，它会调用操作系统底层的输出IO，如果代码运行在完整的linux主机上，那么就会打印到终端窗口中，但是我们的DUT一般没有标准输出IO，所以需要将printf重写，让printf输出到某一个固定的地址，比如一块可读可写的memory区域，然后让SystemVerilog监测这个地址，如果有写操作，将写的内容打印出来，这样C的printf信息就能够和SystemVerilog的打印信息一起打印到log里了。这样整个流程需要一块memory作为管道，C的环境需要重新实现printf，SystemVerilog需要实现对一个地址写访问的检测并打印，下面分别详细说明：

## 管道
处理器一般会提供AXI接口，Synopsis的AXI验证IP（VIP）实现了axi_slave_agent，在这个agent中就包含了一块memory，配置了正确的地址后，对axi slave vip的访问就是对这块memory的读写了。这样将axi slave vip连接到处理器的AXI接口，接口地址范围内的任意一个地址都可以作为C和Systemvirlog通信的管道。

## 重写C的printf
从0实现一个printf还是比较复杂的，感兴趣的可以自行网上搜索，这里本着不重新造轮子的原则，利用现有的工具。不同的处理器可能会选择不同的编译工具链，不管是GCC还是Clang，会有不同C的标准库，如果是arm使用GCC的工具链，那就会比较简单，网上也有很多资料，通过重写`_write`函数重定向printf，这里就不再赘述。如果是其他的，比如我遇到的DSP，使用Clang，它的标准库实现就没有`_write`接口让我去重写，虽然实现机制不同，但是作为标准库，标准的函数都是完整提供的，可以利用标准库的`sprintf`，函数调用和printf一样，将需要打印的数据和字符串等格式化输出到一个`buffer`中，然后再将`buffer`中的字符输出，这样就可以免去处理要打印几个数据，以十六进制还是十进制打印等格式化的问题，`sprintf`的函数原型如下：

```c
int sprintf (char * szBuffer, const char * szFormat, ...)
```
最后的三个点表示部确定个数和内容的参数，这样我们重新写一个sprintf就可以，但是这种形参的传递方式只能在函数最外层调用，无法再封装，调用不够直观，那么有没有更加优雅的方式呢，答案是有的，那就是`vsprintf`,它的函数原型如下：

```c
int vsprintf(char *string, char *format, va_list param);
```
它将可变参数使用一种预定义数据类型传递，可以再次封装，另外`vsprintf`没有对打印的长度做限制，可能会引起内存溢出，更好的方式是使用`vsnprintf`

## 交互

提前确定好一个RAM的地址，用来作为C和SystemVerlog交互的管道，将C的打印转换为对这个地址一个字节一个字节的写操作，同时需要定义一些特殊的字符，比如使用 CTRL-D 即assic码`\x04`，表示C的用来全部结束，可以退出仿真了。

实现的源码如下，为了避免和printf的冲突，使用tube_printf命名，需要注意的是C的代码最后主动调用`tube_exit`，以便通知SystemVerlog自己运行结束了。

```c
#include <stdarg.h>
#include <stdio.h>

#define MAX_CHAR_NUM 256
#define TUBE_ADDRESS 0x13000000u

char tube_printf_buffer[MAX_CHAR_NUM];
char * p_tube_printf_buffer;
volatile char * p_tube = (volatile char *)TUBE_ADDRESS;

void tube_output_char(char output)
{
    *p_tube = output;
}

void tube_printf(const char* fmt, ...)
{
    va_list argptr;
    va_start(argptr, fmt);
    vsnprintf(tube_printf_buffer, MAX_CHAR_NUM, fmt, argptr);
    va_end(argptr);

    p_tube_printf_buffer = &tube_printf_buffer[0];
    do
        tube_output_char(*p_tube_printf_buffer++);
    while((*p_tube_printf_buffer) != '\0');
}

//write CTRL-D (EOT charactter)
//which causes the simulation to terminate
void tube_exit(void)
{
    tube_output_char('\x04');
    //loop forever until the simulator terminates
    while(1);
}
```
## 通过SystemVerilog监听和处理

因为交互的管道是通过AXI slave agent实现，对管道的读写必然要经过AXI接口，所有监听自然要利用AXI接口，可以在这个AXI的接口上连接一个analysis_port的监听器，在这个监听器中实现自己的write函数，当监听到有数据写入时执行自己wirte函数中的代码，这样不会对总线上的其他数据接收方造成任何影响，UVM也定义了这样的类`uvm_subscriber`可以供我们直接继承。这里的transiction可以使用VIP提供的数据结构，类的申明如下：

```systemverilog
class tube extends uvm_subscriber#(svt_axi_transaction);

```
uvm_subscriber类包含一个名为write的纯虚函数，在继承后需要自己实现，在write函数中判断写操作的地址，如果是管道地址，写入的数据就是C要打印的字符，将写入的数据依次存入数组。如果检查到写入的数据是回车或者换行，则表明这句C的打印结束了，这时候就可以使用systemverilog的打印函数将数组中的所有字符，也就是C的打印信息到Systemverilog的log中了。如果检查到写入的数据是CTRL-D，也就是`\x04`则表示C的用例结束，Systemverilog可以调用drop_objection结束仿真。

另外，定义的tube对象需要与axi vip连接
```systemverilog
axi_slave.monitor.item_observed_port.connect(tube.analysis_export);

```

