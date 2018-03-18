# Lab 1

## Exercise 1

Appendix A中将生成`ucore.img`的相关代码列出来了， 下面只列出一部分代码，具体代码见Appendix A。

### 第一部分

```makefile
+ cc kern/libs/stdio.c
+ cc kern/debug/kdebug.c
+ cc kern/debug/kmonitor.c
+ cc kern/debug/panic.c
+ cc kern/driver/clock.c
+ cc kern/driver/console.c
+ cc kern/driver/intr.c
+ cc kern/driver/picirq.c
+ cc kern/trap/trap.c
+ cc kern/trap/trapentry.S
+ cc kern/trap/vectors.S
+ cc kern/mm/pmm.c
```

对应的Makefile代码为

```makefile
KINCLUDE	+= kern/debug/ \
			   kern/driver/ \
			   kern/trap/ \
			   kern/mm/
KCFLAGS		+= $(addprefix -I,$(KINCLUDE))
KSRCDIR		+= kern/init \
			   kern/libs \
			   kern/debug \
			   kern/driver \
			   kern/trap \
			   kern/mm
			   
$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))

```

这一部分的代码首先将`kern`中的文件夹加入`KSRDIR`和`KINCLUDE`中，然后使用`listf_cc`将这些需要编译的文件加进列表中，进而使用`add_files_cc`将这些全部编译。这一部分使用的编译参数有

| Argument             | Meaning                                  |
| -------------------- | ---------------------------------------- |
| -I dir               | Add the directory dir to the list of directories to be searched for header files. |
| -fno-builtin         | Don't recognize built-in functions that do not begin with __builtin_ as prefix. |
| -Wall                | Enable all warnings.                     |
| -ggdb                | Produce debugging information for use by GDB. |
| -m32                 | Produce 32-bit codes.                    |
| -gstabs              | Produce debugging information in stabs format. |
| -nostdinc            | Do not search the standard system directories for header files. Only the directories you have specified with '-I' options (and the directory of the current file, if appropriate) are searched. |
| -fno-stack-protector | Do not produce codes for checking overfitting of buffer. |
| -c file              | Input c source file                      |
| -o file              | Output intermediate build file           |

产生的文件放在`obj/kern/` 中。

### 第二部分

```makefile
+ cc libs/printfmt.c
+ cc libs/string.c
```

这一部分对应的Makefile代码为

```makefile
LIBDIR	+= libs
$(call add_files_cc,$(call listf_cc,$(LIBDIR)),libs,)
```

与前一部分类似地，这一部分的代码将`libs`中的文件全部编译。产生的文件放在`obj/libs/`中。

### 第三部分

```
+ ld bin/kernel
```

这部分是将之前编译好的`*.o`文件链接起来，对应的Makefile代码如下：

```makefile
# create kernel target
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)
```

这里的`-T` 参数表示让连接器使用指定的脚本，`-S`参数表示移除所有符号和重定位信息，`-nostdlib`表示不使用标准库。

### 第四部分

```
+ cc boot/bootasm.S
+ cc boot/bootmain.c
```

这部分对应的Makefile代码为

```makefile
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
```

`-Os`优化了代码大小，`-nostdinc`不使用C标准库。	

### 第五部分

```makefile
+ cc tools/sign.c
```

这部分对应的Makefile代码为

```Makefile
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)
```

首先将`tools/sign.c`编译为`obj/sign/tools/sign.oa`，最终产生`bin/sign`。

### 第六部分

```
+ ld bin/bootblock
```

直接生成`ucore.img`的代码是：

```makefile
# create bootblock
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

bootblock = $(call totarget,bootblock)

$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

$(call create_target,bootblock)

# create ucore.img
UCOREIMG	:= $(call totarget,ucore.img)

$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)
```

这一部分首先将`boot`目录下的文件加入`bootfile`，这里的参数前面已经提及。

然后将`obj/boot`下的`*.o`文件链接起来，产生`obj/bootblock.o`。`-N`设置代码段和数据段均可读写， `-e start`制定了入口为start，`-Ttext 0x7C00`指定代码开始于`0x7C00` 。`-m    elf_i386`产生适合于 elf_i386的代码。

`dd`命令表示用指定大小的块拷贝一个文件，并在拷贝的同时进行指定的转换。其中`if`命令表示输入文件，`of`命令表示输入文件，`count=10000`表示拷贝10000个块，`count-notrunc`不截断输出文件。这段代码将`bootblock`和`kernel`拷贝到`ucore.img`。

**Q：一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?** 

**A：512字节的主引导扇区以0x55AA结尾。** 

## Exercise 2

