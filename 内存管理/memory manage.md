## memory manage

进程如何组织这些区域？
上述几种内存区域中数据段、BSS和堆通常是被连续存储的——内存位置上是连续的，而代码段和栈往往会被独立存放。有趣的是，堆和栈两个区域关系很“暧昧”，他们一个向下“长”（i386体系结构中栈向下、堆向上），一个向上“长”，相对而生。但你不必担心他们会碰头，因为他们之间间隔很大（到底大到多少，你可以从下面的例子程序计算一下），绝少有机会能碰到一起。

“事实胜于雄辩”，我们用一个小例子（原形取自《User-Level Memory Management》）来展示上面所讲的各种内存区的差别与位置。

```
#include<stdio.h>

#include<malloc.h>

#include<unistd.h>

int bss_var;

int data_var0=1;

int main(int argc,char **argv)

{

  printf("below are addresses of types of process's mem\n");

  printf("Text location:\n");

  printf("\tAddress of main(Code Segment):%p\n",main);

  printf("____________________________\n");

  int stack_var0=2;

  printf("Stack Location:\n");

  printf("\tInitial end of stack:%p\n",&stack_var0);

  int stack_var1=3;

  printf("\tnew end of stack:%p\n",&stack_var1);

  printf("____________________________\n");

  printf("Data Location:\n");

  printf("\tAddress of data_var(Data Segment):%p\n",&data_var0);

  static int data_var1=4;

  printf("\tNew end of data_var(Data Segment):%p\n",&data_var1);

  printf("____________________________\n");

  printf("BSS Location:\n");

  printf("\tAddress of bss_var:%p\n",&bss_var);

  printf("____________________________\n");

  char *b = sbrk((ptrdiff_t)0);

  printf("Heap Location:\n");

  printf("\tInitial end of heap:%p\n",b);

  brk(b+4);

  b=sbrk((ptrdiff_t)0);

  printf("\tNew end of heap:%p\n",b);

return 0;

 }
```
它的结果如下

```
below are addresses of types of process's mem

Text location:

   Address of main(Code Segment):0x8048388

____________________________

Stack Location:

   Initial end of stack:0xbffffab4

   new end of stack:0xbffffab0

____________________________

Data Location:

   Address of data_var(Data Segment):0x8049758

   New end of data_var(Data Segment):0x804975c

____________________________

BSS Location:

   Address of bss_var:0x8049864

____________________________

Heap Location:

   Initial end of heap:0x8049868

   New end of heap:0x804986c
```
利用size命令也可以看到程序的各段大小，比如执行size example会得到

text data bss dec hex filename

1654 280   8 1942 796 example

但这些数据是程序编译的静态统计，而上面显示的是进程运行时的动态值，但两者是对应的。

 

通过前面的例子，我们对进程使用的逻辑内存分布已先睹为快。这部分我们就继续进入操作系统内核看看，进程对内存具体是如何进行分配和管理的。

从用户向内核看，所使用的内存表象形式会依次经历“逻辑地址”——“线性地址”——“物理地址”几种形式（关于几种地址的解释在前面已经讲述了）。逻辑地址经段机制转化成线性地址；线性地址又经过页机制转化为物理地址。（但是我们要知道Linux系统虽然保留了段机制，但是将所有程序的段地址都定死为0-4G，所以虽然逻辑地址和线性地址是两种不同的地址空间，但在Linux中逻辑地址就等于线性地址，它们的值是一样的）。沿着这条线索，我们所研究的主要问题也就集中在下面几个问题。

1.     进程空间地址如何管理？

2.     进程地址如何映射到物理内存？

3.     物理内存如何被管理？

以及由上述问题引发的一些子问题。如系统虚拟地址分布；内存分配接口；连续内存分配与非连续内存分配等。

我们下面只抽出和example有关的信息，除了前两行代表的代码段和数据段外，最后一行是进程使用的栈空间。

-------------------------------------------------------------------------------

08048000 - 08049000 r-xp 00000000 03:03 439029                               /home/mm/src/example

