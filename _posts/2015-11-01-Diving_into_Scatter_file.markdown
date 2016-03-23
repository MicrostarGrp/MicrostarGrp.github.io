---
layout: post
title:  "Diving into Scatter file"
date:   November 1, 2015
keywords: Diving,Scatter,file,分散,加载
categories: embedded notes
comments: true
---

深入浅出分散加载描述文件
============
概述
------------
分散加载文件，即[scatter-loading description file]，用来配置链接工具实现镜像的存储分布，镜像由区和输出段组成，区包含不同的加载和执行地址。    
镜像的存储结构组成：

1. 输出段和区的输入段分组信息
2. 指定区域分配的地址信息

当链接器通过调用分散文件创建镜像文件，它根据代码的应用建立特定的区域代号。   
注：本文基于ARM Compiler V5.06

适用场景
------------
1. 复杂的存储分配：代码和数据需要置于多个独立的存储区域
2. 多样的存储类型：如SRAM、ROM、SDRAM、Flash等，他们的具有不同的性能及特征
3. 外设地址映射：以指定地址的方式映射外设存储空间
4. 函数固定地址：将函数指定在同一位置即使其余的代码被修改或重新编译
5. 堆栈标识：建立堆栈标示符，从而指定它们的位置

详细内容
------------

范式标记
===========
- "，引号表示使用字符的字面值，不记入语法，如：B"+"C只等价于B+C
- A ::= B，定义B用A表示，如A::=B"+" \| C，表示A等于B+或C
- [A]，A是可选的，如B[C]D表示BD或BCD
- A+，表示1个及以上，如B+可表示B/BB/BBB...
- A*，表示0个及以上
- A \| B，或逻辑，二选一
- (A B)，与逻辑，两个均包含

语法
===========
分散加载文件可以包含一个及以上的加载区，加载区可包含一个及以上的执行区。 

加载区
----------
成员：名称、地址/偏移地址、属性、最大长度

    load_region_description ::=
      load_region_name (base_address | ("+" offset)) [attribute_list] [max_size]
           "{"
              execution_region_description+ 
           "}"

1. 名称区分大小写
2. 地址应匹配对齐方式
3. 长度单位是字节
4. 属性选项：ABSOLUTE/ALIGN alignment/NOCOMPRESS/OVERLAY/PI/PROTECTED/RELOC

执行区
----------
成员：名称、地址、属性、最大长度

    execution_region_description ::= 
      exec_region_name (base_address | "+" offset) [attribute_list] [max_size | length]
            "{" 
                input_section_description* 
            "}"

1. 执行区对齐属性，加载区和执行区地址都应满足
2. 长度值为负数时，地址表示加载区结束地址
3. 属性选项（增加的）：ALIGNALL value/ANY\_SIZE max\_size/FILL value/FIXED/PADVALUE value/UNINIT/ZEROPAD/EMPTY [-]length

例：
    
    LOAD_ROM 0x0000     ; 加载区
    {
        EXEC_ROM        ; 执行区
        {
            program.o(+RO)  ; 输入段及匹配模式
        }
        DRAM 0x18000 0x8000 ; 执行区
        {
            program.o(+RW,+ZI)  ; 输入段及匹配模式
        }
    }

输入段
----------
成员：目标文件、段选择、标示符名

    input_section_description ::= 
      module_select_pattern 
            [ "(" input_section_selector ( "," input_section_selector )* ")" ]  input_section_selector ::= 
            ("+" input_section_attr | input_section_pattern | input_symbol_pattern | section_properties)

1. \*表示任意字符，?表示任意单字符，如*.o表示所有目标文件
2. 段名匹配，不区分大小写
3. 属性：+FIRST/+LAST/OVERALIGN value

选择关键词

