# Part1 PC BootStrap

**Exercise 1：**
　　练习1并没有什么实质性内容，就是MIT建议我们要对汇编语言有个大致的了解，这样才能让后面的学习更加方便一点。

第1代PC基于16位的Intel8088处理器，只能寻址1Mb的物理内存。因此，早先PC的物理地址空间开始于0x00000000结束于0x0000FFFF而不是0xFFFFFFFF。标示为”Low Memory”的640kb空间只能用于随机访问(RAM)。

　　从0x000A0000开始到0x000FFFFF的384Kb空间被保留用于硬件寻址比如VGA。1MB保留空间中最重要的部分是BIOS，占据了从0x000F0000到0x000FFFFF的64Kb空间。BIOS主要是进行硬件检查和初始化操作，最后从硬盘或CD-ROM上装载操作系统，并将控制权移交。



## PC物理地址空间

![image-20191217204143236](/Users/dylan/Library/Application Support/typora-user-images/image-20191217204143236.png)

早期的16位的8086 CPU只能够访问1MB的内存空间，底640KB空间称为低地址空间，是早期PC的RAM占用的内存空间，从0x000F0000 到 0x000FFFFF这64KB的地址空间由BIOS占用。

当Interl 80286(16MB)和80386(4GB)突破了地址空间的1MB的限制后，PC架构会保留低1MB的地址空间就是为了向后兼容，使得8086的设备仍然能够映射到0x000A0000 ~ 0x00100000这个地址空间，所以这个地址空间是保留不使用的。

在4GB的地址空间BIOS位于4GB地址空间的顶部,然而现在IA64 CPU支持的地址空间超过4GB，因此4GB地址空间那段BIOS还有设备映射的地址空间也被保留下来不被使用，向后兼容留给32位的设备映射，因此现代的计算机的地址空间中会存在两个洞。

开机启动首先执行BIOS，此时计算机处于实模式。新的x86处理器为了向后兼容，都保留了开机运行在实模式下，就是为了让8086上的软件能够继续运行。也就是说在新的x86处理器仍然运行实模式的操作系统，例如DOS。。。所以现在最新的笔记本发行的时候为了节省系统授权的费用，就安装的是DOS，这就是因为新的x86芯片向后兼容，保留了实模式，才使得DOS能够运行。

**Exercise 2**

>Use GDB’s si (Step Instruction) command to trace into the ROM BIOS for a few more instructions, and try to guess what it might be doing. You might want to look at Phil Storrs I/O Ports Description, as well as other materials on the 6.828 reference materials page. No need to figure out all the details - just the general idea of what the BIOS is doing first.



```
0xffff0:  ljmp $0xf000, $0xe05b
```

这是运行的第一条指令，是一条跳转指令，跳转到0xfe05b地址处。因为BIOS的起始运行的地址为`0xffff0`，后面仅仅剩下16B的空间，这些空间是不能容纳所有的BIOS指令。地址`0xfe05b`属于BIOS的内存地址空间：`0x000f0000~0x000fffff`。

```
0xfe05b: cmpl $0x0, $cs:0x6ac8
0xfe062:  jne  0xfd2e1
```



如果地址`0xf6ac8`处的值不是0x0时跳转到地址`0xfd2e1`处。地址`0xf6ac8`处的值应该很特殊，需要检查是否为0，不等于0是一种异常情况。

```
0xfe066:  xor  %dx, %dx
0xfe068:  mov  %dx %ss
0xfe06a:  mov  $0x7000, %esp
0xfe070:  mov  $0xf34d2, %edx
0xfe076:  jmp  0xfd15c
0xfd15c:  mov  %eax, %ecx
```

关闭可以屏蔽的中断, 设置方向标识位为0，表示后续的串操作比如MOVS操作，内存地址的变化方向，如果为0代表从低地址值到高地址。

```
0xfd161:  mov  $0x8f, %eax
0xfd167:  out  %al, $0x70
0xfd169:  in  $0x71, %al
```