1. 删去`tools/gdbinit`中的continue，然后使用si命令单步跟踪即可。


2. 将`tools/gdbinit`文件替换如下

```
define hook-stop
x/i $pc
end
file bin/kernel
target remote :1234
break *0x7c00
continue
```

这样就可以从`0x7C00`处开始单步跟踪了。

我的机器（一台远程ubuntu 16.04LTS实例）因为不明原因不能直接使用`make debug`进行跟踪，因此我将Makefile的一部分改成

```makefile
debug: $(UCOREIMG)
	$(V)$(QEMU) -S -s -parallel stdio -hda $< -serial null &
	# $(V)sleep 2
	# $(V)$(TERMINAL) -e "gdb -q -tui -x tools/gdbinit"
	
debug-nox: $(UCOREIMG)
	$(V)$(QEMU) -S -s -serial mon:stdio -hda $< -nographic &
	# $(V)sleep 2
	# $(V)$(TERMINAL) -e "gdb -q -x tools/gdbinit"
```

然后使用`gdb -q -tui -x tools/gdbinit`进行调试。会停在`0x7C00`处。

3. 可以通过不断执行si，然后观察指令，可以观察到和bootblock.asm中是一模一样的，和bootasm.S的区别只是xor/xorw两种记法的不同。
4. 我将断点设在了`<lab1_print_cur_status+50>`，反汇编的结果为`call   0x100322 <cprintf>`。这与附近的C代码`cprintf("%d: @ring %d\n", round, reg1 & 3);`相符。

## Exercise 3

1. 为了保证80386（准确说式80286之后的处理器）与8086/8088在实模式下的兼容性，操作系统需要在启动的时候将键盘控制器上的A20线置于高电位，使得全部32条地址线可用，以访问4G的内存空间。

```asm
seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port

seta20.2:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60   
```

通过等待键盘控制起不忙，发送写8042端口的指令，然后再等待不忙，然后输出0xdf即打开A20。

2. 初始化lgdt表是通过`lgdt gdtdesc`完成的。

3. 进入使能和保护模式一共有三步

   1. 打开A20

   2. 使用

      ```asm
      movl %cr0, %eax
      orl $CR0_PE_ON, %eax
      movl %eax, %cr0
      ```
      将`cr0`寄存器打开。

   3. 通过`ljmp $PROT_MODE_CSEG, $protcseg`切换到32-bit模式。

## Exercise 4

```c
static void
readsect(void *dst, uint32_t secno) {
    // wait for disk to be ready
    waitdisk();

    outb(0x1F2, 1);                         // count = 1
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors

    // wait for disk to be ready
    waitdisk();

    // read a sector
    insl(0x1F0, dst, SECTSIZE / 4);
}
```

等待硬盘准备就绪。结合硬盘访问概述，我们知道这段代码设置硬盘数量为1，第28位为0表示访问主盘。然后1F7处输出0x20命令，读取扇区。最后一行为读取到dst。

```c
static void
readseg(uintptr_t va, uint32_t count, uint32_t offset) {
    uintptr_t end_va = va + count;

    // round down to sector boundary
    va -= offset % SECTSIZE;

    // translate from bytes to sectors; kernel starts at sector 1
    uint32_t secno = (offset / SECTSIZE) + 1;

    // If this is too slow, we could read lots of sectors at a time.
    // We'd write more to memory than asked, but it doesn't matter --
    // we load in increasing order.
    for (; va < end_va; va += SECTSIZE, secno ++) {
        readsect((void *)va, secno);
    }
}
```

这个函数用于读取任意长的segment，将段切成sectors，然后调用readsect即可。重点在于secno应该从1开始，这是由于0扇区被引导占用了，因此ELF从1开始。

```c
/* bootmain - the entry of bootloader */
void
bootmain(void) {
    // read the 1st page off disk
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // is this a valid ELF?
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph;

    // load each program segment (ignores ph flags)
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();

bad:
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);

    /* do nothing */
    while (1);
}
```

这段代码是`bootloader`的入口，首先读取第一页，然后通过魔数判断是否是合法的ELF，否则就向0x8A00输出一些魔数，并陷入死循环。然后加载每一个程序的段，并转入ELF的进入点。

## Exercise 5

首先获取ebp和eip，然后枚举函数栈，打印ebp，eip，然后打印ebp之后的四个参数，然后打印函数名和行号，最后弹栈。 ebp指的是基址，eip则是指令指针，之后四个则是函数的参数。由于这是从bootmain被调用开始的，而调用bootmain的是汇编代码，因此没有C语言函数名，故为`<unknown>`。

## Exercise 6

