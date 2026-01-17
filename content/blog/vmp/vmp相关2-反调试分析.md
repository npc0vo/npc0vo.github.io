---
title: vmp相关2-反调试分析
tags:
  - blog
  - VMProtection
description:
date: 2025-02-19
aliases:
draft:
---
[https://bbs.kanxue.com/thread-282244.htm](https://bbs.kanxue.com/thread-282244.htm)

## 定位源码

根据文本定位，全局搜索debugger可以找到这个函数

![[a2369a4f5ebc068fdca2c63e733b83de_MD5.png]]

然后shift+f12查找 mtDebuggerFound的引用，就可以定位到反调试函数的具体位置

## 反调试函数分析

```
if (!os_build_number) {
    if (data.options() & LOADER_OPTION_CHECK_DEBUGGER) {
        LoaderMessage(mtDebuggerFound);
        return LOADER_ERROR;
    }
    tmp_loader_data->set_is_debugger_detected(true);
}
```

![[ef2849ee32879af6348fd5327cc40597_MD5.png]]

判断os_build_number运用两种方法peb或者直接从ntdll中解析

感觉没有什么意义，我认为只是初始化反调试(初始化一系列syscall地址)，防止修改后的ntdll导致反调试失效

```
if (peb->BeingDebugged) {
    LoaderMessage(mtDebuggerFound);
    return LOADER_ERROR;
}
```

经典PEB的BeingDebugged标志位反调试

```
if (NT_SUCCESS(reinterpret_cast<tNtQueryInformationProcess *>(syscall | sc_query_information_process)(process, ProcessDebugPort, &debug_object, sizeof(debug_object), NULL)) && debug_object != 0) {
    LoaderMessage(mtDebuggerFound);
    return LOADER_ERROR;
}
```

```
if (NT_SUCCESS(reinterpret_cast<tNtQueryInformationProcess *>(syscall | sc_query_information_process)(process, ProcessDebugObjectHandle, &debug_object, sizeof(debug_object), reinterpret_cast<PULONG>(&debug_object))) 
    || debug_object == 0) {
    LoaderMessage(mtDebuggerFound);
    return LOADER_ERROR;
}
```

调用`NtQueryInformationProcess`查询

[https://blog.csdn.net/Simon798/article/details/100886945](https://blog.csdn.net/Simon798/article/details/100886945)

[https://learn.microsoft.com/zh-cn/windows/win32/api/processthreadsapi/ne-processthreadsapi-process_information_class](https://learn.microsoft.com/zh-cn/windows/win32/api/processthreadsapi/ne-processthreadsapi-process_information_class)

从内核对象`EPROCESS`中获取`ProcessDebugPort``ProcessDebugObjectHandle`进行检测 (这里没有对debugflags的检测)

进程处于被调试状态时，`ProcessDebugPort = 0xffffffff`

没有调试器时：`ProcessDebugPort = 0`

调试的时候会生成调试对象（DebugObject),

- 若进程处于调试状态的时候，调试对象句柄值就存在，
- 处于非调试状态的时候，调试对象句柄值为NULL.

检测Debug Flags(调试标志）的值可以判断进程处于调试状态，通过函数第三个参数就可以获取调试标志的值，

- 若为0，则处于调试状态，
- 若为1,则处于非调试状态。

在OD中调试，则debugFlags为`0x0`

```
__kernel_entry NTSTATUS NtQueryInformationProcess(
  [in]            HANDLE           ProcessHandle,//查询信息的进程的句柄。
  [in]            PROCESSINFOCLASS ProcessInformationClass,
  [out]           PVOID            ProcessInformation,//返回进程信息的缓冲区的指针
  [in]            ULONG            ProcessInformationLength,
  [out, optional] PULONG           ReturnLength
);
```

- `ProcessInformationClass`：进程信息类，指定你希望查询什么类型的信息。这是一个枚举值，可以是如 `ProcessBasicInformation`（基本信息）等。这里列举一些常用的

- `ProcessBasicInformation`：查询基本信息，如进程ID，父进程ID等。
- `ProcessDebugPort`：检查进程是否正在被调试。
- `ProcessWow64Information`：如果进程是运行在WOW64（Windows 32-bit On Windows 64-bit）环境中，则返回一个非零值。
- `ProcessImageFileName`：获取进程的图像文件名。
- `ProcessBreakOnTermination`：查询或设置Debug Break在进程终止时是否会触发。

测试代码

```
#include <windows.h>
#include <ntstatus.h>
#include <stdio.h>

#define NT_SUCCESS(Status) (((NTSTATUS)(Status)) >= 0)

typedef LONG KPRIORITY;

typedef struct _PROCESS_BASIC_INFORMATION {
    NTSTATUS ExitStatus;
    PVOID PebBaseAddress;
    ULONG_PTR AffinityMask;
    KPRIORITY BasePriority;
    ULONG_PTR UniqueProcessId;
    ULONG_PTR InheritedFromUniqueProcessId;
} PROCESS_BASIC_INFORMATION, * PPROCESS_BASIC_INFORMATION;

typedef NTSTATUS(WINAPI* NT_QUERY_INFORMATION_PROCESS)(HANDLE, UINT, PVOID, ULONG, PULONG);

NT_QUERY_INFORMATION_PROCESS NtQueryInfoProcess = NULL;

BOOL IsDebugged() {
    PROCESS_BASIC_INFORMATION pbi;
    DWORD debugPort = 0;
    //ULONG returnLength;

    // Check ProcessDebugPort
    NtQueryInfoProcess(GetCurrentProcess(), 7/*ProcessDebugPort*/, &debugPort, sizeof(debugPort), NULL);
    if (debugPort != 0)
        return TRUE; // Debugged

    // Check ProcessDebugObjectHandle
    HANDLE debugObject = NULL;
    NtQueryInfoProcess(GetCurrentProcess(), 30/*ProcessDebugObjectHandle*/, &debugObject, sizeof(debugObject), NULL);
    if (debugObject != NULL)
        return TRUE; // Debugged

    // Check ProcessDebugFlags
    DWORD debugFlags = 1;
    NtQueryInfoProcess(GetCurrentProcess(), 31/*ProcessDebugFlags*/, &debugFlags, sizeof(debugFlags), NULL);
    if (debugFlags !=1) // 修改这里
        return TRUE; // Debugged

    return FALSE; // Not debugged
}

LRESULT CALLBACK WindowProcedure(HWND hwnd, UINT message, WPARAM wparam, LPARAM lparam)
{
    switch (message)
    {
    case WM_CREATE:
    {
        NtQueryInfoProcess = (NT_QUERY_INFORMATION_PROCESS)GetProcAddress(GetModuleHandle(TEXT("ntdll")), "NtQueryInformationProcess");
        CreateWindow(TEXT("BUTTON"), TEXT("Check Debugger"),
            WS_VISIBLE | WS_CHILD,
            10, 10, 120, 25,
            hwnd, (HMENU)1, NULL, NULL);
    }
    break;
    case WM_COMMAND:
    {
        if (LOWORD(wparam) == 1) {
            if (IsDebugged())
                MessageBox(hwnd, TEXT("Debugging"), TEXT("Status"), MB_OK);
            else
                MessageBox(hwnd, TEXT("No Debugger"), TEXT("Status"), MB_OK);
        }
    }
    break;
    case WM_CLOSE:
        PostQuitMessage(0);
        break;
    }
    return DefWindowProc(hwnd, message, wparam, lparam);
}

int WINAPI WinMain(HINSTANCE hInst, HINSTANCE, LPSTR, int nCmdShow)
{
    MSG msg;
    WNDCLASSW wc = { 0 };

    wc.lpszClassName = L"Class";
    wc.hInstance = hInst;
    wc.lpfnWndProc = WindowProcedure;
    wc.hIcon = LoadIcon(NULL, IDI_APPLICATION);
    wc.hCursor = LoadCursor(NULL, IDC_ARROW);

    RegisterClassW(&wc);
    CreateWindowW(wc.lpszClassName, L"Debugger Check",
        WS_OVERLAPPEDWINDOW | WS_VISIBLE,
        100, 100, 320, 240, NULL, NULL, hInst, NULL);

    while (GetMessage(&msg, NULL, 0, 0)) {
        DispatchMessage(&msg);
    }

    return (int)msg.wParam;
}
```

```
tNtQuerySystemInformation *nt_query_system_information = reinterpret_cast<tNtQuerySystemInformation *>(LoaderGetProcAddress(ntdll, reinterpret_cast<const char *>(FACE_NT_QUERY_INFORMATION_NAME), true));
if (nt_query_system_information) {
    SYSTEM_KERNEL_DEBUGGER_INFORMATION info;
    NTSTATUS status = nt_query_system_information(SystemKernelDebuggerInformation, &info, sizeof(info), NULL);
    if (NT_SUCCESS(status) && info.DebuggerEnabled && !info.DebuggerNotPresent) {
        LoaderMessage(mtDebuggerFound);
        return LOADER_ERROR;
    }
```

调用`NtQuerySystemInformation`

```
__kernel_entry NTSTATUS NtQuerySystemInformation(
  [in]            SYSTEM_INFORMATION_CLASS SystemInformationClass,
  [in, out]       PVOID                    SystemInformation,
  [in]            ULONG                    SystemInformationLength,
  [out, optional] PULONG                   ReturnLength
);
```

[https://learn.microsoft.com/zh-cn/windows/win32/api/winternl/nf-winternl-ntquerysysteminformation](https://learn.microsoft.com/zh-cn/windows/win32/api/winternl/nf-winternl-ntquerysysteminformation)

第一个参数`SystemInformationClass`传入`SystemKernelDebuggerInformation(0x23)`, 第二个参数`SystemInformation`为结构体`SYSTEM_KERNEL_DEBUGGER_INFORMATION`的地址, `SYSTEM_KERNEL_DEBUGGER_INFORMATION.DebuggerEnabled == 1`则表明系统处于调试状态.

这里就是检测是否有内核调试器存在以及检测系统是否开启了调试模式

```
status = nt_query_system_information(SystemModuleInformation, &buffer, 0, &buffer_size);
if (buffer_size) {
    buffer = reinterpret_cast<SYSTEM_MODULE_INFORMATION *>(LoaderAlloc(buffer_size * 2));
    if (buffer) {
        status = nt_query_system_information(SystemModuleInformation, buffer, buffer_size * 2, NULL);
        if (NT_SUCCESS(status)) {
            for (size_t i = 0; i < buffer->Count && !is_found; i++) {
                SYSTEM_MODULE_ENTRY *module_entry = &buffer->Module[i];
                for (size_t j = 0; j < 5 ; j++) {
                    const char *module_name;
                    switch (j) {
                    case 0:
                        module_name = reinterpret_cast<const char *>(FACE_SICE_NAME);
                        break;
                    case 1:
                        module_name = reinterpret_cast<const char *>(FACE_SIWVID_NAME);
                        break;
                    case 2:
                        module_name = reinterpret_cast<const char *>(FACE_NTICE_NAME);
                        break;
                    case 3:
                        module_name = reinterpret_cast<const char *>(FACE_ICEEXT_NAME);
                        break;
                    case 4:
                        module_name = reinterpret_cast<const char *>(FACE_SYSER_NAME);
                        break;
                    }
                    if (Loader_stricmp(module_name, module_entry->Name + module_entry->PathLength, true) == 0) {
                        is_found = true;
                        break;
                    }
                }
            }
        }
        LoaderFree(buffer);
    }
}
```

将 `SystemModuleInformation`（`SYSTEM_INFORMATION_CLASS` 的值为 11）作为参数传递给 `NtQuerySystemInformation`，代码的功能将从检测内核调试器转变为获取系统中加载的模块（如驱动程序和 DLL）的信息。

遍历到模块信息后进行字符串匹配 \取得系统模块列表后，将模块名与解密后的字符串"sice.sys"、"siwvid.sys"、"ntice.sys"、"iceext.sys"和"syser.sys"进行比较，这些都是比较古老的调试器

```
			if (sc_set_information_thread)
				reinterpret_cast<tNtSetInformationThread *>(syscall | sc_set_information_thread)(thread, ThreadHideFromDebugger, NULL, 0);
```

`NtSetInformationThread` 函数的调用，参数解释如下：

- `thread`：目标线程的句柄。
- `ThreadHideFromDebugger`：这是 `THREAD_INFORMATION_CLASS` 枚举的一个值，表示要设置的线程属性为“隐藏线程”。

可以禁止线程产生调试事件。

```
// check breakpoint
uint8_t *ckeck_list[] = { reinterpret_cast<uint8_t*>(create_section),
                        reinterpret_cast<uint8_t*>(open_file),
                        reinterpret_cast<uint8_t*>(map_view_of_section),
                        reinterpret_cast<uint8_t*>(unmap_view_of_section),
                        reinterpret_cast<uint8_t*>(close) };
for (i = 0; i < _countof(ckeck_list); i++) {
    if (*ckeck_list[i] == 0xcc) {
        if (data.options() & LOADER_OPTION_CHECK_DEBUGGER) {
            LoaderMessage(mtDebuggerFound);
            return LOADER_ERROR;
        }
        tmp_loader_data->set_is_debugger_detected(true);
    }
}
```

在 x86 架构中，`0xCC` 是单步调试指令（`int3`）的机器码

`ckeck_list`，用于存储一系列函数的入口地址。

因此这段代码的作用就是检查特定函数的入口点是否被设置为软件断点

```
	tNtProtectVirtualMemory *virtual_protect = NULL;
	if (sc_virtual_protect) {
		for (i = 0; i < data_section_info_size; i += sizeof(SECTION_INFO)) {
			SECTION_INFO *section_info = reinterpret_cast<SECTION_INFO *>(image_base + data_section_info + i);

			DWORD protect, old_protect;
			protect = (section_info->Type & IMAGE_SCN_MEM_EXECUTE) ? PAGE_EXECUTE_READWRITE : PAGE_READWRITE;
			void *address = image_base + section_info->Address;
			SIZE_T size = section_info->Size;
			if (!NT_SUCCESS(reinterpret_cast<tNtProtectVirtualMemory *>(syscall | sc_virtual_protect)(process, &address, &size, protect, &old_protect))) {
				LoaderMessage(mtInitializationError, VIRTUAL_PROTECT_ERROR);
				return LOADER_ERROR;
			}
			if (old_protect & PAGE_GUARD) {
				if (data.options() & LOADER_OPTION_CHECK_DEBUGGER) {
					LoaderMessage(mtDebuggerFound);
					return LOADER_ERROR;
				}
				tmp_loader_data->set_is_debugger_detected(true);
			}
		}
	} 
```

调用 `NtProtectVirtualMemory` 系统调用来修改内存区域的保护属性

Dbg允许设置一个内存访问/写入断点，这种类型的断点是通过页面保护来实现的, 页面保护是通过PAGE_GUARD页面保护修改符来设置的，如果访问的内存地址是受保护页面的一部分，将会产生一个 STATUS_GUARD_PAGE_VIOLATION(0x80000001)异常。如果进程被OllyDbg调试并且受保护的页面被访问，将不会抛 出异常，访问将会被当作内存断点来处理,.

这里的反调试就是逐个修改页面保护属性 检测原来的页面保护符是否包含PAGE_GUARD 如果存在 说明写入了读写断点 这样写既使读写断点失效 又可以检测出debugger

```
if (data.options() & LOADER_OPTION_CHECK_DEBUGGER) {
    typedef BOOL (WINAPI tCloseHandle)(HANDLE hObject);
    tCloseHandle *close_handle = reinterpret_cast<tCloseHandle *>(LoaderGetProcAddress(kernel32, reinterpret_cast<const char *>(FACE_CLOSE_HANDLE_NAME), true));
    if (close_handle) {
        __try {
            if (close_handle(HANDLE(INT_PTR(0xDEADC0DE)))) {
                LoaderMessage(mtDebuggerFound);
                return LOADER_ERROR;
            }
        } __except(EXCEPTION_EXECUTE_HANDLER) {
            LoaderMessage(mtDebuggerFound);
            return LOADER_ERROR;
        }
    }
```

调用 CloseHandle 释放一个无效句柄，如果没有被调试，返回FALSE，GetLastError = ERROR_INVALID_HANDLE；但如果被调试，那么系统将抛出异常 C0000008H。

```
size_t drx;
uint64_t val;
CONTEXT *ctx;
__try {
    __writeeflags(__readeflags() | 0x100);
     val = __rdtsc();
     __nop();
    LoaderMessage(mtDebuggerFound);
    return LOADER_ERROR;
} __except(ctx = (GetExceptionInformation())->ContextRecord,
    drx = (ctx->ContextFlags & CONTEXT_DEBUG_REGISTERS) ? ctx->Dr0 | ctx->Dr1 | ctx->Dr2 | ctx->Dr3 : 0,
    EXCEPTION_EXECUTE_HANDLER) {
    if (drx) {
        LoaderMessage(mtDebuggerFound);
        return LOADER_ERROR;
    }
}
}
```

TrapFlag 与 硬件断点检测 两个反调写到了一起

- `__readeflags()`：读取当前的标志寄存器（EFLAGS）。
- `__writeeflags()`：写入新的标志寄存器值。
- `0x100`：对应标志寄存器中的陷阱标志（TF，Trap Flag）。设置 TF 后，CPU 会在每条指令执行后触发单步中断（`#DB` 异常）。
- `__rdtsc()`：读取时间戳计数器（TSC）的值。
- `__nop()`：执行一条空操作指令（NOP）。
- 这两行代码的目的是触发单步中断（`#DB` 异常），因为之前设置了陷阱标志（TF）。
- 正常程序执行，直接进入异常处理，然后正常运行。如果有调试器附加，调试器会捕获这个异常并处理,如果调试器忽略异常,步入下一步就会被检测到.如果调试器pass异常交给SEH处理，SEH处理函数会检测硬件断点，进行反调试

![[6b62a5ce80ac756a80ae8038eb448ce0_MD5.png]]

check memory crc 防止patch

分析源码可以看到一些查询api使用的是硬编码在程序里的syscall,hook可能无法正常绕过

## 反反调试

### scyllaHide

[https://github.com/x64dbg/ScyllaHide](https://github.com/x64dbg/ScyllaHide)

### TitanHide

[https://github.com/mrexodia/TitanHide](https://github.com/mrexodia/TitanHide)