08049000 - 0804a000 rw-p 00000000 03:03 439029                               /home/mm/src/example

……………

bfffe000 - c0000000 rwxp ffff000 00:00 0

----------------------------------------------------------------------------------------------------------------------

每行数据格式如下：

（内存区域）开始－结束 访问权限  偏移  主设备号：次设备号 i节点  文件。

注意，你一定会发现进程空间只包含三个内存区域，似乎没有上面所提到的堆、bss等，其实并非如此，程序内存段和进程地址空间中的内存区域是种模糊对应，也就是说，堆、bss、数据段（初始化过的）都在进程空间中由数据段内存区域表示。

 

在Linux内核中对应进程内存区域的数据结构是: vm_area_struct, 内核将每个内存区域作为一个单独的内存对象管理，相应的操作也都一致。采用面向对象方法使VMA结构体可以代表多种类型的内存区域－－比如内存映射文件或进程的用户空间栈等，对这些区域的操作也都不尽相同。

vm_area_strcut结构比较复杂，关于它的详细结构请参阅相关资料。我们这里只对它的组织方法做一点补充说明。vm_area_struct是描述进程地址空间的基本管理单元，对于一个进程来说往往需要多个内存区域来描述它的虚拟空间，如何关联这些不同的内存区域呢？大家可能都会想到使用链表，的确vm_area_struct结构确实是以链表形式链接，不过为了方便查找，内核又以红黑树（以前的内核使用平衡树）的形式组织内存区域，以便降低搜索耗时。并存的两种组织形式，并非冗余：链表用于需要遍历全部节点的时候用，而红黑树适用于在地址空间中定位特定内存区域的时候。内核为了内存区域上的各种不同操作都能获得高性能，所以同时使用了这两种数据结构。

## Requirements


1. 验证不同进程的相同的地址可以保存不同的数据。
   (1) 在VS中，设置固定基地址，编写两个不同可执行文件。同时运行这两个文件。然后使用调试器附加到两个程序的进程，查看内存，看两个程序是否使用了相同的内存地址；
   (2) 在不同的进程中，尝试使用VirtualAlloc分配一块相同地址的内存，写入不同的数据。再读出。
2. (难度较高)配置一个Windbg双机内核调试环境，查阅Windbg的文档，了解
   (1) Windbg如何在内核调试情况下看物理内存，也就是通过物理地址访问内存
   (2) 如何查看进程的虚拟内存分页表，在分页表中找到物理内存和虚拟内存的对应关系。然后通过Windbg的物理内存查看方式和虚拟内存的查看方式，看同一块物理内存中的数据情况

## Experiment

### 虚拟内存

在一般计算机中 一个物理地址是唯一的，同一个物理地址下的数据是相同的

在虚拟内存管理当中，每个进程的地址是独立的，实际在物理地址中是两个不同的存储空间，但是虚拟的内存地址是相同的，不同进程，相同内存地址，在内核部分映射成为不同的物理地址，这就是为什么进程能在计算机上相互隔离运行的原因

### 实验过程


#### 使用VirtualAlloc分配一块相同地址的内存，写入不同的数据再读出

##### 代码

