+++
author = "GreyWind"
title = "The Go Programming Language 课后练习 Ch.2"
date = "2023-07-31"
description = "The Go Programming Language 课后练习 Ch.2"
tags = [
    "the-go-programming-language",
]
image="ab638d52-5f92-4e29-955e-064deb1c4728.jpeg"
+++

## 前言

第二章为Program Structure（程序结构），通过7个部分的示例代码深入讨论Go程序基础结构方面的一些细节。第二章课后练习比较基础，了解Go语言程序结构即可完成。

本章所有示例代码链接 [ch2](https://github.com/adonovan/gopl.io/tree/master/ch2) (有需自取，这里不做演示)

## 课后练习
### Packages and Files
示例代码是一个进行摄氏度和华氏度转换的程序，课后练习如下

> 练习 2.1： 向tempconv包添加类型、常量和函数用来处理Kelvin绝对温度的转换，Kelvin 绝对零度是−273.15°C，Kelvin绝对温度1K和摄氏度1°C的单位间隔是一样的。

思路

类似摄氏度和华氏度之间的转换，将开尔文与其转换。

```go
// tempconv.go in the tempconv package

type Celsius float64
type Fahrenheit float64
type Kelvin float64

const (
    AbsoluteZeroC Celsius = -273.15
    freezingF     Celsius = 0
    boilingC      Celsius = 100
)

func (c Celsius) String() string {
    return fmt.Sprintf("%g°C", c)
}

func (f Fahrenheit) String() string {
    return fmt.Sprintf("%g°F", f)
}

func (k Kelvin) String() string {
    return fmt.Sprintf("%g°K", k)
}

func CToF(c Celsius) Fahrenheit {
    return Fahrenheit(c*9/5 + 32)
}

func FToC(f Fahrenheit) Celsius {
    return Celsius((f - 32) * 5 / 9)
}

func KToC(k Kelvin) Celsius {
    return Celsius(k) - AbsoluteZeroC
}

func CToK(c Celsius) Kelvin {
    return Kelvin(c + AbsoluteZeroC)
}

func FToK(f Fahrenheit) Kelvin {
    return kelvin(KToC(f) + AbsoluteZeroC)
}

func KToF(k Kelvin) Fahrenheit {
    return CTok(Celsius(k) - AbsoluteZeroC)
}
```

### Imports

示例代码是一个调用tempconv包中转换函数的程序，课后练习如下

> 练习 2.2： 写一个通用的单位转换程序，用类似cf程序的方式从命令行读取参数，如果缺省的话则是从标准输入读取参数，然后做类似Celsius和Fahrenheit的单位转换，长度单位可以对应英尺和米，重量单位可以对应磅和公斤等。

思路

这里以英尺和米的转换为例

```go
type Meter float64
type Foot float64

func (m Meter) String() string { return fmt.Sprintf("%fm", m) }
func (f Foot) String() string { return fmt.Sprintf("%fft", f) }

func (m Meter) MToF() Foot {
    return Foot(m / 0.3084)
}

func (f Foot) FToM() Meter {
    return Meter(f * 0.3084)
}
```

### Package Initialization

示例代码是一个统计二进制数学中bit为1个数的程序，其中使用init方法初始化程序，课后练习如下。

> **练习 2.3**： 重写PopCount函数，用一个循环代替单一的表达式。比较两个版本的性能。（11.4节将展示如何系统地比较两个不同实现的性能。）
>
> **练习 2.4**： 用移位算法重写PopCount函数，每次测试最右边的1bit，然后统计总数。比较和查表算法的性能差异。
>
> **练习 2.5**： 表达式x&(x-1)用于将x的最低的一个非零的bit位清零。使用这个算法重写PopCount函数，然后比较性能。

思路

用`for`循环代替原函数的暴力相加即可。使用移位算法，通过`x&1`计算最低位是否为1，然后执行`x>>1`移位，依次统计为1的次数即可。使用表达式`x&(x-1)`可以将x的最低的一个非零`bit`位清零，因此只需统计执行次数即可。至于性能判断，通过`time`包下的`Now()`和`Since()`方法比较不同函数执行时间即可。

```go
func PopCount2(x uint64) int {
    var res int
    for i := 1; i < 8; i++ {
        res += int(pc[byte(x>>(i*8))])
    }
    return res
}

// count each bit
func PopCount3(x uint64) int {
    var res int
    for x > 0 {
        if x&1 == 1 {
            res++
        }
        x >>= 1
    }
    return res
}
// x & (x - 1)
func PopCount4(x uint64) int {
    var res int
    for x > 0 {
        x &= (x - 1)
        res++
    }
    return res
}
```
