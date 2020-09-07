---
title: Linux C 动态链接库的生成与使用
categories:
  - 学习笔记
  - C\C++
tags:
  - Linux
  - C\C++
abbrlink: 62d7ab61
date: 2020-09-07 12:38:35
---

## 前言
动态链接库（ `so` 的全称为 Shared Object，因此也称共享库）在 Linux 下用 C\C++ 经常碰到。当多个程序使用同一个动态链接库时，既能将代码复用，又能节约可执行文件的大小，而且还能减少运行时的内存占用。不仅如此，除了能给 C\C++ 调用，动态链接库还能给其他编程语言调用，比如 Python，简直完美。

<!-- more -->

## 编写动态链接库源文件

首先来编写动态链接库源文件。新建 `myfunc.c`，并添加两个函数，一个是 `say_hello` ，另一个是 `cal_sum`

```C
#include "myfunc.h"

void say_hello()
{
    printf("hello world\n");
}


int cal_sum(int x, int y)
{
    return x + y;
}
```

为 `myfunc.c` 编写头文件 `myfunc.h`

```C
#ifndef __MYFUNC_H
#define __MYFUNC_H

#include <stdio.h>
#include <stdlib.h>

void say_hello();
int cal_sum(int x, int y);

#endif
```


## 编译生成动态链接库

编译 `myfunc.c` ：

```shell
gcc -c -fPIC -o myfunc.o myfunc.c
```

参数解析：
> `-c` 表示只编译 ( compile ) ，而不链接，输出目标文（ obj 文件）。
> `-o ` 表示输出文件的文件名。
> `-fPIC` PIC 指 Position Independent Code ， 生成适合在共享库中使用的与位置无关的代码。编译成共享库要求此选项。适用于动态链接并避免对全局偏移表大小的任何限制。

生成动态链接库文件 `libmyfunc.so` ：

```shell
gcc -shared myfunc.o -o libmyfunc.so
```

参数解析：
> `-share` 生成一个共享对象，可以与其他对象链接以形成可执行文件。

**把上面两条命令合成一条就是：**

```shell
gcc -fPIC -shared myfunc.c -o libmyfunc.so
```

**其实动态链接库有其命名规则**。举例 ` libname.so.x.y.z` ：
* `lib` ：**统一前缀**；
* `name` ：**动态链接库的名字**， `libmyfunc.so` 中 `myfunc` 就是其动态链接库名；
* `so` ：**统一后缀**；
* `x` ：**主版本号**，表示该库有重大更新，不同版本号之间是不兼容的；
* `y` ：**次版本号**，表示该库增量升级，在相同主版本号下，次版本号高的兼容次版本号低的库；
* `z` ：**发布版本号**，表示该库的优化、 bugfix 等，相同主次版本号，不同发布版本号之间完全兼容。

好了，上面的步骤就是如何编译生成动态链接库。下面我们继续来说说如何使用动态链接库。


## 使用动态链接库
接下来我们使用 `test.c` 来使用动态链接库。 `test.c` 内容如下：

```C
#include "myfunc.h"

int main(int argc, char const *argv[])
{
    int result = 0;

    say_hello();
    result = cal_sum(2, 3);
    printf("%d\n", result);

    return 0;
}
```

编译命令
```shell
gcc -o test test.c -l myfunc -L .
```

**编译上述包含 `.h` 头文件的程序，GCC编译器需要知道头文件的位置**
* 对于 `#include <...>` ，GCC编译器会在默认 include 搜索路径中寻找。
* 对于 `#include "..."` ，GCC编译器会在当前路径搜索 `.h` 文件。当然你也可以使用 `-I` 选项提供额外的搜索路径，比如 `-I /home/test/` 。