```
#include <windows.h>
#include <tchar.h>
#include <stdio.h>
#include <stdlib.h>             // For exit

int main() {
	printf("Demo A virtualalloc\n");

	LPVOID lpvBase;               // Base address of the test memory
	LPTSTR lpPtr;                 // Generic character pointer
	BOOL bSuccess;                // Flag
	DWORD i;                      // Generic counter
	SYSTEM_INFO sSysInfo;         // Useful information about the system

	GetSystemInfo(&sSysInfo);     // Initialize the structure.

	DWORD dwPageSize = sSysInfo.dwPageSize;
	// dwPageSize = 4096
	printf("%d\n", dwPageSize);

	// Reserve pages in the virtual address space of the process.
	int PAGELIMIT = 1;

	lpvBase = VirtualAlloc(
		(LPVOID)0x40000000,                 // System selects address
		PAGELIMIT*dwPageSize, // Size of allocation
		MEM_RESERVE | MEM_COMMIT,          // Allocate reserved pages
		PAGE_READWRITE);       // Protection = no access
	if (lpvBase == NULL)
	{
		_tprintf(TEXT("Error! %s with error code of %ld.\n"), TEXT("VirtualAlloc reserve failed."), GetLastError());
		exit(0);
	}

	lpPtr = (LPTSTR)lpvBase;

	// Write to memory.
	for (i = 0; i < PAGELIMIT*dwPageSize; i++) {
		lpPtr[i] = 'a';
	}

	// Read from memory
	for (i = 0; i < PAGELIMIT*dwPageSize; i++) {
		printf("%c", lpPtr[i]);
	}

	bSuccess = VirtualFree(
		lpvBase,       // Base address of block
		0,             // Bytes of committed pages
		MEM_RELEASE);  // Decommit the pages

	_tprintf(TEXT("\nRelease %s.\n"), bSuccess ? TEXT("succeeded") : TEXT("failed"));

	return 0;
}
```

##### 测试

- 调试A_demo，观察字符a写入过程
- `61h`是a的ascii值的16进制
- `00411929  mov         byte ptr [eax],61h  
` 代码执行完这句后，去查看eax寄存器的值，存的是a所在的地址

    <img src="./img/finda.png">

    <img src="./img/finda2.png">

    <img src="./img/memory3.png">

- 此时调试B_demo，b也可以写入

    <img src="./img/memoryb1.png">

- 相同内存地址存入了不同的数据，证明不同进程相同内存地址可以保存不同的数据

    <img src="./img/memoryab.png">

### Windbg双机内核调试

#### 思路

- 由于我们直接调试的操作系统内核，所以需要两台计算机安装两个Windows，然后连个计算机使用串口进行连接
- 所以我们需要再虚拟机中安装一个Windows（安装镜像自己找，XP就可以），然后通过虚拟串口和host pipe连接的方式，让被调试系统和windbg链接，windbg可以调试
- 使用Windbg  内核调试 VirtualBox 关键字搜索，能找到很多教程
- 如果决定Windows虚拟机太重量级了，可以用Linux虚拟机+gdb也能进行相关的实验，以gdb 远程内核调试 为关键字搜索，也能找到很多教程

#### 环境

- 主机：windows 10 企业版 1909 64位
- 虚拟机：windows 7 专业版 sp1_vl_build_x64
- Windows Software Development Kit - Windows 10.0.19041

#### 环境搭建

将主机称为Host端，windbg运行在主机上，通过虚拟串口连接虚拟机。将虚拟机称为Guest端，运行着待调试的系统

##### Guest端配置

- 为了让Windos支持内核调试，需要配置启动参数。首先需要对虚拟机配置虚拟串口，目的是为了建立host到guest的调试通信连接。如下图所示，选择com1并且映射成为\\.pip\com_1
  
  <img src="./img/com-config.png">

  - 注意不要选择 连接至现有通道或套接字 因为目前还没有建立管道。否则启动虚拟机时会报错误`不能为虚拟电脑打开一个新任务`
- 启动虚拟机，进入Window内部进行配置。以管理员身份启动CMD，输入以下命令
  
  <img src="./img/bcdedit.png">

  ```cmd
  # 设置端口1
  bcdedit /dbgsettings serial baudrate:115200 debugport:1
  
  # 复制一个开机选项，命名为“DebugEntry”，可任意命名
  
  # 增加一个开机引导项
  bcdedit /copy {current} /d DebugEntry

  # 激活debug
  bcdedit /displayorder {current} {替换第二个命令显示的UUID}
  bcdedit /debug {替换第二个命令显示的UUID} on
  ```

  [BCDEdit](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/bcdedit-command-line-options)用于管理启动配置数据（BCD）的命令行工具
- 重新启动系统，在开机的时候选择DebugEntry[启用调试程序]，能够正常启动系统
  
  <img src="./img/debugentry.png">

