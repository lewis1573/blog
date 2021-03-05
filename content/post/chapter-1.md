---
date: 2021-03-02
description: "含有处理器的系统"
featured_image: "/images/Pope-Edouard-de-Beaumont-1844.jpg"
tags: ["芯片验证"]
title: "System Verilog与 C语言统一验证环境（一）"
---

# 标题一

```
axi_slv.monitor.item_observed_port.connect(tube.analysis_export);

```

```
class tube extends uvm_subscriber#(svt_axi_transaction);

```

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

## 标题二

正文

### 标题三

正文
>  yinyong;
>  
>  neirong  
>  
>    前面四个空格
>    
>  two space before

