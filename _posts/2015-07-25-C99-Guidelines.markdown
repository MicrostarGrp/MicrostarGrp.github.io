---
layout: post
title:  "C99 Guidelines"
date:   July 25, 2015
keywords: C语言,C99,Guidelines,详解
categories: embedded notes
comments: true
---

概述
========
在嵌入式开发中，编译器已经支持C99，本文简要介绍C99的实用功能。要说明的是，很多编译器在标准化前，自行实现了部分，这并不矛盾。		
版本：ISO/IEC 9899:1999.The 1999 International Standard for C.

新特性
========
- 一些来自GNU在C90提供的扩展
- 一些来自C++特性
- 一些新的特性，如复数、初始化等
- 一些已有C90语法扩展

其他
========
由于新特性的引入，C99与C++不再完全兼容。

举例
--------

注释
========
{% highlight ruby %}
 x = a //* Hello World */ b
    - c;
{% endhighlight %}
* C90: x = a / b - c;
* C99: x = a - c;

复合表达式
========
{% highlight ruby %}
int *pint = (int [4]){1, 2 ,3, 0};
struct T
{
   int  num;
   char *name;
} t2;
t2 = (struct T) {43, "world"};
{% endhighlight %}
指定初始化
========
{% highlight ruby %}
typedef struct
{
    char *name;
    int  rank;
} data;
data vars[10] = { [0].name = "foo", [0].rank = 1,
                  [1].name = "bar", [1].rank = 2,
                  [2].name = "baz", 
                  [3].name = "gazonk" };    
{% endhighlight %}
注：未指定初始化的成员默认初始化为0。

16进制浮点数
========
{% highlight ruby %}
0x1.fp3; // 0x1.fp3 = 1.9375*8 = 1.55e1
{% endhighlight %}
注：可使用printf()以"%a"或"%A"输出该格式。

可变长度数组成员
========
{% highlight ruby %}
extern const int n;
typedef struct
{
    int len;
    char p[];
} str;
void foo(void)
{
    size_t str_size = sizeof(str);  // equivalent to offsetoff(str, p)
    str *s = malloc(str_size + (sizeof(char) * n));
}
{% endhighlight %}
注：当结构体包含可变数组成员，结构体是不完整类型，慎用sizeof()。

预定义标示符 \__func__
========
{% highlight ruby %}
void foo(void)
{
    printf("This function is called '%s'.\n", __func__);
}
{% endhighlight %}
输出：This function is called 'foo'.

内联函数
========
关键字：inline	
{% highlight ruby %}
inline int max(int a, int b)
{
    return (a > b) ? a : b;    
}
{% endhighlight %}
注：inline是内联函数的必要条件，C90中使用__inline，与C++中含义不同。

长整形
========
关键字：long long
{% highlight ruby %}
long long int j = 25902068371200;                // length of light
                                                 // day, meters
unsigned long long int i = 94607304725808000ULL; // length of light
                                                 // year, meters
{% endhighlight %}
注：64bits，C90不启用选项[--strict]时也支持，__int64与long long是等价的。

宏定义允许不定长参数
========
{% highlight ruby %}
#define debug(format, ...) fprintf (stderr, format, __VA_ARGS__)
void Variadic_Macros_0()
{
    debug ("a test string is printed out along with %x %x %x\n", 12, 14, 20);
}
{% endhighlight %}
不限定变量定义的位置
========
{% highlight ruby %}
void foo(float i)
{
    i = (i > 0) ? - i : i;
    i++;
    float j = sqrt(i);    // illegal in C90
}
{% endhighlight %}
for(;;)循环：
{% highlight ruby %}
extern int max;
for (int n = max - 1; n >= 0; n--)
{
    // body of loop
}
{% endhighlight %}
预处理操作符：__Pragma
========

{% highlight ruby %}
#define RWDATA(X) PRAGMA(arm section rwdata=#X)
#define PRAGMA(X) _Pragma(#X)
RWDATA(foo)  // same as #pragma arm section rwdata="foo"
int y = 1;   // y is placed in section "foo"
{% endhighlight %}
注：它在C90未启用选项[--strict]时是可用的。

复数
========
{% highlight ruby %}
#include <stdio.h>
#include <complex.h>
int main(void)
{
    complex float z = 64.0 + 64.0*I;
    printf("z = %f + %fI\n", creal(z), cimag(z));
    return 0;
}
{% endhighlight %}
注：支持以下类型：float complex/double complex/long double complex

[restrict]指针
========
关键字：restrict
{% highlight ruby %}
void copy_array(int n, int *restrict a, int *restrict b)
{
    while (n-- > 0)
    *a++ = *b++;
}
void test(void)
{
    extern int array[100];
    copy_array(50, array + 50, array);    // valid
    copy_array(50, array + 1, array);     // undefined behavior
}
{% endhighlight %}
注：表示指针仅本函数访问，这允许编译器更好地优化代码。 C90/C++均支持该选项。
<!>对于2个独有指针，不要试图传入同一参数。

库函数
========
- 一些UNIX标准库中C90的扩展，如snprintf()
- 一些新的库功能，如\<fenv.h>提供的标准化浮点数运行环境
- 一些C90已有的功能、宏定义与函数定义	  
注：由于有些功能已存在于C90/C++，必要时使用[USE_C99_ALL]和[USE_C99_MATH]指定调用。

个人体会
========
入行初期，我并不知道什么版本，就纳闷：变量定义怎么只能紧跟着大括号？
就去读标准文档，先是国标、C99等，厘清了编译器与标准的关系。
想想和教育背景有关联，没老师提标准文件，学校充斥着VC6.0，多数情况只是拿结果逆推语言。
我们了解根源，可以在骨子里建立自信，也得到了自由。	  
另，C++近期在新标准的推动下，表现强劲，大家也可以了解下，不过注意部分编译器还不完整支持C++11.

声明
========

|    版本    |   日期    |
|:---------:|:---------:|
|   V0.1a   |  25 July  |
|   V0.1b   |  26 July  |