**除此之外，GCC编译器还需要知道我们用了哪个动态链接库文件，库文件在哪里**
* 使用 `-l` 选项说明库文件的名字。这里，我们使用的是 `libmyfunc.so` 库文件，所以选项是这样写的： `-l myfunc` 。
* 使用 `-L` 选项说明库文件的路径。这里，我们的库文件是在当前路径，所以选项是这样写的： `-L .` ，其中 `.` 表示当前路径。

**附加：**
我们可以使用下面的命令来获知系统的 include 默认搜索路径:

```shell
$ gcc -print-prog-name=cc1 -v
```

获知库默认搜索路径:

```shell
$ gcc -print-search-dirs
```


## 执行程序

```shell
$ ./test
```

执行程序后出现这样的错误情况：
```
./test: error while loading shared libraries: libmyfunc.so: cannot open shared object file: No such file or directory
```

**这是因为执行程序的时候，系统不知道 `libmyfunc.so` 的位置，系统无法找到库文件**。尽管我们用 GCC 编译的时候，通过 `-L` 选项提供了 `libmyfunc.so` 文件的位置。**但是这个信息没有被写入到可执行程序里面**。下面用命令 `ldd` 命令来查看一下 `test` 所依赖的库文件（ `ldd` 命令是用于显示可执行文件所依赖的库）：

```shell
$ ldd test

    linux-vdso.so.1 =>  (0x00007ffccc9fe000)
    libmyfunc.so => not found
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f0d31a44000)
    /lib64/ld-linux-x86-64.so.2 (0x00007f0d31e0e000)
```

**可以看出可执行文件 `test` 无法找到 `libmyfunc.so` 库文件。**

**解决办法有 4 个：**

1. **将 `libmyfunc.so` 放到 GCC 默认搜索目录。**比如 `/usr/lib/x86_64-linux-gnu` 或者 `/lib/x86_64-linux-gnu` 都可以，这样做简单粗暴。但是这样做需要 root 权限来完成。而且把第三方库文件直接放到系统库文件目录里，我感觉污染了整个系统（洁癖症）。

2. **在 `/etc/ld.so.conf.d` 目录下新建一个 `.conf` 文件。**比如 `mylib.conf` ，在里面添加第三方库目录的绝对路径（比如 `libmyfunc.so` 所在目录的绝对路径）。

3. **设置 `LD_LIBRARY_PATH` 环境变量。**比如 `export LD_LIBRARY_PATH=.` （ `.` 代表当前目录）。当设置这个环境变量后，操作系统将在 `LD_LIBRARY_PATH` 下搜索库文件，再到默认路径中搜索文件。
但是一旦退出 Terminal ，所设置的 `LD_LIBRARY_PATH` 环境变量就会消失。如果需要永久添加变量，需要将 `export LD_LIBRARY_PATH=/xxx/xxx:$LD_LIBRARY_PATH` 写入到 `~/.bashrc` 里面，其中 `/xxx/xxx` 是库文件所在目录的绝对路径。

4. **编译的时候添加 `-Wl,-rpath` 选项。**比如 `gcc -o test test.c -l myfunc -L . -Wl,-rpath=.` 。 `-Wl` 选项告诉编译器将后面的参数传递给链接器


**重新编译和运行 `test`**
```shell
$ gcc -o test test.c -l myfunc -L . -Wl,-rpath=.
$ ./test
hello world
5
```

## 参考资料

1. [C 编译: 动态连接库 ( .so 文件)](http://www.cnblogs.com/vamei/archive/2013/04/04/2998850.html)
2. [Linux 动态库生成与使用指南](https://www.cnblogs.com/jiqingwu/p/linux_dynamic_lib_create.html)
3. [Linux 动态库剖析](https://www.ibm.com/developerworks/cn/linux/l-dynamic-libraries/index.html)
4. [Linux 动态链接](https://www.jianshu.com/p/ea9f4a6b136d)
5. [运行时动态库：not found 及介绍 linux 的 -Wl,-rpath 命令](https://www.cnblogs.com/homejim/p/8004883.html)
