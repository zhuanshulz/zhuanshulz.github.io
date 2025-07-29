---
title: spec2017在Gem5的SE模式下的运行错误问题
date: 2025-06-25 14:56:14
tags: [Gem5]
---

510.parest在Gem5模拟器的SE模式下运行错误问题
<!-- more -->

<!-- --- -->

# 问题介绍

在Gem5模拟器的SE模式下运行510.parest程序会出现如下报错并中断。

```
An error occurred in line <377> of file <source/base/utilities.cc> in function The violated condition was: 

cpuinfo 

The name and call sequence of the exception was: 

ExcIO()
```

# 问题分析和定位

根据报错信息找到source/base/utilities.cc:377的位置：

```C++
double get_cpu_load ()
{
    std::ifstream cpuinfo;
    cpuinfo.open("/proc/loadavg");

    AssertThrow(cpuinfo, ExcIO());

    double load;
    cpuinfo >> load;

    return load;
}
```

可以看出，该函数是在打开/proc/loadavg文件，获取系统的负载情况数据。如果打开文件错误，则会抛出断言错误。

考虑在Gem5模式的SE模式下，模拟环境并没有操作系统，因此并不能读取/proc/loadavg这一个LINUX操作系统文件，故发生断言异常。

# 解决方法

一个简单直接的办法是直接修改这一个函数，将读取文件的操作跳过，然后重新编译。

如果大家有更加好的解决办法，非常欢迎与我交流。

```c++
double get_cpu_load ()
{
    // std::ifstream cpuinfo;
    // cpuinfo.open("/proc/loadavg");

    // AssertThrow(cpuinfo, ExcIO());

    // double load;
    // cpuinfo >> load;

    // return load;
    return 1.0;
}
```

在修改了源代码后，还需要在.cfg配置文件中设置跳过验证，否则编译不会通过。

```cfg
strict_rundir_verify = 0
```


[参考链接-github.io](https://tomsworkspace.github.io/2020/10/11/gem5%E8%BF%90%E8%A1%8CSPECCPU2017benchmark/)