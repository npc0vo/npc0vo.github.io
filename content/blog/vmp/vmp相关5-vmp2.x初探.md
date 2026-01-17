---
title: vmp相关5-vmp2.x初探
tags:
  - blog
  - VMProtection
description:
date: 2025-02-27
aliases:
draft:
---
[https://bbs.kanxue.com/thread-225803.htm](https://bbs.kanxue.com/thread-225803.htm)

## 准备


Demo 版因为代码没有混淆处理，因此在 IDA 中可以分析的很清楚了，甚至还可以根据字节码（VM_DATA）一点点静态还原虚拟指令。

然而正式版本的 VMProtect 虚拟机是有比较严重的混淆的（实际是一种冗余指令的添加），直接使用 IDA 分析十分困难，许多基本块会截断，没有先前明显的解释循环图结构。动态调试也不方便，里面大量的CALL\JMP，跳来跳去。ESI 还有指令的立即数等还有加密，整体复杂度有很大的提升。

使用VMProtect 2.13.8 加密级别为**最大速度** 关闭IAT保护，反调试等保护

```
sub_401000 proc near
mov     eax, dword_403000
add     eax, 12345678h
sub     eax, 12345678h
mov     dword_403000, eax
retn
sub_401000 endp
```

## vmp2.13.8简介

到了 2.13.8 版本， VMProcect 的整体结构仍是和 VMProtect 1.81 一致的。 因此虽然存在混淆，经验丰富的人还是可以迅速找到关键的指令分发点（Dispatcher）位置，从而找到跳转表，提取所有的 Handler。

![[5faca1efeb36ddcbb0c34af32892d661_MD5.png]]

实际上，这几行代码就是典型的 dispatcher 代码，根据 0x404cf8 跳转表，再找到 ESI 解密的方式，就可以提取所有 Handler

## 虚拟机结构分析

从头开始分析

```
.text:00401000                 jmp     loc_405DD8  ; 跳转到 .vmp0 中的虚拟机代码
 
.vmp0:00405DD8                 pushf
.vmp0:00405DD9                 call    sub_405CF7  ; call并不是函数调用，只是跳转进入虚拟机中
 
.vmp0:00405CF7                 mov     byte ptr [esp+0], 0A4h
.vmp0:00405CFB                 mov     dword ptr [esp+4], 95A00657h
.vmp0:00405D03                 mov     [esp+0], dl
.vmp0:00405D06                 mov     dword ptr [esp+0], 6B1DA363h
.vmp0:00405D0D                 pusha
.vmp0:00405D0E                 push    dword ptr [esp+8]
.vmp0:00405D12                 pusha
.vmp0:00405D13                 mov     [esp+20], al
.vmp0:00405D17                 lea     esp, [esp+44h] ; 前面压栈的内容弹出来
.vmp0:00405D1B                 jmp     loc_4048F4
 
.vmp0:004048F4                 jmp     loc_40593D
 
...
```

通过分析一部分指令，我们可以发现`vmp`存在大量的`jmp` 和`call` 将代码切割了很多小片段，ida 无法正常分析，也就无法形成正确的`cfg`，没有正确的`cfg` 无法就无法很容易的确定`handler` 和`dispatcher`在哪里，逆向变得很困难

同时代码片段里还充斥着大量的无用代码，例如 开头向栈内压入了很多数据，但是后面直接降低栈顶，又把数据给弹出去了，相当于什么都没干，也就是混淆代码

如何生成`cfg`:原贴的作者写了一个工具[https://github.com/lmy375/pinvmp](https://github.com/lmy375/pinvmp)

**由于onlydbg与x96dbg trace的速度大概是20条指令，太慢了**

**首先是利用了**`**pin**`**记录**`**trace**`**日志，分析出每个**`**handler**`**的指令序列，再利用**`**miasm**`**进行符号执行来稍微简化部分代码(事实上化简效果并不好，但是已经是7/8年前的项目,原作者很nb了)**

由于项目比较老，要使用要注意几个点:

1. 使用conda 下载 `python2.7`
2. `miasm` 下载 `0.0.1` 版本
3. 下载 [pin-3.2-81205-msvc-windows](https://software.intel.com/en-us/articles/pin-a-binary-instrumentation-tool-downloads),编译项目中的mypintool
4. 报错提示缺什么库 直接安装，注意修改config，注意需要自己创建work目录和image目录
5. 提示`dot.exe`不在当前path，可以使用conda install graphviz

这样我们就能获得`vmp`的cfg图了

![[51084d45f8d7fd947124af7349379ee0_MD5.svg]]

原贴作者分析`vmp`生成cfg的思路如下:

为了恢复这个图，我们还是使用 Trace 分析的方法。Trace 是 x86 指令执行的序列，Trace 中所有 jmp 和 call 会天然的和跳转目标指令连接起来。Trace 分析其实是很强大的功能。单步执行时我们的注意力可能会被寄存器、内存的值所分散，而忽略了程序整体的执行情况。而通过对 Trace 的整体分析，则可以让我们跳出局部，从整体去观察。

虚拟机解释执行的过程是一个循环：首先取指令，解码，然后跳转到 Handler 代码，执行完成后再跳转回来。在 Trace 中我们如何捕捉这个关键的循环？我想到的方法是，由 Trace 构造一个像之前 IDA 显示的 CFG 图类似的图。 通过**找图中的中心结点**，来确定 dispatcher 的位置。

由 Trace 构造的图，是反映了程序执行过程的基本块图，我们可以称之为**执行流图**。构造图的方法说起来比较费力，直接看图。

![[384fbbdf439af44a8ea91cb2809dbcb7_MD5.png]]

假设ABCD是执行的指令序列，按照每条指令在Trace中的先后顺序，就可以构造一个图。相邻结点合并一下，就可以得到最终的执行流图。最终 AB 形成一个块，执行到 CDE 块，再跳转回来，再执行FG块，再跳转回来，再执行HI块，再跳转回来。整个执行的过程就很清楚了。那么AB块就是dispatcher，CDE,FG,HI就是handler

### Dispatcher分析

```
0x40435f        lea edx, ptr [edx*8-0x7525bb7f]
0x404366        mov al, byte ptr [esi-0x1]     ; [esi - 0x1] 取指令
0x404369        inc dl
0x40436b        setns dh
0x40436e        dec esi
0x40436f        sub edx, 0x9ec1cfd2
0x404375        cmp dl, 0xea
0x404378        rcl dl, 0x7
0x40437b        sub al, bl  ; 解密 al
0x40437d        dec dl
0x40437f        push esi
0x404380        call 0x4041ce
0x4041ce        pop edx
0x4041cf        dec al      ; 解密 al
0x4041d1        rcl dh, cl
0x4041d3        sar edx, cl
0x4041d5        or dl, ch
0x4041d7        xor al, 0xcd  ; 解密 al
0x4041d9        lea edx, ptr [ebx*8-0xcbcbf63]
0x4041e0        sar dx, cl
0x4041e3        rcr edx, 0x10
0x4041e6        rol dx, cl
0x4041e9        sub al, 0x23
0x4041eb        shl dl, cl
0x4041ed        bsr dx, dx
0x4041f1        bt bx, 0x9
0x4041f6        sub bl, al
0x4041f8        rcl dh, 0x4
0x4041fb        movzx eax, al  ; eax <- al
0x4041fe        pushad
0x4041ff        mov edx, dword ptr [eax*4+0x404cf8]  ; 取跳转地址， 跳转表就是 0x404cf8
0x404206        cmc
0x404207        sub edx, 0x1aadee33  ; 解密跳转地址
0x40420d        mov byte ptr [esp+0x8], 0x28
0x404212        pushad
0x404213        mov dword ptr [esp+0x40], edx  ; 跳转地址压栈
0x404217        pushfd
0x404218        mov byte ptr [esp+0x4], ch
0x40421c        push dword ptr [esp+0x8]
0x404220        push dword ptr [esp+0x48]
0x404224        ret 0x4c  ; 通过 ret 跳转到前面压栈的 edx
```

相比我们`vmp1.8`分析的`dispatcher`对比 多了很多莫名奇妙的代码，实际上这些代码都是垃圾指令，人力识别这些垃圾指令很耗费时间 我们大概识别其中的关键操作:重点关注对`edi`,`esp`,`eax`的操作，我们大概能分析出操作

也就是先取指令，再用某种解密算法取跳转地址，解密地址，然后利用`ret`跳转到对应的`handler`

### Handler分析

我们使用`pinvmp`来辅助分析,对于一些简单的`handler`分析还是很强大的

#### **[Handler_405640]**

```
0x405640    cmc     ;junk
0x405641    cmp bl, dl    ; edx->0x40    ebx->0x81  junk
0x405643    push dword ptr [ebp]    ;junk ebp->0x19ff70    esp->0x19fe80    esp<-0x19fe7c    [0x19ff70]->0x200    [0x19fe7c]<-0x200
0x405646    push 0x45180ac6    ;junk esp->0x19fe7c    esp<-0x19fe78    [0x19fe78]<-0x45180ac6
0x40564b    jmp 0x404b81    ;junk
0x404b81    bt di, 0x5    ;junk edi->0xfe80
0x404b86    add ebp, 0x4    ; ****ebp->0x19ff70    ebp<-0x19ff74
0x404b89    std     ;junk
0x404b8a    push dword ptr [esp+0x4]    ; junk esp->0x19fe78    esp<-0x19fe74    [0x19fe7c]->0x200    [0x19fe74]<-0x200
0x404b8e    popfd     ; esp->0x19fe74   junk esp<-0x19fe78    [0x19fe74]->0x200
0x404b8f    pushad     ; esp->0x19fe78 junk    edi->0x19fe80    eax->0xa3    ebp->0x19ff74    edx->0x405640    ebx->0x1ee9e381    esi->0x405d74    ecx->0x401004    esp<-0x19fe58    [0x19fe58]<-0x405d740019fe80    [0x19fe60]<-0x19fe780019ff74    [0x19fe70]<-0xa300401004    [0x19fe68]<-0x4056401ee9e381
0x404b90    mov byte ptr [esp+0x4], ah    ;junk eax->0x0    esp->0x19fe58    [0x19fe5c]<-0x0
0x404b94    lea esp, ptr [esp+0x28]    ;junk esp->0x19fe58    esp<-0x19fe80
0x404b98    jmp 0x40435f    ;
```

我们分析汇编，人肉去除掉一些junkcode，发现只是将`ebp+4` ,其中****代表关键操作

这里去除了无用的`jmp`以及用`miasm`进行符号执行后，可以看到化简结果

![[9fd485c8e7a6d4272047b8c5627a3405_MD5.png]]

分析`vmp1.8` 可以知道 此时`ebp`指向`vm_stack`栈顶

即相当于`vPop4`

#### 0x405881

```
main	00405881	push    39C01C1A	[0019FE88]=00405881	EAX=0000000C, ECX=00401004, EDX=00405881, EBX=5ACE87A5, ESP=0019FE8C, EBP=0019FF44, ESI=00405C57, EDI=0019FE8C
main	00405886	mov     esi, dword ptr ss:[esp]	[0019FE88]=39C01C1A	EAX=0000000C, ECX=00401004, EDX=00405881, EBX=5ACE87A5, ESP=0019FE88, EBP=0019FF44, ESI=00405C57, EDI=0019FE8C
main	00405889	bsf     si, cx		EAX=0000000C, ECX=00401004, EDX=00405881, EBX=5ACE87A5, ESP=0019FE88, EBP=0019FF44, ESI=39C01C1A, EDI=0019FE8C
main	0040588D	pushfd	[0019FE84]=0000000C	EAX=0000000C, ECX=00401004, EDX=00405881, EBX=5ACE87A5, ESP=0019FE88, EBP=0019FF44, ESI=39C00002, EDI=0019FE8C
main	0040588E	mov     esi, dword ptr ss:[ebp]	[0019FF44]=00405DD8	EAX=0000000C, ECX=00401004, EDX=00405881, EBX=5ACE87A5, ESP=0019FE84, EBP=0019FF44, ESI=39C00002, EDI=0019FE8C
main	00405891	stc		EAX=0000000C, ECX=00401004, EDX=00405881, EBX=5ACE87A5, ESP=0019FE84, EBP=0019FF44, ESI=00405DD8, EDI=0019FE8C
main	00405892	call    00405662		EAX=0000000C, ECX=00401004, EDX=00405881, EBX=5ACE87A5, ESP=0019FE84, EBP=0019FF44, ESI=00405DD8, EDI=0019FE8C
main	00405662	mov     byte ptr ss:[esp+8], bh	[0019FE88]=1A	EAX=0000000C, ECX=00401004, EDX=00405881, EBX=5ACE87A5, ESP=0019FE80, EBP=0019FF44, ESI=00405DD8, EDI=0019FE8C
main	00405666	pushfd	[0019FE7C]=0FC4000F	EAX=0000000C, ECX=00401004, EDX=00405881, EBX=5ACE87A5, ESP=0019FE80, EBP=0019FF44, ESI=00405DD8, EDI=0019FE8C
main	00405667	add     ebp, 4		EAX=0000000C, ECX=00401004, EDX=00405881, EBX=5ACE87A5, ESP=0019FE7C, EBP=0019FF44, ESI=00405DD8, EDI=0019FE8C
main	0040566A	pushfd	[0019FE78]=5ACE87A5	EAX=0000000C, ECX=00401004, EDX=00405881, EBX=5ACE87A5, ESP=0019FE7C, EBP=0019FF48, ESI=00405DD8, EDI=0019FE8C
main	0040566B	lea     esp, [esp+14]	Address=0019FE8C	EAX=0000000C, ECX=00401004, EDX=00405881, EBX=5ACE87A5, ESP=0019FE78, EBP=0019FF48, ESI=00405DD8, EDI=0019FE8C
main	0040566F	jmp     00404349		EAX=0000000C, ECX=00401004, EDX=00405881, EBX=5ACE87A5, ESP=0019FE8C, EBP=0019FF48, ESI=00405DD8, EDI=0019FE8C
main	00404349	xadd    bh, al		EAX=0000000C, ECX=00401004, EDX=00405881, EBX=5ACE87A5, ESP=0019FE8C, EBP=0019FF48, ESI=00405DD8, EDI=0019FE8C
main	0040434C	and     al, bl		EAX=00000087, ECX=00401004, EDX=00405881, EBX=5ACE93A5, ESP=0019FE8C, EBP=0019FF48, ESI=00405DD8, EDI=0019FE8C
main	0040434E	xadd    dl, al		EAX=00000085, ECX=00401004, EDX=00405881, EBX=5ACE93A5, ESP=0019FE8C, EBP=0019FF48, ESI=00405DD8, EDI=0019FE8C
main	00404351	sar     dh, cl		EAX=00000081, ECX=00401004, EDX=00405806, EBX=5ACE93A5, ESP=0019FE8C, EBP=0019FF48, ESI=00405DD8, EDI=0019FE8C
main	00404353	mov     ebx, esi		EAX=00000081, ECX=00401004, EDX=00400506, EBX=5ACE93A5, ESP=0019FE8C, EBP=0019FF48, ESI=00405DD8, EDI=0019FE8C
main	00404355	cmp     bh, ah		EAX=00000081, ECX=00401004, EDX=00400506, EBX=00405DD8, ESP=0019FE8C, EBP=0019FF48, ESI=00405DD8, EDI=0019FE8C
main	00404357	rcl     al, 7		EAX=00000081, ECX=00401004, EDX=00400506, EBX=00405DD8, ESP=0019FE8C, EBP=0019FF48, ESI=00405DD8, EDI=0019FE8C
main	0040435A	ror     dh, cl		EAX=000000A0, ECX=00401004, EDX=00400506, EBX=00405DD8, ESP=0019FE8C, EBP=0019FF48, ESI=00405DD8, EDI=0019FE8C
main	0040435C	add     esi, dword ptr ss:[ebp]	[0019FF48]=00000000	EAX=000000A0, ECX=00401004, EDX=00405006, EBX=00405DD8, ESP=0019FE8C, EBP=0019FF48, ESI=00405DD8, EDI=0019FE8C
main	0040435F	lea     edx, [edx*8+8ADA4481]		EAX=000000A0, ECX=00401004, EDX=00405006, EBX=00405DD8, ESP=0019FE8C, EBP=0019FF48, ESI=00405DD8, EDI=0019FE8C
main	00404366	mov     al, byte ptr ds:[esi-1]	[00405DD7]=1B	EAX=000000A0, ECX=00401004, EDX=8CDCC4B1, EBX=00405DD8, ESP=0019FE8C, EBP=0019FF48, ESI=00405DD8, EDI=0019FE8C
main	00404369	inc     dl		EAX=0000001B, ECX=00401004, EDX=8CDCC4B1, EBX=00405DD8, ESP=0019FE8C, EBP=0019FF48, ESI=00405DD8, EDI=0019FE8C
main	0040436B	setns   dh		EAX=0000001B, ECX=00401004, EDX=8CDCC4B2, EBX=00405DD8, ESP=0019FE8C, EBP=0019FF48, ESI=00405DD8, EDI=0019FE8C
main	0040436E	dec     esi		EAX=0000001B, ECX=00401004, EDX=8CDC00B2, EBX=00405DD8, ESP=0019FE8C, EBP=0019FF48, ESI=00405DD8, EDI=0019FE8C
main	0040436F	sub     edx, 9EC1CFD2		EAX=0000001B, ECX=00401004, EDX=8CDC00B2, EBX=00405DD8, ESP=0019FE8C, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	00404375	cmp     dl, 0EA		EAX=0000001B, ECX=00401004, EDX=EE1A30E0, EBX=00405DD8, ESP=0019FE8C, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	00404378	rcl     dl, 7		EAX=0000001B, ECX=00401004, EDX=EE1A30E0, EBX=00405DD8, ESP=0019FE8C, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	0040437B	sub     al, bl		EAX=0000001B, ECX=00401004, EDX=EE1A3078, EBX=00405DD8, ESP=0019FE8C, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	0040437D	dec     dl		EAX=00000043, ECX=00401004, EDX=EE1A3078, EBX=00405DD8, ESP=0019FE8C, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	0040437F	push    esi	[0019FE88]=39C01C87	EAX=00000043, ECX=00401004, EDX=EE1A3077, EBX=00405DD8, ESP=0019FE8C, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	00404380	call    004041CE		EAX=00000043, ECX=00401004, EDX=EE1A3077, EBX=00405DD8, ESP=0019FE88, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	004041CE	pop     edx	[0019FE84]=00404385	EAX=00000043, ECX=00401004, EDX=EE1A3077, EBX=00405DD8, ESP=0019FE84, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	004041CF	dec     al		EAX=00000043, ECX=00401004, EDX=00404385, EBX=00405DD8, ESP=0019FE88, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	004041D1	rcl     dh, cl		EAX=00000042, ECX=00401004, EDX=00404385, EBX=00405DD8, ESP=0019FE88, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	004041D3	sar     edx, cl		EAX=00000042, ECX=00401004, EDX=00403A85, EBX=00405DD8, ESP=0019FE88, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	004041D5	or      dl, ch		EAX=00000042, ECX=00401004, EDX=000403A8, EBX=00405DD8, ESP=0019FE88, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	004041D7	xor     al, CD		EAX=00000042, ECX=00401004, EDX=000403B8, EBX=00405DD8, ESP=0019FE88, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	004041D9	lea     edx, [ebx*8+F343409D]		EAX=0000008F, ECX=00401004, EDX=000403B8, EBX=00405DD8, ESP=0019FE88, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	004041E0	sar     dx, cl		EAX=0000008F, ECX=00401004, EDX=F5462F5D, EBX=00405DD8, ESP=0019FE88, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	004041E3	rcr     edx, 10		EAX=0000008F, ECX=00401004, EDX=F54602F5, EBX=00405DD8, ESP=0019FE88, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	004041E6	rol     dx, cl		EAX=0000008F, ECX=00401004, EDX=05EBF546, EBX=00405DD8, ESP=0019FE88, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	004041E9	sub     al, 23		EAX=0000008F, ECX=00401004, EDX=05EB546F, EBX=00405DD8, ESP=0019FE88, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	004041EB	sal     dl, cl		EAX=0000006C, ECX=00401004, EDX=05EB546F, EBX=00405DD8, ESP=0019FE88, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	004041ED	bsr     dx, dx		EAX=0000006C, ECX=00401004, EDX=05EB54F0, EBX=00405DD8, ESP=0019FE88, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	004041F1	bt      bx, 9		EAX=0000006C, ECX=00401004, EDX=05EB000E, EBX=00405DD8, ESP=0019FE88, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	004041F6	sub     bl, al		EAX=0000006C, ECX=00401004, EDX=05EB000E, EBX=00405DD8, ESP=0019FE88, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	004041F8	rcl     dh, 4		EAX=0000006C, ECX=00401004, EDX=05EB000E, EBX=00405D6C, ESP=0019FE88, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	004041FB	movzx   eax, al		EAX=0000006C, ECX=00401004, EDX=05EB000E, EBX=00405D6C, ESP=0019FE88, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	004041FE	pushad		EAX=0000006C, ECX=00401004, EDX=05EB000E, EBX=00405D6C, ESP=0019FE88, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	004041FF	mov     edx, dword ptr ds:[eax*4+404CF8]	[00404EA8]=1AEE4681	EAX=0000006C, ECX=00401004, EDX=05EB000E, EBX=00405D6C, ESP=0019FE68, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	00404206	cmc		EAX=0000006C, ECX=00401004, EDX=1AEE4681, EBX=00405D6C, ESP=0019FE68, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	00404207	sub     edx, 1AADEE33		EAX=0000006C, ECX=00401004, EDX=1AEE4681, EBX=00405D6C, ESP=0019FE68, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	0040420D	mov     byte ptr ss:[esp+8], 28	[0019FE70]=48	EAX=0000006C, ECX=00401004, EDX=0040584E, EBX=00405D6C, ESP=0019FE68, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	00404212	pushad		EAX=0000006C, ECX=00401004, EDX=0040584E, EBX=00405D6C, ESP=0019FE68, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	00404213	mov     dword ptr ss:[esp+40], edx	[0019FE88]=00405DD7	EAX=0000006C, ECX=00401004, EDX=0040584E, EBX=00405D6C, ESP=0019FE48, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	00404217	pushfd	[0019FE44]=00000206	EAX=0000006C, ECX=00401004, EDX=0040584E, EBX=00405D6C, ESP=0019FE48, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	00404218	mov     byte ptr ss:[esp+4], ch	[0019FE48]=8C	EAX=0000006C, ECX=00401004, EDX=0040584E, EBX=00405D6C, ESP=0019FE44, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	0040421C	push    dword ptr ss:[esp+8]	[0019FE4C]=00405DD7	EAX=0000006C, ECX=00401004, EDX=0040584E, EBX=00405D6C, ESP=0019FE44, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	00404220	push    dword ptr ss:[esp+48]	[0019FE88]=0040584E	EAX=0000006C, ECX=00401004, EDX=0040584E, EBX=00405D6C, ESP=0019FE40, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
main	00404224	retn    4C	[0019FE3C]=0040584E	EAX=0000006C, ECX=00401004, EDX=0040584E, EBX=00405D6C, ESP=0019FE3C, EBP=0019FF48, ESI=00405DD7, EDI=0019FE8C
```
剩下就是体力活,识别并还原handler后处理方式就跟1.8版本的一样的