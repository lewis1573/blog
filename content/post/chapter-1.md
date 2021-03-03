---
date: 2021-03-02
description: "含有处理器的系统"
featured_image: "/images/Pope-Edouard-de-Beaumont-1844.jpg"
tags: ["芯片验证"]
title: "System Verilog与 C语言统一验证环境（一）"
---

# 标题一

```c
#define tube_printf(…)                                  \
do                                                      \
{                                                       \
  sprintf(tube_printf_cache,__VA_ARGS__);               \
  p_tube_printf_cache = &tube_printf_cache[0];          \
  do                                                    \
    tube_output_char(*p_tube_printf_cache++);           \
  while((*p_tube_printf_cache) != '\0');                \
}                                                       \
while(0)
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