> `in`，`out`指令是对IO端口的读写指令。CPU与外部设备通讯时，通常是通过访问，修改设备控制器中的寄存器来实现的。那么这些位于设备控制器当中的寄存器也叫做IO端口。为了方便管理，80x86CPU采用IO端口单独编址的方式，即所有设备的端口都被命名到一个IO端口地址空间中。这个空间是独立于内存地址空间的。所以必须采用和访问内存的指令不一样的指令来访问端口。
>
> 　 标准规定端口操作必须要用al寄存器作为缓冲。

操作CMOS存储器中的内容需要两个端口，一个是0x70另一个就是0x71。其中0x70可以叫做索引寄存器，这个8位寄存器的最高位是不可屏蔽中断(NMI)使能位。0x8f中最高位为1，即关闭NMI，这里的作用应该是为了让后面读0x71端口的时候的值避免NMI可能产生的影响。。低7位用于指定CMOS存储器中的存储单元地址，因此前两句指的是使能NMI，并且接下来要访问CMOS的内存地址是`0x0f`。

然后对于这个地址单元的操作，比如读或者写就可以由0x71端口完成，所以接下来的指令就是将CMOS的内存地址`0x0f`处的数据读到al寄存器中。

```
0xfd16b:  in  $0x92, %al
0xfd16d:  or  $0x2, %al
0xfd16f:  out  %al, $0x92
```



激活A20总线（激活前所有地址中的20位将被清零，具体参见[这篇文章](http://www.win.tue.nl/~aeb/linux/kbd/A20.html)），准备进入保护模式。激活A20总线之后BIOS才能访问到1MB以上的内存。由于8086是不能访问到1MB的内存空间，但是物理地址的计算会超过1MB，所以需要将超过的部分round到1MB以内，为了兼容8088，后来的CPU都默认disable A20总线，此时访存超过1MB都会round到1MB以内，只有enable A20才能访问到1MB以上的内存空间。但是在之后的boot loader程序中，计算机还是工作在实模式下，所以在boot loader之前，它肯定还会转换回实模式。这条语句应该是[测试可用的内存空间](http://kernelx.weebly.com/a20-address-line.html)。

```
0xfd171:  lidtw  %cs:0x6ab8
0xfd177:  lgdtw  %cs:0x6a74
```



`lidt`指令：加载中断向量表寄存器(IDTR)。这个指令会把从地址0xf6ab8起始的后面6个字节的数据读入到中断向量表寄存器(IDTR)中。
下一条指令把从0xf6a74为起始地址处的6个字节的值加载到全局描述符表格寄存器中GDTR中。

```
0xfd17d:  mov  %cr0, %eax
0xfd180:  or  $0x1, %eax
0xfd184:  mov  %eax, %cr0
```



计算机中包含CR0~CR3四个控制寄存器，用来控制和确定处理器的操作模式。这三条语句是将CR0寄存器的最低位(0bit)置1。CR0寄存器的0bit是PE位，启动保护位，当该位被置1，代表开启了保护模式。但是这里出现了问题，我们刚刚说过BIOS是工作在实模式之下，后面的boot loader开始的时候也是工作在实模式下，所以这里把它切换为保护模式，显然是自相矛盾。所以只能推测它在检测是否机器能工作在保护模式下。

```
0xfd187:  ljmpl  $0x8, $0xfd18f
0xfd18f:   mov  $0x10, %eax
0xfd194:  mov  %eax, %ds
0xfd196:  mov  %eax, %es
0xfd198:  mov  %eax, %ss
0xfd19a:  mov  %eax, %fs
0xfd19c:  mov  %eax, %gs
```



设置这些寄存器是按照[规定](https://en.wikibooks.org/wiki/X86_Assembly/Global_Descriptor_Table)来的，如果刚刚加载完GDTR寄存器我们必须要重新加载所有的段寄存器的值，而其中CS段寄存器必须通过长跳转指令，这样才能是GDTR的值生效。