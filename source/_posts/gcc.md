---
title: gcc
date: 2017-10-14 20:56:04
tags:
- gcc
---
#### \#include <>与\#inlucde ""
\#include <> 会直接到系统指定的目录中去找头文件。
\#include “” 先到源文件所在文件夹去找，然后再到系统指定的目录中去找头文件。
#### gcc指定头文件的三种情况：
1. 会在默认情况下指定到/usr/include文件夹
2. gcc使用了-I指定路径的方式
-I dir
    Add the directory dir to the list of directories to be searched for
    header files.  Directories named by -I are searched before the
    standard system include directories.  If the directory dir is a
    standard system include directory, the option is ignored to ensure
    that the default search order for system directories and the
    special treatment of system headers are not defeated .  If dir
    begins with "=", then the "=" will be replaced by the sysroot
    prefix; see --sysroot and -isysroot.
3. 参数：-nostdinc使编译器不再系统缺省的头文件目录里面找头文件,一般和-I联合使用,明确限定头文件的位置。

注：参数：-nostdinc使编译器不再系统缺省的头文件目录里面找头文件,一般和-I联合使用,明确限定头文件的位置。
#### 头文件搜索顺序：
1. 由参数-I指定的路径(指定路径有多个路径时，按指定路径的顺序搜索)
2. 然后找gcc的环境变量 C_INCLUDE_PATH, CPLUS_INCLUDE_PATH, OBJC_INCLUDE_PATH
3. 再找系统指定目录
/usr/include
/usr/local/include
/usr/lib/gcc/x86_64-linux-gnu/5.4.0/include
#### 动态库
我们通过以下命令用源程序lib_test.c来创建动态库 lib_test.so。
\# gcc –o lib_test.o -c lib_test.c
\# gcc -shared -fPIC -o lib_test.so lib_test.o
\#
或者直接一条指令：
\#gcc –shared –fPIC –o lib_test.so lib_test.c
\#
注意：
-fPIC参数声明链接库的代码段是可以共享的，
-shared参数声明编译为共享库。请注意这次我们编译的共享库的名字叫做
lib_test.so，这也是Linux共享库的一个命名的惯例了：后缀使用so，而名称使用libxxxx格式。

接着通过以下命令编译main.c,生成目标程序main.out。
\# gcc -o main.out -L. –l_test main.c
\#

请注意为什么是-l_test?

然后把库文件移动到目录/root/lib中。
\# mkdir /root/lib
\# mv lib_test.so /root/lib/ lib_test.so
\#