- RO-CODE，同义词：CODE
- RO-DATA，同义词：CONST
- RO，表示RO-DATA和RO-CODE，同义词：TEXT
- RW-DATA
- RW-CODE
- RW，表示RW-DATA和RW-CODE，同义词：DATA
- XO
- ZI，同义词：BSS
- ENTRY

全局前缀

- :gdef:

例1：

    LoadRegion 0x8000
    {
        ExecReg1 +0
        {
            *(:gdef:mysym1)     ; 全局量mysym1、mysym2
            *(:gdef:mysym2)
        }
            ; 其他加载描述内容
    }

伪属性

- FIRST
- LAST

例2：

- *匹配目标文件或库文件
- *.o匹配目标文件
- math.o匹配只该目标文件
- "file 1.o"匹配目标文件file 1.o
- \*arm*匹配包含arm的文件

表达式
----------
分散加载文件包含的数值，它可以是具体的值，也可以是表达式的值。暂不提及，自行了解。

对齐方式
===============

源文件
------------
通过关键字\_\_align(n)关键字限定对齐方式    

分散加载
------------
使用ALIGNALL/OVERALIGN指定对齐方式    
例1：  

    ER_DATA … ALIGNALL 8
    {
       …
    }

例2

    ER_DATA …
    {
       *.o(.bar, OVERALIGN 8)
       …
    }

引导执行区
------------
镜像的初始入口必须位于引导区，否则链接器会给出错误信息。    
例1：

    LR_1 0x040000          ; 加载区开始地址 0x40000
    {                      ; 执行区描述起始
        ER_RO 0x040000     ; 地址相等
        {
            * (+RO)        ; 所有RO段 (包含初始入口)
        }
        …                  ; 其余分散加载信息
    }

例2：

    LR_1 0x040000
    {
        ER_RO 0x040000
        {
            * (+RO)
        }
        ER_INIT 0x080000 FIXED ; 指定该执行区加载地址和执行地址 0x80000
        {
            init.o(+RO)        ; init.o的所有RO段
        }
        …
    }

函数与数据
------------
将函数或数据置于指定的地址有以下途径：

- 使用\_\_attribute\_\_((at(address)))指定变量于指定的地址
- 使用\_\_attribute\_\_((section("name")))指定函数或变量于指定名称的段
- 使用AERA控制汇编代码中的存储分配
- 使用编译器选项--split_sections为每个函数生成独立的段

例：    
源代码：

    int variable __attribute__((section("foo"))) = 10;

分散加载文件：

    FLASH 0x24000000 0x4000000
    {
        …                                ; 其他分配信息
        ADDER 0x08000000
        {
            file.o (foo)                 ; 选择file.o中的foo段
        }
    }

空区
------------
设置无内容的存储块，保留指定区域另有用途，如堆/栈，使用EMPTY标示符。
例：  

    LR_1 0x80000   
    {
        STACK 0x800000 EMPTY -0x10000   ; 结束于0x800000，长度0x10000
        {
        
        }
        HEAP +0 EMPTY 0x10000           ; 始于前区末尾，长度0x10000
        {
        
        }
        …
    }

注：EMPTY执行区的ZI区将不会被运行库初始化为零。


自定义堆栈
------------
定义2个EMPTY执行区，名为ARM\_LIB\_HEAP、ARM\_LIB\_STACK，这时运行库会调用\_\_user\_setup\_stackheap()选择非默认的实现而使用以下标识符：

    Image$$ARM\_LIB\_STACK$$Base
    Image$$ARM\_LIB\_STACK$$Limit
    Image$$ARM\_LIB\_HEAP$$Base
    Image$$ARM\_LIB\_HEAP$$Limit

当然，我们可以重定义该函数，覆盖运行库的实现。

缺省未定义的标识符
------------

    Image$$RO$$Base
    Image$$RO$$Limit
    Image$$RW$$Base
    Image$$RW$$Limit
    Image$$XO$$Base
    Image$$XO$$Limit
    Image$$ZI$$Base
    Image$$ZI$$Limit
