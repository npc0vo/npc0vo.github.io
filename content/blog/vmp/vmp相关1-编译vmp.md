---
title: vmp相关1-编译vmp
tags:
  - "#VMProtection"
  - "#blog"
description:
date: 2025-02-15
aliases:
draft:
---
[https://bbs.kanxue.com/thread-279882.htm](https://bbs.kanxue.com/thread-279882.htm)

[https://www.chinapyg.com/thread-149872-1-1.html](https://www.chinapyg.com/thread-149872-1-1.html)

[https://bbs.kanxue.com/thread-279886.htm](https://bbs.kanxue.com/thread-279886.htm)

## 准备工作

Visual Studio 2019 QT 5.6.0

[https://builds.dotnet.microsoft.com/dotnet/Sdk/3.1.426/dotnet-sdk-3.1.426-win-x64.exe](https://builds.dotnet.microsoft.com/dotnet/Sdk/3.1.426/dotnet-sdk-3.1.426-win-x64.exe)

[https://download.qt.io/new_archive/qt/5.6/5.6.0/qt-opensource-windows-x86-msvc2015_64-5.6.0.exe](https://download.qt.io/new_archive/qt/5.6/5.6.0/qt-opensource-windows-x86-msvc2015_64-5.6.0.exe)

![[3cd9ee349e1e468af972377298b8ef66_MD5.png]]

## 编译

只是为了学习vmp的源码，实际上只编译console版即可。但是命令行还是有点不方便，所以编译了一个共享库版本的ultimate vmp。静态编译有点麻烦，而且也用不上，所以就没搞

编译过程具体参考上方的网站，这里记录我编译过程遇到的问题

初次装载项目

![[2040e9067ac17e6f9d929b5627561ab9_MD5.png]]

如果VMProtect.Netcore装载不上下载[https://builds.dotnet.microsoft.com/dotnet/Sdk/3.1.426/dotnet-sdk-3.1.426-win-x64.exe](https://builds.dotnet.microsoft.com/dotnet/Sdk/3.1.426/dotnet-sdk-3.1.426-win-x64.exe)

打开项目文件，自动检查解决方案不匹配的地方，显示如下对话框，允许项目升级

设置工程为X64

直接在VMProtectSDK工程上右键“生成”，很快显示成功。

```
“void IntelFunction::Mutate(const CompileContext   &,bool)”:“IntelFunction”中没有找到重载的成员函数
void Mutate(const CompileContext &ctx, bool for_virtualization, int index = 0);
原因是声明中使用默认构造参数，定义中直接缺少了构造参数。
intel.cc将其修改为：
void IntelFunction::Mutate(const CompileContext& ctx, bool for_virtualization, int index )
```

```
“void IntelObfuscation::Compile(IntelFunction   *,size_t)”:“IntelObfuscation”中没有找到重载的成员函数
intel.cc将其修改为：

void IntelObfuscation::Compile(IntelFunction* func, size_t index, size_t end_index, bool for_virtualization)
```

```
无法打开 源 文件   "QtCore/QMap"
```

右键VMProtect属性，修改VC++目录的包含目录和库目录

![[2dc0ae0bf384f9dfa8d61754d9f42072_MD5.png]]

修改附加包含目录

![[95a5110ac51f276cfc5bf58df6e42d8f_MD5.png]]

编码报错：双击打开错误打开文件，然后点击：文件——高级保存选项——以"Unicode-代码页1200"保存。或者直接用vscode打开重新复制然后粘贴到vsstudio即可

```
Your project does not reference".NETFramework,Version=v4.8" framework. Add a reference to".NETFramework,Version=v4.8" in the "TargetFrameworks"property of your project file and then re-run NuGet restore.
```

"F:\codes\vmp\runtime\VMProtect.Runtime\VMProtect.Netcore.csproj"

![[b205994dbb3a12d0d31fa659f4dc0507_MD5.png]]

补充net48后出现

```
Your project file doesn't list 'win-x64' asa "RuntimeIdentifier". You should add 'win-x64' to the"RuntimeIdentifiers" property in your project file and then re-runNuGet restore.
```

![[929df5d55444aef8e0f2cac3f523499c_MD5.png]]

补充`<RuntimeIdentifier>win-x64</RuntimeIdentifier>`

```
error CS0030: 无法将类型“object”转换为“System.TypedReference”
```

(TypedReference)args[0]`改为__makeref(args[0])

```
res.bat: QT resources 'VMProtect\resources.cc' are up to date
```

修改`"F:\codes\vmp\VMProtect\res.bat"`

```
echo res.bat: generating QT resources...
SET RC_DIR=%~dp0

set rc_out=%RC_DIR%resources.cc
set rc=%rc_out%
set check_rc=0
if exist %rc_out% (
    set rc_out=%RC_DIR%resources.cc.tmp
    set check_rc=1
)


"F:\QT\QT5.6\5.6\msvc2015_64\bin\rcc.exe" %RC_DIR%application.qrc -o %rc_out%

call :ReplaceOld %check_rc% %rc_out% %rc%
goto :EOF

:ReplaceOld
setlocal enableextensions enabledelayedexpansion
if "%1" == "1" (
  fc %2 %3 /B>>nul 2>&1
  if !ERRORLEVEL! == 1 (
   copy /y %2 %3>>nul 2>&1
   del %2>>nul 2>&1
   echo  res.bat: QT resources '%3' are updated 
  ) else (
   echo  res.bat: QT resources '%3' are up to date
  )
) else (
 echo  res.bat: QT resources '%3' are generated
)
endlocal
goto :EOF
```

缺少MOC文件:

重新生成解决方案，神秘原因会导致VMProtect目录下的MOC文件也被删除，直接复制粘贴补回去就行了

最终:

![[e6cff7cb710ac45f9b5cf52cc6042e55_MD5.png]]

## 项目结构

![[953ac3c696b638678c8692ef330cf9f0_MD5.png]]

Common文件夹存在两个项目，是核心模块，包括了vmp的核心功能,重点分析

![[43d50a47202ec0c0f5180c6aa45afc80_MD5.png]]

misc文件夹存在3个项目

Tests就是测试模块

ipn_tool 是注册网络验证模块,添加水印等

ShellExt Windows外壳扩展，增强文件或文件夹的右键菜单功能。

不重要

![[edd7ed8268fa20a65df662131c722a01_MD5.png]]

- 这个文件夹通常包含解决方案级别的配置文件，如：

- `NetX64.testsettings` 和 `NetX86.testsettings`：这些文件可能用于配置不同平台（64位和32位）的测试环境。

不重要

![[6e6b245c8e738190632e105de24842ab_MD5.png]]

QT版本的vmp

不重要

![[1edeb88a44650043d8d067c5730ed1c2_MD5.png]]

不重要

![[a7aa28efb974816237b6513d209b12c5_MD5.png]]

不重要

![[d34db021c97228b3b0d94b6581d8139a_MD5.png]]

![[87a655f40526d9c12b99dae5c43375f2_MD5.png]]

![[da0a5007d4637408b9ee60d1e733593d_MD5.png]]

控制台版本的vmp

![[ba15d31e8c76e383bae43d2b148a2850_MD5.png]]

比较重要

## vmp相关分析资料

[GitHub - lmy375/awesome-vmp: 虚拟化保护（VMP壳）分析相关资料](https://github.com/lmy375/awesome-vmp)

[[原创]vm虚拟机的初探-Android安全-看雪-安全社区|安全招聘|kanxue.com](https://bbs.kanxue.com/thread-284883.htm)

[[原创]某短视频虚拟机分析和还原-Android安全-看雪-安全社区|安全招聘|kanxue.com](https://bbs.kanxue.com/thread-282300.htm)

[VMP 简单源码分析（.net）_vmp源码讲解-CSDN博客](https://blog.csdn.net/Misaka10046/article/details/138485125)

[[原创]VMP3.6代码虚拟化从了解到放弃-软件逆向-看雪-安全社区|安全招聘|kanxue.com](https://bbs.kanxue.com/thread-274112.htm)

[VMProtect分析-虚拟化流程](https://blog.planejun.cn/2023/01/05/VMProtect%E5%88%86%E6%9E%90-%E8%99%9A%E6%8B%9F%E5%8C%96%E6%B5%81%E7%A8%8B/)

[[原创]如何分析虚拟机（1）：新手篇VMProtect 1.81 Demo-软件逆向-看雪-安全社区|安全招聘|kanxue.com](https://bbs.kanxue.com/thread-225262.htm[%E5%A6%82%E4%BD%95%E5%88%86%E6%9E%90%E8%99%9A%E6%8B%9F%E6%9C%BA%EF%BC%881%EF%BC%89%EF%BC%9A%E6%96%B0%E6%89%8B%E7%AF%87VMProtect)

[[原创]如何分析虚拟机（2）：进阶篇 VMProtect 2.13.8-软件逆向-看雪-安全社区|安全招聘|kanxue.com](https://bbs.kanxue.com/thread-225803.htm)

[🔥 Quick look around VMP 3.x - Part 2 : Code Mutation](https://whereisr0da.github.io/blog/posts/2021-01-26-vmp-2/)

[Lifting Binaries, Part 0: Devirtualizing VMProtect and Themida: It’s Just Flattening?](https://nac-l.github.io/2025/01/25/lifting_0.html)