1. 中断向量表一个表项占用8字节，其中2-3字节是段选择子，0-1字节和6-7字节拼成位移，

两者联合便是中断处理程序的入口地址。

2. ​

```c
extern uintptr_t __vectors[];
int i;
for(i = 0;i < sizeof(idt) / sizeof(struct gatedesc); i ++) {
    if (i == T_SYSCALL) 
        SETGATE(idt[i], 1, GD_KTEXT, __vectors[i], DPL_USER)
    else if (i == T_SWITCH_TOK)
        SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_USER)
    else     
        SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL)
}
lidt(&idt_pd);
```
这一部分的实现和lab1\_answer中的略有不同，这里我检查了i是否是系统调用，以及转换为内核态的命令（这一部分是Challenge 1中的内容），分别使用了不同的DPL。

3. 这部分很简单，直接统计tick的个数，如果是100的倍数则输出即可。

## Challenge 1

```c
static void
lab1_switch_to_user(void) {
    //LAB1 CHALLENGE 1 : 2015011347
    asm volatile (
	    "sub $0x8, %%esp \n"
	    "int %0 \n"
	    "movl %%ebp, %%esp"
	    : 
	    : "i"(T_SWITCH_TOU)
	);
}

static void
lab1_switch_to_kernel(void) {
    //LAB1 CHALLENGE 1 : 2015011347
    asm volatile (
	    "int %0 \n"
	    "movl %%ebp, %%esp \n"
	    : 
	    : "i"(T_SWITCH_TOK)
	);
}
```

我们可以通过内联汇编的方法系统调用T_SWITCH_TOU和T_SWITCH_TOK。在trap.c的trap_dispatch中，我们可以实现T_SWITCH_TOU和T_SWITCH_TOK这两个中断，这可以通过新建一个trap_frame，然后修改原来trap_frame里面的cs，ds，es，eflags四个标志来实现。

***

## Appendix A

```makefile
+ cc kern/init/init.c
gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/init/init.c -o obj/kern/init/init.o
+ cc kern/libs/readline.c
gcc -Ikern/libs/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/libs/readline.c -o obj/kern/libs/readline.o
+ cc kern/libs/stdio.c
gcc -Ikern/libs/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/libs/stdio.c -o obj/kern/libs/stdio.o
+ cc kern/debug/kdebug.c
gcc -Ikern/debug/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/debug/kdebug.c -o obj/kern/debug/kdebug.o
+ cc kern/debug/kmonitor.c
gcc -Ikern/debug/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/debug/kmonitor.c -o obj/kern/debug/kmonitor.o
+ cc kern/debug/panic.c
gcc -Ikern/debug/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/debug/panic.c -o obj/kern/debug/panic.o
+ cc kern/driver/clock.c
gcc -Ikern/driver/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/driver/clock.c -o obj/kern/driver/clock.o
+ cc kern/driver/console.c
gcc -Ikern/driver/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/driver/console.c -o obj/kern/driver/console.o
+ cc kern/driver/intr.c
gcc -Ikern/driver/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/driver/intr.c -o obj/kern/driver/intr.o
+ cc kern/driver/picirq.c
gcc -Ikern/driver/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/driver/picirq.c -o obj/kern/driver/picirq.o
+ cc kern/trap/trap.c
gcc -Ikern/trap/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/trap/trap.c -o obj/kern/trap/trap.o
+ cc kern/trap/trapentry.S
gcc -Ikern/trap/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/trap/trapentry.S -o obj/kern/trap/trapentry.o
+ cc kern/trap/vectors.S
gcc -Ikern/trap/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/trap/vectors.S -o obj/kern/trap/vectors.o
+ cc kern/mm/pmm.c
gcc -Ikern/mm/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/mm/pmm.c -o obj/kern/mm/pmm.o
+ cc libs/printfmt.c
gcc -Ilibs/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/  -c libs/printfmt.c -o obj/libs/printfmt.o
+ cc libs/string.c
gcc -Ilibs/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/  -c libs/string.c -o obj/libs/string.o
+ ld bin/kernel
ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o obj/kern/libs/readline.o obj/kern/libs/stdio.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/debug/panic.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/intr.o obj/kern/driver/picirq.o obj/kern/trap/trap.o obj/kern/trap/trapentry.o obj/kern/trap/vectors.o obj/kern/mm/pmm.o  obj/libs/printfmt.o obj/libs/string.o
+ cc boot/bootasm.S
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
+ cc boot/bootmain.c
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o
+ cc tools/sign.c
gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
+ ld bin/bootblock
ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o	
dd if=/dev/zero of=bin/ucore.img count=10000
dd if=bin/bootblock of=bin/ucore.img conv=notrunc
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
```

