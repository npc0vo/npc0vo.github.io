---
title: vmp相关4-vmp1.8初探
tags:
  - blog
  - VMProtection
description:
date: 2025-02-25
aliases:
draft:
---

[https://www.52pojie.cn/thread-713219-1-1.html](https://www.52pojie.cn/thread-713219-1-1.html)

## 准备

vmp 1.8.1 demo(因为此版本的虚拟机结构未被混淆，适合我这样的新手入门vmp)

除了虚拟化什么都不开

![[4bb5110b174e416039b59caa461bd0d1_MD5.png]]

可见ida的cfg很清晰，vmp的handler ，dispatcher 很容易识别

二进制文件在[https://www.52pojie.cn/thread-713219-1-1.html](https://www.52pojie.cn/thread-713219-1-1.html)里下载

源码

```

    include \masm32\include\masm32rt.inc

    .data
      item dd 0deadbeefh

    .code

main proc
    mov eax, item
    add eax, 12345678h
    sub eax, 12345678h
    mov item, eax    
    ret

main endp



start:   

    call main
    exit



; 

end start
```

## handler分析

在未被混淆的虚拟化下: vmp直接进入虚拟机的入口，首先压入vm_data

```
.text:00401000 sub_401000      proc near               ; CODE XREF: start↓p
.text:00401000                 push    offset VM_DATA
.text:00401005                 call    sub_40472C      ; 虚拟机入口,保存各种寄存器
```

虚拟机的入口

![[251e0e898d7ba974c1a8c16ce33a2a29_MD5.png]]

进行各种初始化工作 分配栈空间 初始化esi edi ebp esp

接下来

```
.vmp0:00404751 loc_404751:
.vmp0:00404751 mov     al, [esi]
.vmp0:00404753 movzx   eax, al
.vmp0:00404756 inc     esi  ;此时esi指向VM_DATA
.vmp0:00404757 jmp     ds:jpt_404757[eax*4] ; switch 256 cases
```

此时我们可以分析出`esi`即为`VM_EIP`

`edi` 与 `ebp` 仍然未知用途,继续分析

```
.vmp0:00404058 and     al, 3Ch
.vmp0:0040405B mov     edx, [ebp+0]
.vmp0:0040405E add     ebp, 4
.vmp0:00404061 mov     [edi+eax], edx
.vmp0:00404064 jmp     loc_404751
```

先取al后2位 此时`ebp`指向的栈的某个值赋给了`[edi+eax]` 结合vmp流程分析，第一个handler往往是进行初始化，vmp同时维护了一个`VM_CONTEXT` 通过这里的操作结合后面的操作我们可以猜测出 这就是`寄存器出栈`的handler ， `ebp`指向`VM_ESP` `edi` 执行 `VM_CONTEXT` 即vm维护的寄存器

```
.vmp0:0040462B
.vmp0:0040462B loc_40462B:             ; jumptable 00404757 cases 105,232
.vmp0:0040462B mov     eax, [esi]
.vmp0:0040462D sub     ebp, 4
.vmp0:00404630 lea     esi, [esi+4]
.vmp0:00404633 mov     [ebp+0], eax
.vmp0:00404636 jmp     loc_40400F
```

发现从`VM_DATA`取出了4字节常量,再抬高栈顶，往栈顶写入了这个常量，因此我们可以分析出这个Handler就是压入立即数

```
.vmp0:00404000 loc_404000:             ; jumptable 00404757 cases 92,122,171,247
.vmp0:00404000 mov     eax, [ebp+0]
.vmp0:00404003 add     [ebp+4], eax
.vmp0:00404006 pushf
.vmp0:00404007 pop     dword ptr [ebp+0]
```

ebp + 0 是栈顶， ebp + 4 是次栈顶。 二者相加，保存在 [ebp + 4]。 eflag 值保存在 [ebp + 0]。  
即先从栈中弹出两个数，相加后将结果压入栈中，再将eflag值压入栈中。

即`vAdd4`

```
.vmp0:00404058 and     al, 3Ch
.vmp0:0040405B mov     edx, [ebp+0]
.vmp0:0040405E add     ebp, 4
.vmp0:00404061 mov     [edi+eax], edx
.vmp0:00404064 jmp     loc_404751
```

pop一个值，然后赋给`vmContext` 因此为 `vPopReg4`

继续运行

```
.vmp0:00404584 loc_404584:             ; jumptable 00404757 cases 121,143,159,174,230
.vmp0:00404584 mov     eax, [ebp+0]
.vmp0:00404587 mov     eax, [eax]
.vmp0:00404589 mov     [ebp+0], eax
.vmp0:0040458C jmp     loc_404751
```

将栈顶的值当做地址，读取ds:edx值并存到栈顶，我们用`vReadMemDs4`来表示

```
.vmp0:004045AF and     al, 3Ch
.vmp0:004045B2 mov     edx, [edi+eax]
.vmp0:004045B5 sub     ebp, 4
.vmp0:004045B8 mov     [ebp+0], edx
.vmp0:004045BB jmp     loc_40400F
```

从寄存器指向的地址获取值，并入栈 因此为`vPushReg4`

```
.vmp0:004046B2 mov     eax, ebp
.vmp0:004046B4 sub     ebp, 4
.vmp0:004046B7 mov     [ebp+0], eax
.vmp0:004046BA jmp     loc_40400F
```

将ebp的值压入压顶 因此为 `vPushEBP`

```
.vmp0:00404069 mov     eax, [ebp+0]
.vmp0:0040406C mov     eax, ss:[eax]
.vmp0:0040406F mov     [ebp+0], eax
.vmp0:00404072 jmp     loc_404751
```

将栈顶的值存入`eax` 并读取 ss:[eax]的值 存入栈顶 因此称之为 `vReadMemSs4`

```
.vmp0:0040451A loc_40451A:             ; jumptable 00404757 cases 17,107,120
.vmp0:0040451A mov     eax, [ebp+0]
.vmp0:0040451D mov     edx, [ebp+4]
.vmp0:00404520 not     eax
.vmp0:00404522 not     edx
.vmp0:00404524 and     eax, edx
.vmp0:00404526 mov     [ebp+4], eax
.vmp0:00404529 pushf
.vmp0:0040452A pop     dword ptr [ebp+0]
```

结合之前的vmp原理学习，我们可以知道这就是vmp的逻辑运算的实现，这里实际上就是pop 了2个值 然后进行nor(~a & ~b)操作 最后的结果入栈,并且压入eflags 因此称之为`vNor4`

---

分析一下没有走到的handler

```
.vmp0:0040476F mov     al, [ebp+0]
.vmp0:00404772 sub     ebp, 2
.vmp0:00404775 add     [ebp+4], al
.vmp0:00404778 pushf
.vmp0:00404779 pop     dword ptr [ebp+0]
.vmp0:0040477C jmp     loc_40400F
.vmp0:0040477C sub_40472C endp
```

实际上是`vAdd1` ,逻辑没有变化，只是操作数改变了而已

```
.vmp0:00404041 not     dword ptr [ebp+0]
.vmp0:00404044 mov     ax, [ebp+0]
.vmp0:00404048 sub     ebp, 2
.vmp0:0040404B and     [ebp+4], ax
.vmp0:0040404F pushf
.vmp0:00404050 pop     dword ptr [ebp+0]
```

实际上是`vNor2`

```
.vmp0:00404077 mov     eax, [ebp+0]
.vmp0:0040407A mov     cl, [ebp+4]
.vmp0:0040407D sub     ebp, 2
.vmp0:00404080 shr     eax, cl
.vmp0:00404082 mov     [ebp+4], eax
.vmp0:00404085 pushf
.vmp0:00404086 pop     dword ptr [ebp+0]
```

栈顶弹出到eax，再弹出2字节到cl，然后eax shr cl，因此分析为`vShr2`

```
.vmp0:004044BE mov     ax, [esi]
.vmp0:004044C1 cwde
.vmp0:004044C2 add     esi, 2
.vmp0:004044C5 sub     ebp, 4
.vmp0:004044C8 mov     [ebp+0], eax
```

一眼丁真为 `vPush4Imm2`

```
.vmp0:004044D0 mov     al, [esi]
.vmp0:004044D2 mov     al, [edi+eax]
.vmp0:004044D5 sub     ebp, 2
.vmp0:004044D8 sub     esi, 0FFFFFFFFh
.vmp0:004044DB mov     [ebp+0], ax
```

`vPushReg1`

```
.vmp0:004044E4 movzx   eax, byte ptr [esi]
.vmp0:004044E7 sub     ebp, 2
.vmp0:004044EA mov     [ebp+0], ax
.vmp0:004044EE add     esi, 1
```

`vPush2Imm1` ,但是在栈的内存还是占2字节

```
.vmp0:00404532 mov     bp, [ebp+0]
```

`vSetBP`

```
.vmp0:0040453B mov     al, [esi]
.vmp0:0040453D cbw
.vmp0:0040453F cwde
.vmp0:00404540 sub     ebp, 4
.vmp0:00404543 lea     esi, [esi+1]
.vmp0:00404546 mov     [ebp+0], eax
```

`vPush4Imm1`,但是栈的内存占4字节

```
.vmp0:0040454E v2Shr2:                 ; jumptable 00404757 cases 11,39,61,95
.vmp0:0040454E mov     ax, [ebp+0]
.vmp0:00404552 mov     cl, [ebp+2]
.vmp0:00404555 sub     ebp, 2
.vmp0:00404558 shr     ax, cl
.vmp0:0040455B mov     [ebp+4], ax
.vmp0:0040455F pushf
.vmp0:00404560 pop     dword ptr [ebp+0
```

还是`vShr2` 用`v2Shr2`来命名

```
.vmp0:0040457C vSetEBP:                ; jumptable 00404757 cases 15,53,85,93,166
.vmp0:0040457C mov     ebp, [ebp+0]
```

`vSetEBP`

```
.vmp0:00404591 mov     ax, [ebp+0]
.vmp0:00404595 mov     dx, [ebp+2]
.vmp0:00404599 not     al
.vmp0:0040459B not     dl
.vmp0:0040459D sub     ebp, 2
.vmp0:004045A0 and     al, dl
.vmp0:004045A2 mov     [ebp+4], ax
.vmp0:004045A6 pushf
.vmp0:004045A7 pop     dword ptr [ebp+0]
```

`vNor1`

```
.vmp0:0040408E mov     esp, ebp
.vmp0:00404090 pop     edx
.vmp0:00404091 pop     ebx
.vmp0:00404092 pop     ecx
.vmp0:00404093 popf
.vmp0:00404094 pop     ebp
.vmp0:00404095 pop     edx
.vmp0:00404096 pop     eax
.vmp0:00404097 pop     ebx
.vmp0:00404098 pop     esi
.vmp0:00404099 pop     edi
.vmp0:0040409A pop     esi
.vmp0:0040409B retn
```

恢复寄存器环境

`vRet`

```
.vmp0:00404719 mov     eax, [ebp+0]
.vmp0:0040471C add     ebp, 2
.vmp0:0040471F mov     ax, ss:[eax]
.vmp0:00404723 mov     [ebp+0], ax
.vmp0:00404727 jmp     loc_404751
```

`vReadMemSs2`

```
.vmp0:00404706 mov     eax, [ebp+0]
.vmp0:00404709 mov     dx, [ebp+4]
.vmp0:0040470D add     ebp, 6
.vmp0:00404710 mov     ss:[eax], dx
```

`vWriteMemSs2` 将栈顶的值作为地址 写入2字节(次栈顶的值)

```
.vmp0:004046DA mov     al, [esi]
.vmp0:004046DC add     esi, 1
.vmp0:004046DF mov     dx, [ebp+0]
.vmp0:004046E3 add     ebp, 2
.vmp0:004046E6 mov     [edi+eax], dx
```

`vPopReg2`

```
.vmp0:004046BF mov     eax, [ebp+0]
.vmp0:004046C2 mov     edx, [ebp+4]
.vmp0:004046C5 mov     cl, [ebp+8]
.vmp0:004046C8 add     ebp, 2
.vmp0:004046CB shld    eax, edx, cl
.vmp0:004046CE mov     [ebp+4], eax
.vmp0:004046D1 pushf
.vmp0:004046D2 pop     dword ptr [ebp+0]
```

出栈两个值到`eax` `edx` shl `cl` 又入栈

`v8shl1`

```
.vmp0:00404670 mov     eax, [ebp+0]
.vmp0:00404673 mov     edx, [ebp+4]
.vmp0:00404676 add     ebp, 8
.vmp0:00404679 mov     [eax], edx
```

`vWriteMemDs4`

```
.vmp0:0040465F mov     edx, [ebp+0]
.vmp0:00404662 add     ebp, 2
.vmp0:00404665 mov     al, [edx]
.vmp0:00404667 mov     [ebp+0], ax
```

`vReadMemDs1`

```
.vmp0:00404610 mov     eax, [ebp+0]
.vmp0:00404613 mov     edx, [ebp+4]
.vmp0:00404616 mov     cl, [ebp+8]
.vmp0:00404619 add     ebp, 2
.vmp0:0040461C shrd    eax, edx, cl
.vmp0:0040461F mov     [ebp+4], eax
.vmp0:00404622 pushf
.vmp0:00404623 pop     dword ptr [ebp+0]
```

`v8shr1` 出栈4 字节 出栈 4 字节 出栈 2字节 前2个组成8字节shr val:1字节

```
.vmp0:004045D8 mov     eax, [ebp+0]
.vmp0:004045DB mov     edx, [ebp+4]
.vmp0:004045DE add     ebp, 8
.vmp0:004045E1 mov     ss:[eax], edx
```

`vWriteMemSs4` 弹出4字节->eax 弹出4字节->edx 将eax作为地址，在该地址存入edx的内容

```
.vmp0:00404568 mov     al, [esi]
.vmp0:0040456A mov     dx, [ebp+0]
.vmp0:0040456E sub     esi, 0FFFFFFFFh
.vmp0:00404571 add     ebp, 2
.vmp0:00404574 mov     [edi+eax], dl
```

`vPopReg1` 弹出2字节 但是 只将低字节存入 指定寄存器

```
.vmp0:00404508 mov     eax, [ebp+0]
.vmp0:0040450B add     ebp, 2
.vmp0:0040450E mov     ax, [eax]
.vmp0:00404511 mov     [ebp+0], ax
```

弹出2字节->ax `vReadMemDs2`

```
.vmp0:004044F6 mov     eax, [ebp+0]
.vmp0:004044F9 mov     dx, [ebp+4]
.vmp0:004044FD add     ebp, 6
.vmp0:00404500 mov     [eax], dx
```

弹出4字节->eax pop 2字节->dx 将eax指向的地址 写入dx

`vWriteMemDs2`

```
.vmp0:004044AE mov     eax, [ebp+0]
.vmp0:004044B1 mov     dl, [ebp+4]
.vmp0:004044B4 add     ebp, 6
.vmp0:004044B7 mov     [eax], dl
```

弹出4字节->eax 弹出2字节->dl 实际使用1字节

`vWriteMemDs1`

```
.vmp0:0040449C mov     edx, [ebp+0]
.vmp0:0040449F add     ebp, 2
.vmp0:004044A2 mov     al, ss:[edx]
.vmp0:004044A5 mov     [ebp+0], ax
```

弹出4字节->edx 弹出2字节-> 实际写入1字节

`vReadMemSs1`

```
.vmp0:0040475E mov     eax, [ebp+0]
.vmp0:00404761 mov     dl, [ebp+4]
.vmp0:00404764 add     ebp, 6
.vmp0:00404767 mov     ss:[eax], dl
```

`vWriteMemSs1`

```
.vmp0:004046EF mov     eax, [ebp+0]
.vmp0:004046F2 mov     cl, [ebp+4]
.vmp0:004046F5 sub     ebp, 2
.vmp0:004046F8 shl     eax, cl
.vmp0:004046FA mov     [ebp+4], eax
.vmp0:004046FD pushf
.vmp0:004046FE pop     dword ptr [ebp+0]
```

`v4shl2`

```
.vmp0:0040464D vPush2Imm2:             ; jumptable 00404757 cases 3,83,145,198,229,252
.vmp0:0040464D mov     ax, [esi]
.vmp0:00404650 lea     esi, [esi+2]
.vmp0:00404653 sub     ebp, 2
.vmp0:00404656 mov     [ebp+0], ax
```

`vPush2Imm2`

```
.vmp0:0040463B loc_40463B:             ; jumptable 00404757 cases 108,135,141,144,203
.vmp0:0040463B mov     eax, ebp
.vmp0:0040463D sub     ebp, 2
.vmp0:00404640 mov     [ebp+0], ax
```

`vPushBP`

```
.vmp0:004045FC loc_4045FC:             ; jumptable 00404757 cases 164,167,175,180,204
.vmp0:004045FC mov     ax, [ebp+0]
.vmp0:00404600 sub     ebp, 2
.vmp0:00404603 add     [ebp+4], ax
.vmp0:00404607 pushf
.vmp0:00404608 pop     dword ptr [ebp+0]
```

`vAdd2`

```
.vmp0:004045E9 loc_4045E9:             ; jumptable 00404757 cases 156,220,250,253
.vmp0:004045E9 mov     al, [esi]
.vmp0:004045EB inc     esi
.vmp0:004045EC mov     ax, [edi+eax]
.vmp0:004045F0 sub     ebp, 2
.vmp0:004045F3 mov     [ebp+0], ax
```

`vPush2Imm1`

---

至此 40多个handler都分析完毕 发现关键handler其实就几种 但是会有很多重复的 只有操作类型大小不一样

![[b0fed944283f95055aa4db5d0445748e_MD5.png]]

## vmp的还原

### 方法1: 基于ida python

由于这个vmp的结构没被混淆，比较简单，甚至可以写ida python脚本 解析VM_DATA来获得执行顺序，然后自己模拟这个虚拟机，把汇编打印出来，就是传统CTF虚拟机的解法，缺点就是比较耗费精力

基于ida python 我们可以考虑写一个python脚本自动trace这些handler

```
vPopReg4
vPushImm4
vAdd4
vPopReg4
vPopReg4
vPopReg4
...
```

通过分析handler的地址，我们能知道程序的执行流程，但是我们还需要分析出来具体的操作数,写一个简单的ida自动hook脚本

```
"""
summary: programmatically drive a debugging session

description:
  Start a debugging session, step through instructions one by one.
  Each instruction is disassembled and printed after execution.
"""

import ida_dbg
import ida_ida
import ida_lines

class MyDbgHook(ida_dbg.DBG_Hooks):
    """ Own debug hook class that implements the callback functions """
    def __init__(self):
        ida_dbg.DBG_Hooks.__init__(self)  # important
        self.steps = 0
        self.handler_map = {
            0x0404000: 'vAdd4',
            0x0404041: 'vNor2',
            0x0404058: 'vPopReg4',
            0x0404069: 'vReadMemSs4',
            0x0404077: 'vShr4',
            0x040408E: 'vRet',
            0x040449C: 'vReadMemSs1',
            0x04044AE: 'vWriteMemDs1',
            0x04044BE: 'vPushImmSx2',
            0x04044D0: 'vPushReg1',
            0x04044E4: 'vPushImm1',
            0x04044F6: 'vWriteMemDs2',
            0x0404508: 'vReadMemDs2',
            0x040451A: 'vNor4',
            0x0404532: 'vPopBP',
            0x040453B: 'vPushImmSx1',
            0x040454E: 'vShr2',
            0x0404568: 'vPopReg1',
            0x040457C: 'vPopEBP',
            0x0404584: 'vReadMemDs4',
            0x0404591: 'vNor1',
            0x04045AF: 'vPushReg4',
            0x04045C0: 'vShl1',
            0x04045D8: 'vWriteMemSs4',
            0x04045E9: 'vPushReg2',
            0x04045FC: 'vAdd2',
            0x0404610: 'vShrd4',
            0x040462B: 'vPushImm4',
            0x040463B: 'vPushBP',
            0x040464D: 'vPushImm2',
            0x040465F: 'vReadMemDs1',
            0x0404670: 'vWriteMemDs4',
            0x0404680: 'vShr1',
            0x0404698: 'vShl2',
            0x04046B2: 'vPushEBP',
            0x04046BF: 'vShld4',
            0x04046DA: 'vPopReg2',
            0x04046EF: 'vShl4',
            0x0404706: 'vWriteMemSs2',
            0x0404719: 'vReadMemSs2',
            0x040475E: 'vWriteMemSs1',
            0x040476F: 'vAdd1'
        }

    def log(self, msg):
        print(">>> %s" % msg)

    def dbg_process_start(self, pid, tid, ea, name, base, size):
        self.log("Process started, pid=%d tid=%d name=%s" % (pid, tid, name))

    def dbg_process_exit(self, pid, tid, ea, code):
        self.log("Process exited pid=%d tid=%d ea=0x%x code=%d" % (pid, tid, ea, code))

    def dbg_library_load(self, pid, tid, ea, name, base, size):
        self.log("Library loaded: pid=%d tid=%d name=%s base=%x" % (pid, tid, name, base))

    def dbg_bpt(self, tid, ea):
        self.log("Break point at 0x%x pid=%d" % (ea, tid))
        return 0

    def dbg_step_into(self):
        eip = ida_dbg.get_reg_val("EIP")
        if eip in self.handler_map:
            #print(self.handler_map[eip])
            ins=self.handler_map[eip]
            if ins=='vPopReg4' or ins =='vPushReg4':
                eax=ida_dbg.get_reg_val("EAX")
                print(f"{ins}\tR{(eax&0x3C)//4}")
            else:
                print(ins)
        elif eip-2 in self.handler_map:
            if self.handler_map[eip-2]=='vPushImm4' or self.handler_map[eip-2]=='vPushImmSx2':
                ins=self.handler_map[eip-2]
                imm=ida_dbg.get_reg_val("EAX")
                print(f"{ins}\t{hex(imm)}")

        ida_dbg.request_step_into()

    def dbg_run_to(self, pid, tid=0, ea=0):
        self.log("Runto: tid=%d, ea=%x" % (tid, ea))
        ida_dbg.request_step_into()  # Start stepping


# Remove an existing debug hook
try:
    if debughook:
        print("Removing previous hook ...")
        debughook.unhook()
except:
    pass

# Install the debug hook
debughook = MyDbgHook()
debughook.hook()

ep = ida_ida.inf_get_start_ip()
if ida_dbg.request_run_to(ep):  # Request stop at entry point
    ida_dbg.run_requests()      # Launch process
else:
    print("Impossible to prepare debugger requests. Is a debugger selected?")
```

还需要结合虚拟机入口来分析

![[ab7b6b817961f47a734e00e55c2df273_MD5.png]]执行vmhandler时,ebp初始值为esp代表VM_ESP 建议此时用excel画个栈来同时模拟 否则容易晕

```
vPopReg4	R3  ;R3=0
vPushImm4	0x765da981 
vAdd4				
vPopReg4	R7	;R7=eflags
vPopReg4	R6	;R6=0x765da981
vPopReg4	R2 	;R2=ecx
vPopReg4	R15	;R15=eflags
vPopReg4	R11	;R11=ebp
vPopReg4	R9	;R9=edx
vPopReg4	R13	;R13=eax
vPopReg4	R14	;R14=ebx
vPopReg4	R10	;R10=esp
vPopReg4	R4	;R4=edi
vPopReg4	R0	;R0=esi
vPopReg4	R1	;push/call 的ret address
vPopReg4	R8	;push的vm_data
;mov eax,dword_403000
vPushImm4	0x403000	
vReadMemDs4	
vPopReg4	R5	;R5=[0x403000]
;add eax,0x12345678
vPushImm4	0x12345678
vPushReg4	R5
vAdd4		;R5+0x12345678
vPopReg4	R7	;R7=add_eflags
vPopReg4	R13	;R13=R5+0x12345678
;sub eax,0x12345678
vPushImm4	0x12345678	;b=0x12345678
vPushReg4	R13			;a=R13
vPushEBP				;
vReadMemSs4				;两条指令连在一起相当于push a ，此时栈顶两个a
vNor4		;nor(a,a)=not(a)
vPopReg4	R1	;R1=not_flag(a)
vAdd4			
vPopReg4	R15	;R15=add_flag(not(a)+b)
vPushEBP
vReadMemSs4		; 栈顶两个not(a)+b  =c
vNor4			;nor(c,c)=not(c)=not(not(a)+b)=sub(a,b)
vPopReg4	R8	;R8=nor_flag(sub(a,b))
vPopReg4	R1	;R1=sub(a,b)=R13-0x12345678
vPushReg4	R15	;R15=add_flag(not(a)+b)
vPushEBP	
vReadMemSs4		;栈顶2个R15
vNor4			;not(R15)
vPopReg4	R5	;R5=not_flag
vPushImmSx2		0xfffff7ea  ;
vNor4			;nor(0xfffff7ea, not(R15)) = and (0x815, R15) = and(0x815, add_flag(not(a) + b))
vPopReg4	R7	;R7=nor_flag(nor(0xfffff7ea, not(R15)))
vPushReg4	R8	;R8=nor_flag(sub(a,b))
vPushReg4	R8	
vNor4			;not(R8)
vPopReg4	R12	;R12=nor_flag
vPushImmSx2	0x815
vNor4			;nor(0x815,not(R8))=and(0xfffff7ea，R8)=and(0xfffff7ea，nor_flag(sub(a,b)))
vPopReg4	R12	;R12=nor_flag
vAdd4			;add
vPopReg4	R12	;R12=add_flag
vPopReg4	R13	;R13 = and(FFFFF7EA, not_flag(not(a) + b)) + and(0x815 , add_flag(not(a), b)) = sub_flag(a, b)
;mov dword_403000,eax
vPushReg4	R1	;
vPushImm4	0x403000
vWriteMemDs4	;[0x403000]=R1=R13-0x12345678
;ret
vPushReg4	R0 	;ESI
vPushReg4	R4	;EDI
vPushReg4	R8	;eflags
vPushReg4	R14	;ebx
vPushReg4	R1	;eax
vPushReg4	R9	;edx
vPushReg4	R11	;ebp
vPushReg4	R13	;sub_flags
vPushReg4	R2	;ecx
vPushReg4	R7	;eflags
vPushReg4	R0	;esi
vRet
```

一些关键推导与化简

not(a) = nor(a, a)

and(a, b) = nor(not(a), not(b))

sub(a, b) = not(not(a) + b) = nor(not(a) + b, not(a) + b) = nor(nor(a, a) + b, nor(a, a) + b)

0xfffff7ea = not(0x815)

not_flag(a) = nor_flag(a, a)

and_flag(a, b) = nor_flag(a, not(b))

sub_flag(a, b) = and(FFFFF7EA, not_flag(not(a) + b)) + and(0x815 , add_flag(not(a), b))

其中有很多操作都是在计算eflags，特别是减法标志位的计算步骤很多,我们还原的最后的结果就是

```
    mov eax, [0x403000]
    add eax, 12345678h
    sub eax, 12345678h
    mov [0x403000], eax    
    ret
```

与源码是一致的

### 方法2 基于od trace

这是原贴的方法，基于od的trace log 可以实现相同的效果，不再赘述

记录一下原贴给的parse od tracelog的脚本

```
import re
import argparse

def INT16(s):
    try:
        return int(s, 16)
    except:
        return -1

class Trace(object):
    def __init__(self, line):
        """
        Parse Ollydbg 2.0 trace line.
        """
        try:
            mod, addr, disasm, mem_str, reg_str = line.split('\t')
            
            # print line.split('\t')

            self.module = mod
            self.address = INT16(addr)
            self.disasm = disasm           
            self.memorys = {}
            self.registers = {}

            for addr, value in re.findall('\[([0-9A-F]+)\]=([0-9A-F]+)', mem_str):
                self.memorys[INT16(addr)] = INT16(value)
            
            for reg, value in re.findall('([EABCDXSPI]+)=([0-9A-F]+)', reg_str):
                self.registers[reg] = INT16(value)

            self.prev = None
            self.next = None
        except:
            raise Exception('[!] Unknown format: %s' % line)
            

    def __str__(self):
        mem_str = ', '.join(['[%#x]=%#x' % (addr, value) 
            for addr, value in self.memorys.items()])
        
        reg_str = ', '.join(['%s=%#x' % (reg, value) 
            for reg, value in self.registers.items()])
            
        return '%s\t%#x\t%s\t%s\t%s' % (self.module,
            self.address,
            self.disasm,
            mem_str,
            reg_str)

    def __repr__(self):
        return '<Trace %#x %s>' % (self.address, self.disasm)


    def test(self, addr=None, m=None, mv=None, r=None, rv=None):
        if addr:
            if self.address != addr: return False

        if m:
            if m not in self.memorys: return False
        if mv:
            if mv not in self.memorys.values(): return False
        if m and mv:
            if self.memorys[m] != mv : return False

        if r:
            if r not in self.registers: return False
        if rv:
            if rv not in self.registers.values(): return False
        if r and rv:
            if self.registers[r] != rv: return False
        
        return True
    
    def next_trace(self, step=1):
        cur = self
        for i in range(step):
            if cur.next is None: 
                break
            cur = cur.next
        return cur

def parse_od2_trace(filename):
    traces = []
    
    buf = open(filename,'rb').read()
    lines = buf.splitlines()

    prev = None
    for line in lines:
        try:
            t = Trace(line)
            traces.append(t)
            
            if prev:
                t.prev = prev
                prev.next = t
            prev = t
        except:
            pass
    return traces

def search_trace(traces, a=None, m=None, mv=None, r=None, rv=None):
    result = []
    for t in traces:
        if t.test(a, m, mv, r, rv):
            result.append(t)
    return result

def main():
    parser = argparse.ArgumentParser(description='Parser of Ollydbg 2.0 Trace file.')
    parser.add_argument("file", help='Input Ollydbg 2.0 trace file.')
    parser.add_argument("-a", "--address", type=INT16, help='Search instructions at this address.')
    parser.add_argument("-m", "--memory-address", type=INT16)
    parser.add_argument("-mv", "--memory-value", type=INT16)
    parser.add_argument("-r",  "--register-name")
    parser.add_argument("-rv",  "--register-value", type=INT16)
    # return parser

    args = parser.parse_args()

    traces = parse_od2_trace(args.file)

    print '[+] %d traces parsed.' % len(traces)

    print '[+] Search result:'

    result = search_trace(traces, args.address, 
            args.memory_address, args.memory_value, 
            args.register_name, args.register_value)

    for t in result:
        print t 

    print '[+] %d traces found.' % len(result)

if __name__ == '__main__':

    main()
```

---

## 总结

vmp1.8.1已经是10多年前的老壳了

偶遇vmp，拼尽全力无法战胜

虽然已经是被大削的vmp1.8.1 demo壳，没有开启反调试和代码编译，分析清楚不难，但是还是很耗费精力，只还原出3,4条汇编，一晚上就过去了，呜呜呜(正式版 VMProtect 虚拟机的解释执行过程都是添加了冗余指令的, VMProtect 3.x 已经没有解释循环，使用链式寻址代替,提取字节码就更复杂了)

**人生苦短，远离vmp**