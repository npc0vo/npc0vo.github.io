---
title: 初识Pyd逆向
tags:
  - "#blog"
  - "#pyd"
description:
date: 2025-01-13
aliases:
draft:
---
[https://tenthousandmeters.com/blog/python-behind-the-scenes-8-how-python-integers-work/](https://tenthousandmeters.com/blog/python-behind-the-scenes-8-how-python-integers-work/)

[https://www.cnblogs.com/lordtianqiyi/p/18614184](https://www.cnblogs.com/lordtianqiyi/p/18614184)

[https://xz.aliyun.com/t/16155?time__1311=GuD%3D7Iqmxfhx%2FD0lD2DUgDj2RjGCK%2Bj3WfeD](https://xz.aliyun.com/t/16155?time__1311=GuD%3D7Iqmxfhx%2FD0lD2DUgDj2RjGCK%2Bj3WfeD)

[https://blog.hxzzz.asia/](https://blog.hxzzz.asia/)

这次国赛出了2个cython，发现我都不会，太菜了

赛完学习一波

## Basic

>>> Cython = 一个可以将 Python 转化为 C 的编译器，也是一种语言（Python 的超集）

>>> Python = 一种众所周知的编程语言，可以让你快速工作并更有效地集成系统。 >>> CPython = Python 解释器的源码，并给用户提供 Python-C API 支持以编写 C 扩展

python 源文件(.py) 经过解释器 得到(.pyc)字节码文件，而字节码 经过 python的vm运行 得到的pyd文件是运行在机器的机器码文件

### PY_LONG的解析

在逆向pyd的时候，由于python自行实现了一套大数运算系统，所以在Number的解析中，只看内存很难找到对应具体数值(当时卡了一天....）

这里直接去看cython源码

```
struct _object {
    #if (defined(__GNUC__) || defined(__clang__)) \
        && !(defined __STDC_VERSION__ && __STDC_VERSION__ >= 201112L)
    // On C99 and older, anonymous union is a GCC and clang extension
    __extension__
        #endif
        #ifdef _MSC_VER
        // Ignore MSC warning C4201: "nonstandard extension used:
        // nameless struct/union"
        __pragma(warning(push))
        __pragma(warning(disable: 4201))
        #endif
        union {
            #if SIZEOF_VOID_P > 4
            PY_INT64_T ob_refcnt_full; /* This field is needed for efficient initialization with Clang on ARM */
            struct {
                #  if PY_BIG_ENDIAN
                uint16_t ob_flags;
                uint16_t ob_overflow;
                uint32_t ob_refcnt;
                #  else
                uint32_t ob_refcnt;
                uint16_t ob_overflow;
                uint16_t ob_flags;
                #  endif
            };
            #else
            Py_ssize_t ob_refcnt;
            #endif
        };
    #ifdef _MSC_VER
    __pragma(warning(pop))
        #endif

        PyTypeObject *ob_type;
};
```

```
/*  =========> python12  PyLongObject structure <========
#define _PyLong_NON_SIZE_BITS 3 
The number of digits (ndigits) is stored in the high bits of
   the lv_tag field (lvtag >> _PyLong_NON_SIZE_BITS). 


The sign of the value is stored in the lower 2 bits of lv_tag.
- 0: Positive
- 1: Zero
- 2: Negative
The third lowest bit of lv_tag is reserved for an immortality flag, but is
not currently used.



struct _object {
    Py_ssize_t ob_refcnt;
    PyTypeObject *ob_type;
};

typedef struct _PyLongValue {
    uintptr_t lv_tag;  //Number of digits, sign and flags //  
    digit ob_digit[1];
} _PyLongValue;
typedef struct _object PyObject;
#define PyObject_HEAD                   PyObject ob_base;
struct _longobject {
    PyObject_HEAD //_object
    _PyLongValue long_value;
};

typedef struct _longobject PyLongObject;
*/
```

[Python behind the scenes #8: how Python integers work](https://tenthousandmeters.com/blog/python-behind-the-scenes-8-how-python-integers-work/)

以我的电脑小端序64位来举例

![[b608ec82961da554f2de93e6e3ecb288_MD5.png]]

这里的每个python_number都是30bits的,剩下的2个bits是lv_tag

(val&0xffffffff) |(val>>32)<<30 才能正确解析出数值

### cython逆向基础

[系统讲解pyd逆向/Cython逆向_pyd的逆向-CSDN博客](https://blog.csdn.net/qq_36791177/article/details/144567790)

### 常见的题型

- 源代码经过混淆得到的py逆向,or只提供字节码的文本
- .pyc字节码逆向 pyc反编译 (以及常见的花指令和魔改)
- pyinstaller打包 解包 pyc反编译
- cython编译的拓展(.pyc/so) # 难度最大 手撕+动态

## 方法总结

### 注入类

可以列出所有的导入库

```
import pefile

def import_check(pyd_file):
    pe = pefile.PE(pyd_file)
    for entry in pe.DIRECTORY_ENTRY_IMPORT:
        print(f"Module: {entry.dll.decode('utf-8')}")
        for imp in entry.imports:
            print(f"  Function: {imp.name.decode('utf-8') if imp.name else None}")

import_check('cy.cp38-win_amd64.pyd')
```

根据涉及的运算可以大概猜测一下加密类型

比如只有xor，shift，可以大致猜测是一个tea家族

如果有类的话我们可以注入类

例如2024SCTF的 ez_cython

```
import cy

class Symbol:
  def __init__(self, name):
    self.name = name

  def __repr__(self):
    return self.name

  def __rshift__(self, other):
    if isinstance(other, Symbol):
      expression = Symbol(f"({self.name} >> {other.name})")
    else:
      expression = Symbol(f"({self.name} >> {other})")
    return expression

  def __lshift__(self, other):
    if isinstance(other, Symbol):
      expression = Symbol(f"({self.name} << {other.name})")
    else:
      expression = Symbol(f"({self.name} << {other})")
    return expression

  def __rxor__(self, other):
    if isinstance(other, Symbol):
      expression = Symbol(f"({self.name} ^ {other.name})")
    else:
      expression = Symbol(f"({self.name} ^ {other})")
    return expression

  def __xor__(self, other):
    if isinstance(other, Symbol):
      expression = Symbol(f"({self.name} ^ {other.name})")
    else:
      expression = Symbol(f"({self.name} ^ {other})")
    return expression

  def __add__(self, other):
    if isinstance(other, Symbol):
      expression = Symbol(f"({self.name} + {other.name})")
    else:
      expression = Symbol(f"({self.name} + {other})")
    return expression

  def __and__(self, other):
    if isinstance(other, Symbol):
      expression = Symbol(f"({self.name} & {other.name})")
    else:
      expression = Symbol(f"({self.name} & {other})")
    return expression

class AList:
  def __init__(self, nums):
    self.nums = [Symbol(str(num)) for num in nums]

  def __getitem__(self, key):
    return self.nums[key]

  def copy(self):
    return AList(self.nums)

  def __len__(self):
    return len(self.nums)

  def __setitem__(self, key, value):
    print(f"new_{self.nums[key]} = {value}")
    self.nums[key] = Symbol(f"new_{self.nums[key].name}")

  def __eq__(self, other):
    print(f"{self.nums} == {other}")
    return self.nums == other

inp = AList([f"a[{i}]" for i in range(32)])
res = cy.sub14514(inp)

if __name__ == '__main__':
  print(res)
```

有一点局限性，就是在运算过程中不会出现一些类型转换之类的，否则还是无法hook

### 手写一个pyd(恢复符号)

linux编译一份相同python版本的so文件，ida载入，File->Produce File->Create C Header File导出结构体

加载需要逆向的pyd，File->Load File->Parse C Header File，导入so文件导出xxx.h(有错误就修复)，导入so文件导出的xxx.h是因为错误比较少，windows带调试符号导出的header错误较多比较难修复

手动编译一个相同版本的pyd文件，可以使用bindiff恢复一下符号，静态分析更容易一点

定位__Pyx_CreateStringTabAndInitStrings()，还原__pyx_mstate结构体(python变量名)

定位_Pyx_InitConstants()，还原__pyx_mstate结构中的整数成员

定位到核心函数，动态调试

根据调用Cython api的库函数，重新定义变量类型为导入的结构体，来高效还原python代码

```
def check(flag):
  a = 1 * 1
  b = 2 * 2
  c = 2 ^ 3
  a = [0] * 6
  b = 0
  while b:
    print("11")
    b -= 1
  rand0m(flag) 
  return a + b + c + flag

def myxor(a, b):
  return a ^ b
def rand0m(x):
  return x ^ 123

__test__ = {}

```

```
from setuptools import setup, Extension
from Cython.Build import cythonize

ext_module = [
  Extension(
    name = "test",
    sources=["test.py"],
    extra_compile_args=["/Zi"],
    extra_link_args=["/DEBUG"]
   )
]
 
setup(
  name="test",
  ext_modules=cythonize(ext_module, annotate=True),
)

```

```
python build.py build_ext --inplace
```

这种方法生成的pyd文件在windows下默认是无符号的，在linux下默认是有符号的

不过我们可以指定生成有符号的 pyd文件:

```
python setup.py build_ext --inplace --debug
```

这里需要在安装python3.12版本时勾选download debug binaries

bindiff后找到**initstring**函数

```
typedef struct {
    __int64 *__pyx_0;
    __int64 *__pyx_1;
    __int64 *__pyx_2;
    __int64 *__pyx_3;
    __int64 *__pyx_4;
    __int64 *__pyx_5;
    __int64 *__pyx_6;
    __int64 *__pyx_7;
    __int64 *__pyx_8;
    __int64 *__pyx_9;
    __int64 *__pyx_10;
    __int64 *__pyx_11;
    __int64 *__pyx_12;
    __int64 *__pyx_13;
    __int64 *__pyx_14;
    __int64 *__pyx_15;
    __int64 *__pyx_16;
    __int64 *__pyx_17;
    __int64 *__pyx_18;
    __int64 *__pyx_19;
    __int64 *__pyx_20;
    __int64 *__pyx_21;
    __int64 *__pyx_22;
    __int64 *__pyx_23;
    __int64 *__pyx_24;
    __int64 *__pyx_25;
    __int64 *__pyx_26;
    __int64 *__pyx_27;
    __int64 *__pyx_28;
    __int64 *__pyx_29;
    __int64 *__pyx_30;
    __int64 *__pyx_31;
    __int64 *__pyx_32;
    __int64 *__pyx_33;
    __int64 *__pyx_34;
    __int64 *__pyx_35;
    __int64 *__pyx_36;
    __int64 *__pyx_37;
    __int64 *__pyx_38;
    __int64 *__pyx_39;
} __pyx_mstate_me;
```

然后找到initstring函数对应的名称填充指针名称

然后静态分析可以大大提高效率

### 动态trace

可以利用条件断点对pyd的运算进行插桩，然后分析加密过程

这里对于Lshift、Rshift、And等逻辑操作来说，是hook不到的，由于优化的缘故不走 Py提供的库函数，直接在pyd中自己实现了

当然也可以一条一条的下断点打log，去trace，体力活

这里我就直接动调直接看了

对于Lshift Rshift And等操作不走Py提供的库函数是直接被内联优化掉了

要hook的话就要找到具体的函数

例如lshift 可能会走 long_lshift函数 ![[35a5b70e60cc2d00b666e92788501c73_MD5.png]]

#### 2025ciscn-rand0m

##### idapro的trace

下条件断点trace 基本运算

```
import idc
def calVal(val,size):
    
    if ord(size)==8:
        val=int.from_bytes(val,byteorder='little')
        result=val
    elif ord(size)==16:
        val=int.from_bytes(val,byteorder='little')
        result=(val&0xffffffff) |(val>>32)<<30
    return result

arg2_addr=idc.get_reg_value("rdx")
arg1_addr=idc.get_reg_value("rcx")
arg2_size=idc.get_bytes(arg2_addr+0x10,1)
arg1_size=idc.get_bytes(arg1_addr+0x10,1)
arg2_val=idc.get_bytes(arg2_addr+0x18,ord(arg2_size)//2)
arg1_val=idc.get_bytes(arg1_addr+0x18,ord(arg1_size)//2)
val2=calVal(arg2_val,arg2_size)
val1=calVal(arg1_val,arg1_size)
print(f"{hex(val2|val1)}={hex(val1)}|{hex(val2)}")
```

```
import idc
def calVal(val,size):
    
    if ord(size)==8:
        val=int.from_bytes(val,byteorder='little')
        result=val
    elif ord(size)==16:
        val=int.from_bytes(val,byteorder='little')
        result=(val&0xffffffff) |(val>>32)<<30
    return result

arg2_addr=idc.get_reg_value("rdx")
arg1_addr=idc.get_reg_value("rcx")
arg2_size=idc.get_bytes(arg2_addr+0x10,1)
arg1_size=idc.get_bytes(arg1_addr+0x10,1)
arg2_val=idc.get_bytes(arg2_addr+0x18,ord(arg2_size)//2)
arg1_val=idc.get_bytes(arg1_addr+0x18,ord(arg1_size)//2)
val2=calVal(arg2_val,arg2_size)
val1=calVal(arg1_val,arg1_size)
print(f"{hex(val2&val1)}={hex(val1)}&{hex(val2)}")
```

```
import idc
def calVal(val,size):
    
    if ord(size)==8:
        val=int.from_bytes(val,byteorder='little')
        result=val
    elif ord(size)==16:
        val=int.from_bytes(val,byteorder='little')
        result=(val&0xffffffff) |(val>>32)<<30
    return result

arg2_addr=idc.get_reg_value("rdx")
arg1_addr=idc.get_reg_value("rcx")
arg2_size=idc.get_bytes(arg2_addr+0x10,1)
arg1_size=idc.get_bytes(arg1_addr+0x10,1)
arg2_val=idc.get_bytes(arg2_addr+0x18,ord(arg2_size)//2)
arg1_val=idc.get_bytes(arg1_addr+0x18,ord(arg1_size)//2)
val2=calVal(arg2_val,arg2_size)
val1=calVal(arg1_val,arg1_size)
print(f"{hex(val2%val1)}={hex(val1)}%{hex(val2)}")
    
```

```
import idc
def calVal(val,size):
    
    if ord(size)==8:
        val=int.from_bytes(val,byteorder='little')
        result=val
    elif ord(size)==16:
        val=int.from_bytes(val,byteorder='little')
        result=(val&0xffffffff) |(val>>32)<<30
    return result

arg2_addr=idc.get_reg_value("rdx")
arg1_addr=idc.get_reg_value("rcx")
arg2_size=idc.get_bytes(arg2_addr+0x10,1)
arg1_size=idc.get_bytes(arg1_addr+0x10,1)
arg2_val=idc.get_bytes(arg2_addr+0x18,ord(arg2_size)//2)
arg1_val=idc.get_bytes(arg1_addr+0x18,ord(arg1_size)//2)
val2=calVal(arg2_val,arg2_size)
val1=calVal(arg1_val,arg1_size)
print(f"{hex(val1)}?={hex(val2)}")
```

```
import idc
def calVal(val,size):
    
    if ord(size)==8:
        val=int.from_bytes(val,byteorder='little')
        result=val
    elif ord(size)==16:
        val=int.from_bytes(val,byteorder='little')
        result=(val&0xffffffff) |(val>>32)<<30
    return result

arg2_addr=idc.get_reg_value("rdx")
arg1_addr=idc.get_reg_value("rcx")
arg2_size=idc.get_bytes(arg2_addr+0x10,1)
arg1_size=idc.get_bytes(arg1_addr+0x10,1)
arg2_val=idc.get_bytes(arg2_addr+0x18,ord(arg2_size)//2)
arg1_val=idc.get_bytes(arg1_addr+0x18,ord(arg1_size)//2)
val2=calVal(arg2_val,arg2_size)
val1=calVal(arg1_val,arg1_size)
print(f"{hex(val2^val1)}={hex(val1)}^{hex(val2)}")
```

```
import idc
def calVal(val,size):
    
    if ord(size)==8:
        val=int.from_bytes(val,byteorder='little')
        result=val
    elif ord(size)==16:
        val=int.from_bytes(val,byteorder='little')
        result=(val&0xffffffff) |(val>>32)<<30
    return result

arg2_addr=idc.get_reg_value("rdx")
arg1_addr=idc.get_reg_value("rcx")
arg2_size=idc.get_bytes(arg2_addr+0x10,1)
arg1_size=idc.get_bytes(arg1_addr+0x10,1)
arg2_val=idc.get_bytes(arg2_addr+0x18,ord(arg2_size)//2)
arg1_val=idc.get_bytes(arg1_addr+0x18,ord(arg1_size)//2)
val2=calVal(arg2_val,arg2_size)
val1=calVal(arg1_val,arg1_size)
print(f"{hex(val>>val2)}={hex(val1)}>>{hex(val2)}")
```

```
import idc
def calVal(val,size):
    
    if ord(size)==8:
        val=int.from_bytes(val,byteorder='little')
        result=val
    elif ord(size)==16:
        val=int.from_bytes(val,byteorder='little')
        result=(val&0xffffffff) |(val>>32)<<30
    return result

arg2_addr=idc.get_reg_value("rdx")
arg1_addr=idc.get_reg_value("rcx")
arg2_size=idc.get_bytes(arg2_addr+0x10,1)
arg1_size=idc.get_bytes(arg1_addr+0x10,1)
arg2_val=idc.get_bytes(arg2_addr+0x18,ord(arg2_size)//2)
arg1_val=idc.get_bytes(arg1_addr+0x18,ord(arg1_size)//2)
val2=calVal(arg2_val,arg2_size)
val1=calVal(arg1_val,arg1_size)
print(f"{hex(val<<val2)}={hex(val1)}<<{hex(val2)}")
```

```
import idc
def calVal(val,size):
    
    if ord(size)==8:
        val=int.from_bytes(val,byteorder='little')
        result=val
    elif ord(size)==16:
        val=int.from_bytes(val,byteorder='little')
        result=(val&0xffffffff) |(val>>32)<<30
    return result

arg2_addr=idc.get_reg_value("rdx")
arg1_addr=idc.get_reg_value("rcx")
arg2_size=idc.get_bytes(arg2_addr+0x10,1)
arg1_size=idc.get_bytes(arg1_addr+0x10,1)
arg2_val=idc.get_bytes(arg2_addr+0x18,ord(arg2_size)//2)
arg1_val=idc.get_bytes(arg1_addr+0x18,ord(arg1_size)//2)
val2=calVal(arg2_val,arg2_size)
val1=calVal(arg1_val,arg1_size)
print(f"{hex(val1-val2)}={hex(val1)}-{hex(val2)}")
```

```
import idc
def calVal(val,size):
    
    if ord(size)==8:
        val=int.from_bytes(val,byteorder='little')
        result=val
    elif ord(size)==16:
        val=int.from_bytes(val,byteorder='little')
        result=(val&0xffffffff) |(val>>32)<<30
    return result

arg2_addr=idc.get_reg_value("rdx")
arg1_addr=idc.get_reg_value("rcx")
arg2_size=idc.get_bytes(arg2_addr+0x10,1)
arg1_size=idc.get_bytes(arg1_addr+0x10,1)
arg2_val=idc.get_bytes(arg2_addr+0x18,ord(arg2_size)//2)
arg1_val=idc.get_bytes(arg1_addr+0x18,ord(arg1_size)//2)
val2=calVal(arg2_val,arg2_size)
val1=calVal(arg1_val,arg1_size)
print(f"{hex(val1+val2)}={hex(val1)}+{hex(val2)}")
```

```
import idc
def calVal(val,size):
    
    if ord(size)==8:
        val=int.from_bytes(val,byteorder='little')
        result=val
    elif ord(size)==16:
        val=int.from_bytes(val,byteorder='little')
        result=(val&0xffffffff) |(val>>32)<<30
    return result

arg2_addr=idc.get_reg_value("rdx")
arg1_addr=idc.get_reg_value("rcx")
arg2_size=idc.get_bytes(arg2_addr+0x10,1)
arg1_size=idc.get_bytes(arg1_addr+0x10,1)
arg2_val=idc.get_bytes(arg2_addr+0x18,ord(arg2_size)//2)
arg1_val=idc.get_bytes(arg1_addr+0x18,ord(arg1_size)//2)
val2=calVal(arg2_val,arg2_size)
val1=calVal(arg1_val,arg1_size)
print(f"{hex(val1*val2)}={hex(val1)}*{hex(val2)}")
```

```
import idc
def calVal(val,size):
    
    if ord(size)==8:
        val=int.from_bytes(val,byteorder='little')
        result=val
    elif ord(size)==16:
        val=int.from_bytes(val,byteorder='little')
        result=(val&0xffffffff) |(val>>32)<<30
    return result

arg2_addr=idc.get_reg_value("rdx")
arg1_addr=idc.get_reg_value("rcx")
arg2_size=idc.get_bytes(arg2_addr+0x10,1)
arg1_size=idc.get_bytes(arg1_addr+0x10,1)
arg2_val=idc.get_bytes(arg2_addr+0x18,ord(arg2_size)//2)
arg1_val=idc.get_bytes(arg1_addr+0x18,ord(arg1_size)//2)
val2=calVal(arg2_val,arg2_size)
val1=calVal(arg1_val,arg1_size)
print(f"{hex(val1)}**{hex(val2)}")
```

```
import idc
def calVal(val,size):
    
    if ord(size)==8:
        val=int.from_bytes(val,byteorder='little')
        result=val
    elif ord(size)>=16:
        val=int.from_bytes(val,byteorder='little')
        result=(val&0xffffffff) |(val>>32)<<30
    return result

arg2_addr=idc.get_reg_value("rdx")
arg1_addr=idc.get_reg_value("rcx")
arg2_size=idc.get_bytes(arg2_addr+0x10,1)
arg1_size=idc.get_bytes(arg1_addr+0x10,1)
arg2_val=idc.get_bytes(arg2_addr+0x18,ord(arg2_size)//2)
arg1_val=idc.get_bytes(arg1_addr+0x18,ord(arg1_size)//2)
val2=calVal(arg2_val,arg2_size)
val1=calVal(arg1_val,arg1_size)
print(f"div={hex(val1)}%{hex(val2)}")
```

```
import idc
def calVal(val,size):
    
    if ord(size)==8:
        val=int.from_bytes(val,byteorder='little')
        result=val
    elif ord(size)==16:
        val=int.from_bytes(val,byteorder='little')
        result=(val&0xffffffff) |(val>>32)<<30
    return result

arg2_addr=idc.get_reg_value("rdx")
arg1_addr=idc.get_reg_value("rcx")
arg2_size=idc.get_bytes(arg2_addr+0x10,1)
arg1_size=idc.get_bytes(arg1_addr+0x10,1)
arg2_val=idc.get_bytes(arg2_addr+0x18,ord(arg2_size)//2)
arg1_val=idc.get_bytes(arg1_addr+0x18,ord(arg1_size)//2)
val2=calVal(arg2_val,arg2_size)
val1=calVal(arg1_val,arg1_size)
print(f"{hex(val1)}?={hex(val2)}")
```

```
rand0m.check('11223344aabbccddeeffeeffaaccddee')
0x283af96e130?=0x283af96e130
0x8f154afd=0x11223344^0x9e3779b9
0x12223440=0x112233440&0xfa3affff
0x12223441=0x12223440+0x1
0x11e2a9**0x10001
0x13c501cc=0xef90a60e6d9e2a9%0xfffffffd  // 这里是不准确的，因为数字太大了，有0xd0b7位，所以只写了部分数字，猜测是pow之后的结果 然后mod 0xfffffffd
0x12223441?=0x12287f38
0x348cb564=0xaabbccdd^0x9e3779b9
0xaa38cdd0=0xaabbccdd0&0xfa3affff
0xaa38cdda=0xaa38cdd0+0xa
0x69196**0x10001
0x0=0x0%0xfffffffd
0xaa38cdda?=0x4a30f74d
0x43fbc213=0xddccbbaa^0x9e3779b9
0xd80abaa0=0xddccbbaa0&0xfa3affff
0xd80abaad=0xd80abaa0+0xd
0x87f78**0x10001
0x0=0x0%0xfffffffd
0xd80abaad?=0x23a1268
0xda045ba8=0x44332211^0x9e3779b9
0x42322110=0x443322110&0xfa3affff
0x42322114=0x42322110+0x4
0x1b408b**0x10001
0xc5e0a74e=0xccb22419f7f408b%0xfffffffd
0x42322114?=0x88108807
```

结合静态分析，每一轮有两次cmp，如何第一次cmp失败，第二次就不比较了，所以我们都下断点到richcmpare，将每次返回结果改为trueobject，然后获得新的tracelog

```
0x252bbc2e130?=0x252bbc2e130
0x8f154afd=0x11223344^0x9e3779b9
0x12223440=0x112233440&0xfa3affff
0x12223441=0x12223440+0x1
0x11e2a9**0x10001
div=0xa019a7204a7965087e77970426e62484218bd070e82b679cdc532edc68dfb2ace88e676cef90a60e6d9e2a9%0xfffffffd
0x12223441?=0x12287f38
0x27fe60b7?=0x98d24b3a
0x348cb564=0xaabbccdd^0x9e3779b9
0xaa38cdd0=0xaabbccdd0&0xfa3affff
0xaa38cdda=0xaa38cdd0+0xa
0x69196**0x10001
div=0x0%0xfffffffd
0xaa38cdda?=0x4a30f74d
0x2b356987?=0xe0f1db77
0x70c89746=0xeeffeeff^0x9e3779b9
0xea3aeff0=0xeeffeeff0&0xfa3affff
0xeeffeeffe=0xeeffeeff0+0xe
0xe1912**0x10001
div=0x0%0xfffffffd
0xea3aeffe?=0x23a1268 
0x81ac0cf0?=0xadf38403
0x34fba457=0xaaccddee^0x9e3779b9
0xa808dee0=0xaaccddee0&0xfa3affff
0xaaccddeea=0xaaccddee0+0xa
0x69f74**0x10001
div=0x0%0xfffffffd
0xa808deea?=0x88108807
0x61ff12d1?=0xd8499bb6
```

有pow 0x10001 以及 mod 0xfffffffd 我们可以猜测是一个rsa加密

由于pow的运算的第一个参数 位数太大了，勾的值并不是真实的value，有0xd087位，这里我们猜一下就知道是什么

我们盲猜第一个参数就是0x11e2a9**0x10001的结果

这里hex

![[f6aee8bc56944e2f69b3db1c86a248fe_MD5.png]]

恰好就是![[d97a83aa51917cbf1e4e04233dd6490f_MD5.png]]第二个cmp比较的结果

可以验证我们的猜想正确

现在我们不明白0x11e2a9 到底是不是与我们的输入有关，可以多输入几轮测试一下

```
0x252bbc2e130?=0x252bbc2e130
0x8f2668a8=0x11111111^0x9e3779b9
0x10101110=0x111111110&0xfa3affff
0x11e4cd**0x10001
div=0x6a21c24449c1db9452f195dcc87cf7a4c6173dfc07e6982cc02d8080186cd284c5ba2c3caf39ee88b7ad161d6c5e4cd%0xfffffffd
0x10101111?=0x12287f38
0x8f2668a8=0x11111111^0x9e3779b9
0x10101110=0x111111110&0xfa3affff
0x11e4cd**0x10001
div=0x6a21c24449c1db9452f195dcc87cf7a4c6173dfc07e6982cc02d8080186cd284c5ba2c3caf39ee88b7ad161d6c5e4cd%0xfffffffd
0x10101111?=0x4a30f74d
0x8f2668a8=0x11111111^0x9e3779b9
0x10101110=0x111111110&0xfa3affff
0x11e4cd**0x10001
div=0x6a21c24449c1db9452f195dcc87cf7a4c6173dfc07e6982cc02d8080186cd284c5ba2c3caf39ee88b7ad161d6c5e4cd%0xfffffffd
0x10101111?=0x23a1268
0x8f2668a8=0x11111111^0x9e3779b9
0x10101110=0x111111110&0xfa3affff
0x11e4cd**0x10001
div=0x6a21c24449c1db9452f195dcc87cf7a4c6173dfc07e6982cc02d8080186cd284c5ba2c3caf39ee88b7ad161d6c5e4cd%0xfffffffd
0x10101111?=0x88108807
```

这里我看不出来到底是怎么与input对应的,只能去静态分析一下

![[a9c231f269bcf9ea58896b617fc12276_MD5.png]]

参数a3为11

定位到这里，发现右移了11位

![[efe1128560b815d915f85aacb2099ee5_MD5.png]]验证猜想是正确的

这里我们可以发现实际上一组密文进行了2组处理

```
pow(((k^0x9e3779b9)>>11),0x10001,0xfffffffd)   ->即将一组密文进行rsa加密
((k<<4)&0xfa3affff)|k>>28   0x7ff0
```

这里的两组加密是会有信息缺失的，我们可以通过这两组加密完整恢复出明文

第一组，我们可以恢复出高22位，第二组恢复出低12位即可

trace已经dump出了每组密文

```
from Crypto.Util.number import *
cipher=[(0x98d24b3a,0x12287f38),(0xe0f1db77,0x4a30f74d),(0xadf38403,0x23a1268),(0xd8499bb6,0x88108807)]

def dersa(cipher):
    p=9241
    q=464773
    e=0x10001
    phi=(p-1)*(q-1)
    d=inverse(e,phi)
    return pow(cipher,d,p*q) 

for a,b in cipher:
    k1=dersa(a)
    k1=k1<<11
    k1^=0x9e3779b9
    k2=b
    k2&=0x7ff0
    k2>>4
    k=k1|k2
    print(hex(k))

#0x813affb9
#0xd4b37ff9
#0x802bb3f9
#0x789509b9
```

##### xdbg

xdbg条件断点

```
PyNumber_And
0x{([rcx+0x18] & 0xffffffff | (([rcx+0x18] >> 0x20) << 0x1e)) & ([rdx+0x18] & 0xffffffff | (([rdx+0x18] >> 0x20) << 0x1e))} = 0x{[rcx+0x18] & 0xffffffff | (([rcx+0x18] >> 0x20) << 0x1e)} & 0x{[rdx+0x18] & 0xffffffff | (([rdx+0x18] >> 0x20) << 0x1e)}
PyNumber_Add
0x{([rcx+0x18] & 0xffffffff | (([rcx+0x18] >> 0x20) << 0x1e)) + ([rdx+0x18] & 0xffffffff | (([rdx+0x18] >> 0x20) << 0x1e))} = 0x{[rcx+0x18] & 0xffffffff | (([rcx+0x18] >> 0x20) << 0x1e)} + 0x{[rdx+0x18] & 0xffffffff | (([rdx+0x18] >> 0x20) << 0x1e)}
PyNumber_Xor
0x{([rcx+0x18] & 0xffffffff | (([rcx+0x18] >> 0x20) << 0x1e)) ^ ([rdx+0x18] & 0xffffffff | (([rdx+0x18] >> 0x20) << 0x1e))} = 0x{[rcx+0x18] & 0xffffffff | (([rcx+0x18] >> 0x20) << 0x1e)} ^ 0x{[rdx+0x18] & 0xffffffff | (([rdx+0x18] >> 0x20) << 0x1e)}
PyNumber_LShift
0x{([rcx+0x18] & 0xffffffff | (([rcx+0x18] >> 0x20) << 0x1e)) << ([rdx+0x18] & 0xffffffff | (([rdx+0x18] >> 0x20) << 0x1e))} = 0x{[rcx+0x18] & 0xffffffff | (([rcx+0x18] >> 0x20) << 0x1e)} << 0x{[rdx+0x18] & 0xffffffff | (([rdx+0x18] >> 0x20) << 0x1e)}
PyNumber_RShift
0x{([rcx+0x18] & 0xffffffff | (([rcx+0x18] >> 0x20) << 0x1e)) >> ([rdx+0x18] & 0xffffffff | (([rdx+0x18] >> 0x20) << 0x1e))} = 0x{[rcx+0x18] & 0xffffffff | (([rcx+0x18] >> 0x20) << 0x1e)} >> 0x{[rdx+0x18] & 0xffffffff | (([rdx+0x18] >> 0x20) << 0x1e)}
PyNumber_Reminder
0x{([rcx+0x18] & 0xffffffff | (([rcx+0x18] >> 0x20) << 0x1e)) % ([rdx+0x18] & 0xffffffff | (([rdx+0x18] >> 0x20) << 0x1e))} = 0x{[rcx+0x18] & 0xffffffff | (([rcx+0x18] >> 0x20) << 0x1e)} % 0x{[rdx+0x18] & 0xffffffff | (([rdx+0x18] >> 0x20) << 0x1e)}
PyNumber_Power
0x{[rcx+0x18] & 0xffffffff | (([rcx+0x18] >> 0x20) << 0x1e)} ** 0x{[rdx+0x18] & 0xffffffff | (([rdx+0x18] >> 0x20) << 0x1e)}
PyObject_RichCompare
0x{[rcx+0x18] & 0xffffffff | (([rcx+0x18] >> 0x20) << 0x1e)} ?= 0x{[rdx+0x18] & 0xffffffff | (([rdx+0x18] >> 0x20) << 0x1e)}
```

command:run

#### 2025ciscn-cython

pyins 解包

pycdc无法反编译，用pycdas

```
 [Disassembly]
        0       RESUME                          0
        2       LOAD_CONST                      0: 0
        4       LOAD_CONST                      1: None
        6       IMPORT_NAME                     0: ez
        8       STORE_NAME                      0: ez
        10      PUSH_NULL
        12      LOAD_NAME                       1: input
        14      LOAD_CONST                      2: '璇疯緭鍏lag:'
        16      PRECALL                         1
        20      CALL                            1
        30      STORE_NAME                      2: flag
        32      PUSH_NULL
        34      LOAD_NAME                       3: list
        36      LOAD_NAME                       2: flag
        38      PRECALL                         1
        42      CALL                            1
        52      STORE_NAME                      4: flag1
        54      BUILD_LIST                      0
        56      STORE_NAME                      5: value
        58      LOAD_CONST                      0: 0
        60      STORE_NAME                      6: b
        62      LOAD_CONST                      0: 0
        64      STORE_NAME                      7: ck
        66      PUSH_NULL
        68      LOAD_NAME                       8: len
        70      LOAD_NAME                       4: flag1
        72      PRECALL                         1
        76      CALL                            1
        86      LOAD_CONST                      3: 24
        88      COMPARE_OP                      2 (==)
        94      POP_JUMP_FORWARD_IF_FALSE       259 (to 616)
        98      PUSH_NULL
        100     LOAD_NAME                       9: range
        102     LOAD_CONST                      0: 0
        104     PUSH_NULL
        106     LOAD_NAME                       8: len
        108     LOAD_NAME                       4: flag1
        110     PRECALL                         1
        114     CALL                            1
        124     LOAD_CONST                      4: 4
        126     PRECALL                         3
        130     CALL                            3
        140     GET_ITER
        142     FOR_ITER                        112 (to 368)
        144     STORE_NAME                      10: i
        146     PUSH_NULL
        148     LOAD_NAME                       11: ord
        150     LOAD_NAME                       4: flag1
        152     LOAD_NAME                       10: i
        154     BINARY_SUBSCR
        164     PRECALL                         1
        168     CALL                            1
        178     LOAD_CONST                      3: 24
        180     BINARY_OP                       3 (<<)
        184     PUSH_NULL
        186     LOAD_NAME                       11: ord
        188     LOAD_NAME                       4: flag1
        190     LOAD_NAME                       10: i
        192     LOAD_CONST                      5: 1
        194     BINARY_OP                       0 (+)
        198     BINARY_SUBSCR
        208     PRECALL                         1
        212     CALL                            1
        222     LOAD_CONST                      6: 16
        224     BINARY_OP                       3 (<<)
        228     BINARY_OP                       7 (|)
        232     PUSH_NULL
        234     LOAD_NAME                       11: ord
        236     LOAD_NAME                       4: flag1
        238     LOAD_NAME                       10: i
        240     LOAD_CONST                      7: 2
        242     BINARY_OP                       0 (+)
        246     BINARY_SUBSCR
        256     PRECALL                         1
        260     CALL                            1
        270     LOAD_CONST                      8: 8
        272     BINARY_OP                       3 (<<)
        276     BINARY_OP                       7 (|)
        280     PUSH_NULL
        282     LOAD_NAME                       11: ord
        284     LOAD_NAME                       4: flag1
        286     LOAD_NAME                       10: i
        288     LOAD_CONST                      9: 3
        290     BINARY_OP                       0 (+)
        294     BINARY_SUBSCR
        304     PRECALL                         1
        308     CALL                            1
        318     BINARY_OP                       7 (|)
        322     STORE_NAME                      6: b
        324     LOAD_NAME                       5: value
        326     LOAD_METHOD                     12: append
        348     LOAD_NAME                       6: b
        350     PRECALL                         1
        354     CALL                            1
        364     POP_TOP
        366     JUMP_BACKWARD                   113 (to 142)
        368     BUILD_LIST                      0
        370     LOAD_CONST                      10: (102, 108, 97, 103)
        372     LIST_EXTEND                     1
        374     STORE_NAME                      13: key
        376     BUILD_LIST                      0
        378     STORE_NAME                      14: flag_encrypt
        380     PUSH_NULL
        382     LOAD_NAME                       9: range
        384     LOAD_CONST                      0: 0
        386     LOAD_CONST                      11: 6
        388     LOAD_CONST                      7: 2
        390     PRECALL                         3
        394     CALL                            3
        404     GET_ITER
        406     FOR_ITER                        56 (to 520)
        408     STORE_NAME                      10: i
        410     PUSH_NULL
        412     LOAD_NAME                       0: ez
        414     LOAD_ATTR                       15: encrypt
        424     LOAD_NAME                       5: value
        426     LOAD_NAME                       10: i
        428     BINARY_SUBSCR
        438     LOAD_NAME                       5: value
        440     LOAD_NAME                       10: i
        442     LOAD_CONST                      5: 1
        444     BINARY_OP                       0 (+)
        448     BINARY_SUBSCR
        458     LOAD_NAME                       13: key
        460     PRECALL                         3
        464     CALL                            3
        474     STORE_NAME                      16: res
        476     LOAD_NAME                       14: flag_encrypt
        478     LOAD_METHOD                     12: append
        500     LOAD_NAME                       16: res
        502     PRECALL                         1
        506     CALL                            1
        516     POP_TOP
        518     JUMP_BACKWARD                   57 (to 406)
        520     PUSH_NULL
        522     LOAD_NAME                       0: ez
        524     LOAD_ATTR                       17: check
        534     LOAD_NAME                       14: flag_encrypt
        536     PRECALL                         1
        540     CALL                            1
        550     STORE_NAME                      7: ck
        552     LOAD_NAME                       7: ck
        554     LOAD_CONST                      9: 3
        556     COMPARE_OP                      2 (==)
        562     POP_JUMP_FORWARD_IF_FALSE       13 (to 590)
        564     PUSH_NULL
        566     LOAD_NAME                       18: print
        568     LOAD_CONST                      12: 'yes!!!,you get right flag'
        570     PRECALL                         1
        574     CALL                            1
        584     POP_TOP
        586     LOAD_CONST                      1: None
        588     RETURN_VALUE
        590     PUSH_NULL
        592     LOAD_NAME                       18: print
        594     LOAD_CONST                      13: 'wrong!!!'
        596     PRECALL                         1
        600     CALL                            1
        610     POP_TOP
        612     LOAD_CONST                      1: None
        614     RETURN_VALUE
        616     LOAD_CONST                      1: None
        618     RETURN_VALUE
```

用chatgpt翻译一下

```
import ez

flag = "aaaabbbbccccddddeeeeffff"  # Input message (possibly in a different encoding)
flag1 = list(flag)  # Convert flag to a list of characters
value = []
b = 0
ck = 0

# Check if length of input flag is 24
if len(flag1) == 24:
    for i in range(0, len(flag1), 4):
        b = (
            (ord(flag1[i]) << 24) |
            (ord(flag1[i + 1]) << 16) |
            (ord(flag1[i + 2]) << 8) |
            ord(flag1[i + 3])
        )
        value.append(b)

    key = [102, 108, 97, 103]  # ASCII values of 'f', 'l', 'a', 'g'
    flag_encrypt = []

    for i in range(0, 6, 2):
        res = ez.encrypt(value[i], value[i + 1], key)
        flag_encrypt.append(res)

    ck = ez.check(flag_encrypt)

    if ck == 3:
        print('yes!!!,you get right flag')
    else:
        print('wrong!!!')
else:
    print('wrong!!!')
```

ez的help

```
Help on module ez:

NAME
    ez

FUNCTIONS
    FormatError(...)
        FormatError([integer]) -> string

        Convert a win32 error code into a string. If the error code is not
        given, the return value of a call to GetLastError() is used.

    POINTER(...)

    addressof(...)
        addressof(C instance) -> integer
        Return the address of the C instance internal buffer

    alignment(...)
        alignment(C type) -> integer
        alignment(C instance) -> integer
        Return the alignment requirements of a C instance

    byref(...)
        byref(C instance[, offset=0]) -> byref-object
        Return a pointer lookalike to a C instance, only usable
        as function argument

    check(flag_encrypt)

    encrypt(V0, V1, key)

    get_errno(...)

    get_last_error(...)

    pointer(...)

    resize(...)
        Resize the memory buffer of a ctypes instance

    set_errno(...)

    set_last_error(...)

    sizeof(...)
        sizeof(C type) -> integer
        sizeof(C instance) -> integer
        Return the size in bytes of a C instance

DATA
    DEFAULT_MODE = 0
    GetLastError = <_FuncPtr object>
    RTLD_GLOBAL = 0
    RTLD_LOCAL = 0
    __test__ = {}
    cdll = <ctypes.LibraryLoader object>
    data = [(3572312064, 2468433000), (3725770449, 3734689877), (338735471...
    memmove = <CFunctionType object>
    memset = <CFunctionType object>
    oledll = <ctypes.LibraryLoader object>
    pydll = <ctypes.LibraryLoader object>
    pythonapi = <PyDLL 'python dll', handle 7ffa16710000>
    windll = <ctypes.LibraryLoader object>

FILE
    c:\users\npc\desktop\ctf\events\2025ciscn\cython\cython.exe_extracted\ez.pyd
```

这里可以看到data，应该就是密文了,结合源码三组一一对应

```
[(3572312064, 2468433000), (3725770449, 3734689877), (3387354714, 1042217819)]
```

部分tracelog

```
0x4fc00605f3=0x400606f0^0x4f80000303
0x204dc006c6d1=0x1ffe400605f3+0x4f8000c0de
0x80001122=0x0+0x80001122
0x4fc006d7f3=0x4006c6d1^0x4f80001122
0x13ec03dabda=0x13e803db500^0x40001eda
0x1ffe8045627a=0x1ffe403dabda+0x4007b6a0
0xd4647576=0x54646454+0x80001122
0x13ed421170c=0x13e8045627a^0x54647576
0x13c205e380f=0x2a10ebf50^0x13e8150875f
0x2f4800ff9=0x2a05e380f+0x5421d7ea
0xd4647576=0x54646454+0x80001122
0x2a0e47a8f=0x2f4800ff9^0x54647576
0x785e239bc=0x507618978^0x28283b0c4
0x5a6ce6aeb=0x505e239bc+0xa0ec312f
0xa8c8fbec=0xa8c8c8a8+0x3344
0x50e069107=0x5a6ce6aeb^0xa8c8fbec
0x390cbe62b=0x311434788^0x8188a1a3
0x372f44f1c=0x310cbe62b+0x622868f1
0x128c8d9ca=0xa8c8c8a8+0x80001122
0x3da3c96d6=0x372f44f1c^0xa8c8d9ca
0xd8aae308=0x3d9464028^0x301eca320
0x453d3ab0d=0x3d8aae308+0x7b28c805
0xfd2d6040=0xfd2d2cfc+0x3344
0x4aefecb4d=0x453d3ab0d^0xfd2d6040
0x497d3d20=0x8939a1f0^0xc0449cd0
0x15aa4715e=0x897d3d20+0xd127343e
0x17d2d3e1e=0xfd2d2cfc+0x80001122
0x67894f40=0x9aa4715e^0xfd2d3e1e
0x7961a7275=0x71590ba28^0x838ac85d
0x7f8cc89ba=0x7161a7275+0xe2b21745
0x5191e6b6=0x51919150+0x5566
0x7a95d6f0c=0x7f8cc89ba^0x5191e6b6
0x596cf08dd=0x5d4251a50^0x42ea128d
0x69153ac27=0x5d6cf08dd+0xba84a34a
0xd191a272=0x51919150+0x80001122
0x6c0c20e55=0x69153ac27^0x5191a272
0xd92cfc46=0x51ba12cd0^0x5c28dd096
0x5bca121e0=0x5192cfc46+0xa374259a
0xa5f64b0a=0xa5f5f5a4+0x5566
0x519576aea=0x5bca121e0^0xa5f64b0a
0x61daf0198=0x69ee071a0^0x834f7038
0x7718b0fcc=0x69daf0198+0xd3dc0e34
0x125f606c6=0xa5f5f5a4+0x80001122
0x7d47d090a=0x7718b0fcc^0xa5f606c6
0x53e56b19a=0x3bf897520^0x681dfc4ba
0x43647e03e=0x3be56b19a+0x77f12ea4
0xfa5ad180=0xfa5a59f8+0x7788
0x4cc1d31be=0x43647e03e^0xfa5ad180
0x43db61b6f=0x4ffc9ff90^0xc27fe4ff
0x59daf5b61=0x4fdb61b6f+0x9ff93ff2
0x17a5a6b1a=0xfa5a59f8+0x80001122
0x567f5307b=0x59daf5b61^0xfa5a6b1a
0x23c4d6184=0x6ff32f8f8^0x4c37f997c
0x7dc33c0a3=0x6fc4d6184+0xdfe65f1f
0x4ebf35d4=0x4ebebe4c+0x7788
0x7928cf577=0x7dc33c0a3^0x4ebf35d4
0x1d4fbb39d=0x19431ab48^0x40ca18d5
0x20781e906=0x194fbb39d+0x72863569
0xcebecf6e=0x4ebebe4c+0x80001122
0x1893f2668=0x1c781e906^0x4ebecf6e
0x2c888ba2e=0x3492c2c38^0x181a49616
0x3b1ae3fb5=0x34888ba2e+0x69258587
0x1232333c2=0xa32322a0+0x80001122
0x3128d0c77=0x3b1ae3fb5^0xa32333c2
0x2a98e4207=0x2289a0f00^0x81144d07
0x26ea183e7=0x2298e4207+0x451341e0
0x1232333c2=0xa32322a0+0x80001122
0x2cd82b025=0x26ea183e7^0xa32333c2
0x3b59b0db6=0x1b541ad60^0x200daa0d6
0x3ec434362=0x1b59b0db6+0x236a835ac
0x177879816=0xf78786f4+0x80001122
0x11bc4db74=0x1ec434362^0xf7879816
0x3c7438ad5=0x306c0eaa0^0xc1836075
0x3681ba829=0x307438ad5+0x60d81d54
0x177879816=0xf78786f4+0x80001122
0x39f9c303f=0x3681ba829^0xf7879816
0x5b17a3ecf=0x6b2232f58^0x303591197
0x787bea4ba=0x6b17a3ecf+0xd64465eb
0x4bec1e8c=0x4bebeb48+0x3344
```

这里的shift没有用到pynumber_shift，而是经过了底层优化编程的C语言的运算符，所以trace不出来，我们只能静态分析

这里的数字位数确定不了,写hook脚本很难受

```
import idc
def calVal(val,size):
    val=int.from_bytes(val,byteorder='little')
    result=(val&0xffffffff) |(val>>32)<<30
    return result

arg2_addr=idc.get_reg_value("rdx")
arg1_addr=idc.get_reg_value("rcx")
arg2_size=idc.get_bytes(arg2_addr+0x10,1)
arg1_size=idc.get_bytes(arg1_addr+0x10,1)
arg2_val=idc.get_bytes(arg2_addr+0x18,8)
arg1_val=idc.get_bytes(arg1_addr+0x18,8)
val2=calVal(arg2_val,arg2_size)
val1=calVal(arg1_val,arg1_size)
print(f"{hex(val1<<val2)}={hex(val1)}<<{hex(val2)}")
```

直接静态看也不是不可以

![[3f85506ffe61cec0ddf20c2cd47552cd_MD5.png]]

![[75d573f5029f6adab437af74bd811d6b_MD5.png]]

首先找到加密函数,第二个参数是输入的第一个数组，第三个参数是输入的第二个数据，第三个应该是列表的地址

进去后，中间有一个大循环

![[55b3b0a4c8a71790dc8778e55815fa60_MD5.png]]

循环前面的一堆都是初始化代码，可以不看

![[5244330940e8a9753f20410789b7e417_MD5.png]]

![[06af0ebb4c7b9ca65ab8ede86f5b9a49_MD5.png]]

![[7c7b4b7bc2f058fcc7a508f31b7402c5_MD5.png]]

跟踪数据流 结合tracelog最后就能得到最后的等式

最后的exp

```
#include "stdio.h"
#include "stdint.h"
typedef uint32_t uint32;
typedef uint8_t uint8;
void decipher(unsigned int num_rounds, uint32_t v[2], uint32_t const key[4])
{
    unsigned int i;
    uint32_t v0 = v[0], v1 = v[1], delta = 0x54646454, sum = delta * num_rounds;
    for (i = 0; i < num_rounds; i++)
    {
        v1 -= (((v0 << 3) ^ (v0 >> 6)) + v0) ^ (sum + key[(sum >> 11) & 3]);
        sum -= delta;
        v0 -= (((v1 << 3) ^ (v1 >> 6)) + v1) ^ (sum + key[sum & 3]);
    }
    v[0] = v0;
    v[1] = v1;
}

int main()
{
    uint32 v[] = {
        3572312064, 2468433000, 3725770449, 3734689877, 3387354714,  1042217819
        };

    uint32 key[] = {
        102, 108, 97, 103
        };

    for (int i = 0; i < 6; i += 2) {
        decipher(64, v + i, key);
    }

    uint8 *ptr = (uint8 *) v;

    for (int i = 0; i < 24; i += 4) {
        printf("%c%c%c%c", ptr[i + 3], ptr[i + 2], ptr[i + 1], ptr[i]);
    }

    return 0;
}

#flag{pFmuS0qUf8ic2rYrXO}
```