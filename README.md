已提交pr此仓库作废

[TDengine 官方地址：https://www.taosdata.com](https://www.taosdata.com)

## 基于TDengine-ver-1.6.4.4在windows 10下cmake+msys2编译(windows cgo 使用)
## cgo 使用
**release页面下载编译好的库**   
我们只需要关注 **libtaos.dll** 和 **libtaos.dll.a**  
go 文件使用：
~~~go
/*
#cgo CFLAGS : -I(taos.h文件文件夹)
#cgo LDFLAGS: (libtaos.dll.a文件位置)
*/
~~~
例如：
~~~go
/*
#cgo CFLAGS : -IE:/software/msys64/usr/local/include
#cgo LDFLAGS: E:/software/msys64/usr/local/lib/libtaos.dll.a
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <taos.h>
*/
import "C"
~~~

## 运行时
程序运行时将 **libtaos.dll** 放到程序同级目录，需要保证服务端版本也是1.6.4.4。若版本不同请使用官方动态库改名使用(见注)。  
**注：** 由于使用纯c编写使用vc编译器编译的动态库可以由 gcc 调用，所以官方提供的 **taos.dll** 可以改名为 **libtaos.dll** 使用。

## 背景
TDengine 提供的 go 连接器使用的是 cgo 且只能在 liunx 系统下使用，windows上的动态库是由vc编译器编译而成，cgo 无法使用，本文提供 windows 下用 gcc 编译器编译 TDengine 的步骤和本人编译后的成品。  
**重中之重！交叉编译后的动态库并不保证质量！生产环境 慎用！慎用！慎用！生产环境使用官方提供的 **taos.dll** 改名为 **libtaos.dll** 使用**

## 安装部署 msys2
### 安装
[https://mirror.tuna.tsinghua.edu.cn/help/msys2/](https://mirror.tuna.tsinghua.edu.cn/help/msys2/)
安装完后打开 c:\msys64\msys2_shell.cmd 在窗口上右击， 选择 Options ，更改字符集 zh_cn gbk  
修改 pacman 配置见上面网页 （有梯子可以省略这步）
修改后执行

```bash
pacman -S make
pacman -S mingw-w64-x86_64-gcc
 ```
### 配置环境变量
**C:\msys64\mingw64\bin
C:\msys64\usr\bin**
按以上顺序添加到系统变量 path
## 安装cmake:
 [https://cmake.org/download/](https://cmake.org/download/)
 默认安装
## 下载 TDengine
 [https://github.com/taosdata/TDengine/archive/ver-1.6.4.4.zip](https://github.com/taosdata/TDengine/archive/ver-1.6.4.4.zip)
 
## 修改说明
### CMakeLists.txt 
注释或删除以下行:
~~~ bash
    SET(COMMON_FLAGS "/nologo /WX- /Oi /Oy- /Gm- /EHsc /MT /GS /Gy /fp:precise /Zc:wchar_t /Zc:forScope /Gd /errorReport:prompt /analyze-")
    SET(DEBUG_FLAGS "/Zi /W3 /GL")
    SET(RELEASE_FLAGS "/W0 /GL")
~~~
### src/client/CMakeLists.txt 
注释或删除以下行:
~~~ bash
SET_TARGET_PROPERTIES(taos PROPERTIES LINK_FLAGS /DEF:${TD_COMMUNITY_DIR}/src/client/src/taos.def)
~~~

### deps/iconv/iconv.c 
修改以下行
~~~c
const struct alias *
aliases2_lookup (register const char *str)
{
  const struct alias * ptr;
  unsigned int count;
  for (ptr = sysdep_aliases, count = sizeof(sysdep_aliases)/sizeof(sysdep_aliases[0]); count > 0; ptr++, count--)
    if (!strcmp(str, stringpool2 + ptr->name))
      return ptr;
  return NULL;
}
~~~
改为
~~~c
// gcc -o0 bug fix 
// see http://git.savannah.gnu.org/gitweb/?p=libiconv.git;a=blobdiff;f=lib/iconv.c;h=31853a7f1c47871221189dbf597473a16d8a8da7;hp=5a1a32597fa3efc5f69624d37a2eb96f308cd241;hb=b29089d8b43abc8fba073da7e6dccaeba56b2b70;hpb=0a04404c90d6a725b8b6bbcd65e10c5fcf5993e9

static const struct alias *
aliases2_lookup (register const char *str)
{
  const struct alias * ptr;
  unsigned int count;
  for (ptr = sysdep_aliases, count = sizeof(sysdep_aliases)/sizeof(sysdep_aliases[0]); count > 0; ptr++, count--)
    if (!strcmp(str, stringpool2 + ptr->name))
      return ptr;
  return NULL;
}
~~~

### /os/windows/inc/os.h
添加头文件
~~~c
#include <WS2tcpip.h>
#include <winbase.h>
~~~
修改1:
~~~c
// #define str2int64 _atoi64
int64_t str2int64(char *str);
~~~
修改2:
~~~c
// #define atomic_val_compare_exchange_8(ptr, oldval, newval) _InterlockedCompareExchange8((char volatile*)(ptr), (char)(newval), (char)(oldval))
#define atomic_val_compare_exchange_8 __sync_val_compare_and_swap
~~~
修改3:
~~~c
// #define atomic_fetch_add_8(ptr, val) _InterlockedExchangeAdd8((char volatile*)(ptr), (char)(val))
// #define atomic_fetch_add_16(ptr, val) _InterlockedExchangeAdd16((short volatile*)(ptr), (short)(val))
#define atomic_fetch_add_8 __sync_fetch_and_ad
#define atomic_fetch_add_16 __sync_fetch_and_add
~~~
修改4:
~~~c
// char interlocked_and_fetch_8(char volatile* ptr, char val);
// short interlocked_and_fetch_16(short volatile* ptr, short val);
long interlocked_and_fetch_32(long volatile* ptr, long val);
__int64 interlocked_and_fetch_64(__int64 volatile* ptr, __int64 val);

// #define atomic_and_fetch_8(ptr, val) interlocked_and_fetch_8((char volatile*)(ptr), (char)(val))
// #define atomic_and_fetch_16(ptr, val) interlocked_and_fetch_16((short volatile*)(ptr), (short)(val))
#define atomic_and_fetch_32(ptr, val) interlocked_and_fetch_32((long volatile*)(ptr), (long)(val))
#define atomic_and_fetch_64(ptr, val) interlocked_and_fetch_64((__int64 volatile*)(ptr), (__int64)(val))

~~~
修改5:
~~~c
// #define atomic_fetch_and_8(ptr, val) _InterlockedAnd8((char volatile*)(ptr), (char)(val))
// #define atomic_fetch_and_16(ptr, val) _InterlockedAnd16((short volatile*)(ptr), (short)(val))
#define atomic_fetch_and_32(ptr, val) _InterlockedAnd((long volatile*)(ptr), (long)(val))

~~~
修改6:
~~~c
// char interlocked_or_fetch_8(char volatile* ptr, char val);
// short interlocked_or_fetch_16(short volatile* ptr, short val);
long interlocked_or_fetch_32(long volatile* ptr, long val);
__int64 interlocked_or_fetch_64(__int64 volatile* ptr, __int64 val);

// #define atomic_or_fetch_8(ptr, val) interlocked_or_fetch_8((char volatile*)(ptr), (char)(val))
// #define atomic_or_fetch_16(ptr, val) interlocked_or_fetch_16((short volatile*)(ptr), (short)(val))
#define atomic_or_fetch_32(ptr, val) interlocked_or_fetch_32((long volatile*)(ptr), (long)(val))
#define atomic_or_fetch_64(ptr, val) interlocked_or_fetch_64((__int64 volatile*)(ptr), (__int64)(val))

~~~
修改7:
~~~c
// #define atomic_fetch_or_8(ptr, val) _InterlockedOr8((char volatile*)(ptr), (char)(val))
// #define atomic_fetch_or_16(ptr, val) _InterlockedOr16((short volatile*)(ptr), (short)(val))
#define atomic_fetch_or_32(ptr, val) _InterlockedOr((long volatile*)(ptr), (long)(val))


~~~
修改8:
~~~c
// char interlocked_xor_fetch_8(char volatile* ptr, char val);
// short interlocked_xor_fetch_16(short volatile* ptr, short val);
long interlocked_xor_fetch_32(long volatile* ptr, long val);
__int64 interlocked_xor_fetch_64(__int64 volatile* ptr, __int64 val);

// #define atomic_xor_fetch_8(ptr, val) interlocked_xor_fetch_8((char volatile*)(ptr), (char)(val))
// #define atomic_xor_fetch_16(ptr, val) interlocked_xor_fetch_16((short volatile*)(ptr), (short)(val))
#define atomic_xor_fetch_32(ptr, val) interlocked_xor_fetch_32((long volatile*)(ptr), (long)(val))
#define atomic_xor_fetch_64(ptr, val) interlocked_xor_fetch_64((__int64 volatile*)(ptr), (__int64)(val))

~~~
修改9:
~~~c
// #define atomic_fetch_xor_8(ptr, val) _InterlockedXor8((char volatile*)(ptr), (char)(val))
// #define atomic_fetch_xor_16(ptr, val) _InterlockedXor16((short volatile*)(ptr), (short)(val))
#define atomic_fetch_xor_32(ptr, val) _InterlockedXor((long volatile*)(ptr), (long)(val))
~~~
修改10:
~~~c
// #define MILLISECOND_PER_SECOND (1000i64)
#define MILLISECOND_PER_SECOND (1000LL)
~~~

### src/os/windows/src/twindows.c

添加头:
~~~c
#include <intrin.h>
#include <winbase.h>
#include <Winsock2.h>
~~~

修改1:
~~~c
// add
// char interlocked_add_fetch_8(char volatile* ptr, char val) {
//   return _InterlockedExchangeAdd8(ptr, val) + val;
// }

// short interlocked_add_fetch_16(short volatile* ptr, short val) {
//   return _InterlockedExchangeAdd16(ptr, val) + val;
// }

char interlocked_add_fetch_8(char volatile* ptr, char val) {
  return __sync_fetch_and_add(ptr, val) + val;
}

short interlocked_add_fetch_16(short volatile* ptr, short val) {
  return __sync_fetch_and_add(ptr, val) + val;
}
~~~
修改2:
~~~c
// and
// char interlocked_and_fetch_8(char volatile* ptr, char val) {
//   return _InterlockedAnd8(ptr, val) & val;
// }

// short interlocked_and_fetch_16(short volatile* ptr, short val) {
//   return _InterlockedAnd16(ptr, val) & val;
// }
~~~
修改3:
~~~c
// or
// char interlocked_or_fetch_8(char volatile* ptr, char val) {
//   return _InterlockedOr8(ptr, val) | val;
// }

// short interlocked_or_fetch_16(short volatile* ptr, short val) {
//   return _InterlockedOr16(ptr, val) | val;
// }
~~~
修改4:
~~~c
// xor
// char interlocked_xor_fetch_8(char volatile* ptr, char val) {
//   return _InterlockedXor8(ptr, val) ^ val;
// }

// short interlocked_xor_fetch_16(short volatile* ptr, short val) {
//   return _InterlockedXor16(ptr, val) ^ val;
// }
~~~
修改5:
添加以下代码
~~~c
int64_t str2int64(char *str) {
  char *endptr = NULL;
  return strtoll(str, &endptr, 10);
}

uint64_t htonll(uint64_t val)
{
    return (((uint64_t) htonl(val)) << 32) + htonl(val >> 32);
}
~~~

### src/inc/taos.h
将要导出的函数添加
~~~c
__declspec(dllexport)
~~~
例如：
~~~c
__declspec(dllexport) void  taos_init();
~~~
需要导出的函数见 src/client/src/taos.def
## cmake 转换
Configure 选择 **MSYSMakefiles**

Generator
## make
进到转换后的目录执行 make
成功之后在编译目录的 **build/bin** 文件夹可以看到生成 **taos.exe**  
**build/lib** 下存在以下文件

 - libiconv.a
 - libos.a
 - libpthread.a
 - libregex.a
 - libtaos.dll
 - libtaos.dll.a
 - libtals_static.a
 - libtrpc.a
 - libutil.a
