---
title: CUDA 12.4与glibc 2.41头文件冲突解决全记录
date: 2025-04-11 09:48:08
tags: [CUDA, nvcc, glibc]
---

## 碎碎念

很久没有写`CUDA`，打算重新拾起，找到了之前写的代码，编译一下看看，结果直接error了
```
/usr/include/bits/mathcalls.h(79): error: exception specification is incompatible with that of previous function "cospi" (declared at line 5554 of /usr/local/cuda-12.4/bin/../targets/x86_64-linux/include/crt/math_functions.h)
   extern double cospi (double __x) noexcept (true); extern double __cospi (double __x) noexcept (true);
                                    ^

/usr/include/bits/mathcalls.h(81): error: exception specification is incompatible with that of previous function "sinpi" (declared at line 5442 of /usr/local/cuda-12.4/bin/../targets/x86_64-linux/include/crt/math_functions.h)
   extern double sinpi (double __x) noexcept (true); extern double __sinpi (double __x) noexcept (true);
                                    ^

/usr/include/bits/mathcalls.h(79): error: exception specification is incompatible with that of previous function "cospif" (declared at line 5606 of /usr/local/cuda-12.4/bin/../targets/x86_64-linux/include/crt/math_functions.h)
   extern float cospif (float __x) noexcept (true); extern float __cospif (float __x) noexcept (true);
                                   ^

/usr/include/bits/mathcalls.h(81): error: exception specification is incompatible with that of previous function "sinpif" (declared at line 5502 of /usr/local/cuda-12.4/bin/../targets/x86_64-linux/include/crt/math_functions.h)
   extern float sinpif (float __x) noexcept (true); extern float __sinpif (float __x) noexcept (true);
```
还以为是`gcc`或者`g++`编译器不对导致的，但实际上我尝试了强制指定版本，并且尝试了删除系统用的高版本`gcc`和`g++`，
问题依然存在。

> 我这里系统用的是14的编译器版本，而`CUDA` 12不支持这么高的版本，我另外安装了12的编译器

开始出现这个问题的时候，比较急，因为这个报错甚至都没有提是我的代码哪里写错了，也就是报错的地方和我写的代码可能没关系。
后面一想，还真没关系，毕竟这其实就是头文件定义的函数冲突了，这两个地方的代码分别是：
```cpp
// /usr/local/cuda-12.4/bin/../targets/x86_64-linux/include/crt/math_functions.h(5554)
extern __DEVICE_FUNCTIONS_DECL__ __device_builtin__ double                 cospi(double x);
```

```cpp
// /usr/include/bits/mathcalls.h(79)
__MATHCALL_VEC (cospi,, (_Mdouble_ __x));
```

这俩声明确实不一样，然后给冲突了。

