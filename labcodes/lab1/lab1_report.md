#lab_1 report

和answer中的report使用相同格式

## [练习1]

[练习1.1] 操作系统镜像文件 ucore.img 是如何一步一步生成的?(需要比较详细地解释 Makefile 中
每一条相关命令和命令参数的含义,以及说明命令导致的结果)

由于我之前对makefile不熟悉，所以参考了answer中的说明，也在网上查询了一些make的使用方法。

首先有一些定义在function.mk中，所以不能仅仅只看makefile一个文件。

```
UCOREIMG	:= $(call totarget,ucore.img)
$(UCOREIMG): $(kernel) $(bootblock)
    //kernal
    //为了生存kernel,我们需要遍历KSRCDIR这个宏中定义的所有文件，编译出最终目标
    $(kernel): tools/kernel.ld
    $(kernel): $(KOBJS)
        //KOBJS
        KOBJS	= $(call read_packet,kernel libs)//libs文件夹
        //body
        @echo + ld $@//遍历并链接所有文件,之后使用linux的echo命令输出
        $(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)//连接kernal.ld
        @$(OBJDUMP) -S $@ > $(call asmfile,kernel)//使用objdump进行反汇编
        @$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
        //通过下面这条指令添加所有需要编译链接的文件
        $(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
        //之后可以使用@来遍历这些文件，减少makefile的工作量
    //bootblock
    //为了生成bootblock，首先需要生成bootasm.o、bootmain.o、sign
    $(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
        //我们发现bootfiles的作用是生成宏把bootblock需要的源文件编译
        $(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
        //接下来是链接的过程，比较繁琐
        @echo + ld $@
        //这一句把宏翻译之后就是生成bootblock.o
        $(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
        //下面是objdump进行反汇编
        @$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
        @$(OBJDUMP) -t $(call objfile,bootblock) | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,bootblock)
        @$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
        //使用sign工具处理bootblock.out，生成bootblock
        @$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
```


[练习1.2] 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?

从sign.c的代码来看，一个磁盘主引导扇区只有512字节。且
第510个（倒数第二个）字节是0x55，
第511个（倒数第一个）字节是0xAA。

```
char buf[512];
buf[510] = 0x55;
buf[511] = 0xAA;
```

## [练习2]

[练习2.1] 从 CPU 加电后执行的第一条指令开始,单步跟踪 BIOS 的执行。

第一步，在 lab1/tools/gdbinit 中 `target remote :1234` 前加一行 `set architecture i8086`，
然后使用makefile中debug命令
```
debug: $(UCOREIMG)
		$(V)$(TERMINAL) -e "$(QEMU) -S -s -d in_asm -D $(BINDIR)/q.log -parallel stdio -hda $< -serial null"
		$(V)sleep 2
		$(V)$(TERMINAL) -e "gdb -q -tui -x tools/gdbinit"
```
这样就会出现一个神奇的gdb界面(比我之前的gdb要多一个看代码的区域)

[练习2.2] 在初始化位置0x7c00 设置实地址断点,测试断点正常。

在运行gdb之后，输入以下命令：
```
    b *0x7c00  //在0x7c00处设置断点。此地址是bootloader入口点地址，可看boot/bootasm.S的start地址处
	c          //continue简称，表示继续执行
	x /2i $pc  //显示当前eip处的汇编指令
```
我们可以看到：
```
	=> 0x7c00:      cli    
	   0x7c01:      cld    
```
       

[练习2.3] 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。

在调用qemu 时增加-d in_asm -D q.log 参数，便可以将运行的汇编指令保存在q.log 中。
将执行的汇编代码与bootasm.S 和 bootblock.asm 进行比较，看看二者是否一致。

我们在makefile中增加一个和debug几乎一样只是能输出log的指令`debug-mon`
```
    debug-mon: $(UCOREIMG)
    $(V)$(TERMINAL) -e "$(QEMU) -S -s -d in_asm -D $(BINDIR)/q.log -monitor stdio -hda $< -serial null"
    $(V)sleep 2
    $(V)$(TERMINAL) -e "gdb -q -x tools/gdbinit"
```

## [练习3]
分析bootloader 进入保护模式的过程。

## [练习4]
分析bootloader加载ELF格式的OS的过程。

## [练习5] 
实现函数调用堆栈跟踪函数 

## [练习6]
完善中断初始化和处理

[练习6.1] 中断向量表中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

[练习6.2] 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。

[练习6.3] 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数

## [练习7]

增加syscall功能，即增加一用户态函数（可执行一特定系统调用：获得时钟计数值），
当内核初始完毕后，可从内核态返回到用户态的函数，而用户态的函数又通过系统调用得到内核态的服务