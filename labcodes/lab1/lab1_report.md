#lab_1 report

和answer中的report使用相同格式

## [练习1]

[练习1.1] 操作系统镜像文件 ucore.img 是如何一步一步生成的?(需要比较详细地解释 Makefile 中
每一条相关命令和命令参数的含义,以及说明命令导致的结果)

由于我之前对makefile不熟悉，所以参考了answer中的说明，也在网上查询了一些make的使用方法。

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
    bootblock = $(call totarget,bootblock)
    $(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
        @echo + ld $@
        $(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
        @$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
        @$(OBJDUMP) -t $(call objfile,bootblock) | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,bootblock)
        @$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
        @$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

    $(call create_target,bootblock)
```


[练习1.2] 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?

## [练习2]

[练习2.1] 从 CPU 加电后执行的第一条指令开始,单步跟踪 BIOS 的执行。

[练习2.2] 在初始化位置0x7c00 设置实地址断点,测试断点正常。

[练习2.3] 在调用qemu 时增加-d in_asm -D q.log 参数，便可以将运行的汇编指令保存在q.log 中。
将执行的汇编代码与bootasm.S 和 bootblock.asm 进行比较，看看二者是否一致。

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