后来我搜了相关的报错，找到一个[比较关键的帖子](https://forums.developer.nvidia.com/t/error-exception-specification-is-incompatible-for-cospi-sinpi-cospif-sinpif-with-glibc-2-41/323591)，
从这里知道了原因：`CUDA`暂时不支持`glibc 2.41`，同时我也知道了一个简单的解决办法，就是给`CUDA`中这几个冲突的函数定义，
添加`noexcept (true)`。

> 不过我最开始没有注意到添加`noexcept (true)`的方法，只是看到`glibc`版本问题，就往使用低版本`glibc`的方向去想了。

## 问题复现

### 环境

- 系统：`openSUSE Tumbleweed x86_64`

- `CUDA`版本：`V12.4.131`

- `Nvidia`驱动版本：`570.133.07`

- `glibc`版本：`2.41`

### MWE

```cpp
// mwe.cu
#include<cuda_runtime.h>

int main()
{
    return 0;
}
```

### 报错

在这个环境下，编译`mwe.cu`
```shell
# nvcc mwe.cu 

/usr/include/bits/mathcalls.h(79): error: exception specification is incompatible with that of previous function "cospi" (declared at line 5554 of /usr/local/cuda-12.4/bin/../targets/x86_64-linux/include/crt/math_functions.h)
   extern double cospi (double __x) noexcept (true); extern double __cospi (double __x) noexcept (true);
                                    ^

/usr/include/bits/mathcalls.h(81): error: exception specification is incompatible with that of previous function "sinpi" (declared at line 5442 of /usr/local/cuda-12.4/bin/../targets/x86_64-linux/include/crt/math_functions.h)
   extern double sinpi (double __x) noexcept (true); extern double __sinpi (double __x) noexcept (true);
                                    ^

/usr/include/bits/mathcalls.h(79): error: exception specification is incompatible with that of previous function "cospif" (declared at line 5606 of /usr/local/cuda-12.4/bin/../targets/x86_64-linux/include/crt/math_functions.h)
   extern float cospif (float __x) noexcept (true); extern float __cospif (float __x) noexcept (true);
                                   ^

/usr/include/bits/mathcalls.h(81): error: exception specification is incompatible with that of previous function "sinpif" (declared at line 5502 of /usr/local/cuda-12.4/bin/../targets/x86_64-linux/include/crt/math_functions.h)
   extern float sinpif (float __x) noexcept (true); extern float __sinpif (float __x) noexcept (true);
                                   ^

4 errors detected in the compilation of "mwe.cu".
```

## 修复

除了去改`CUDA`中的代码，我最开始想到的就是，我之前就能正常编译，但是系统更新之后，就编译不了了，那我想办法让它回到之前能编译的状态就行。
直接降系统的`glibc`肯定是不行的，但我装一个低版本的`glibc`并且在编译的时候，指定使用低版本的，而不是系统的`glibc`，应该就可以。

### 安装低版本`glibc`

我这里下载的是[2.40版本的`glibc`](https://ftp.gnu.org/gnu/glibc/glibc-2.40.tar.gz)，然后解压、编译、安装
```shell
tar -xvzf glibc-2.40.tar.gz
cd glibc-2.40
mkdir build
cd build
mkdir $HOME/glibc-2.40
../configure --prefix=$HOME/glibc-2.40
make
make install
```

### 设置`g++`使用低版本`glibc`

我创建了文件`$HOME/bin/g++-12-glibc-2.40`（这个路径是在`$PATH`环境变量中的），最终是选择了如下这样一个wrapper

```shell
#!/bin/sh
GLIBC_PATH="$HOME/glibc-2.40"
GLIBC_INCLUDE="$GLIBC_PATH/include"
GLIBC_LIB="$GLIBC_PATH/lib"

exec /usr/bin/g++-12 -I"$GLIBC_INCLUDE" -L"$GLIBC_LIB" "$@"
```

这里实际上生效的可能只是改变了`include`搜索路径的优先级，也就是说优先使用了`glibc-2.40`的头文件，
而不是使用系统中比较新的2.41版本的头文件，那么原则上来说，链接的文件也应当使用`glibc-2.40`的动态库文件，
从我设置的选项来看，似乎是这样子的。
> 但我对动态库文件的链接过程并不理解，我通过一些博客和DeepSeek了解到可以添加`-Wl,-rpath=$GLIBC_LIB`
这种选项去控制运行时搜索动态链接库的路径，但我尝试设置这个选项后，会导致编译的文件只在我设置的这个路径中搜索动态库，
找不到其他的动态库，就会`segmentation fault (core dumped)`。
使用这个选项的时候我通过`patchelf --print-rpath`查找到的路径确实只有我设置的路径，而当我不设置这个选项的时候，
查找到的路径是空的。一个合理的推测是，程序如果设置了`rpath`可能搜索的路径就会限制在这个路径中，如果不设置就会受其他变量影响。
我期望的肯定是添加一个path，让程序运行的时候优先搜索`glibc-2.40`的动态库文件，但可惜暂时不知道怎么做才能达到。
暂时来说我使用低版本的头文件能通过编译，链接的时候可能链接的是低版本的库文件，但是运行的时候还是找的高版本库文件，只要我程序中没有调用到那种在高版本没有实现的函数，应该也不会出问题。

### 设置`nvcc`使用`g++-12-glibc-2.40`

通过[`CUDA`官方文档](https://docs.nvidia.com/cuda/cuda-compiler-driver-nvcc/index.html#nvcc-environment-variables)，
一个可行的方案是，设置这个环境变量

```shell
export NVCC_PREPEND_FLAGS="--compiler-bindir=g++-12-glibc-2.40"
```
这样使用`nvcc`进行编译的时候，就会自动选择我设置好的`g++`。

> 文档中提到的设置`NVCC_CCBIN`这个变量我也尝试了，但是这个变量似乎不生效，即使我设置了，也不会使用环境变量中指定的编译器，而是会使用`/usr/bin`下的编译器。