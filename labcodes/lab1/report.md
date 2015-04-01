# Lab1 report

## [练习1]

##### [练习1.1] 操作系统镜像文件 ucore.img 是如何一步一步生成的?(需要比较详细地解释 Makefile 中每一条相关命令和命令参数的含义,以及说明命令导致的结果)

查看```Makefile```可以发现如下代码:

	# create ucore.img
	UCOREIMG	:= $(call totarget,ucore.img)

	$(UCOREIMG): $(kernel) $(bootblock)
		$(V)dd if=/dev/zero of=$@ count=10000
		$(V)dd if=$(bootblock) of=$@ conv=notrunc
		$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

	$(call create_target,ucore.img)

执行```make "V=" > makelog.txt```,打开```makelog.txt```可以发现如下内容:

	dd if=/dev/zero of=bin/ucore.img count=10000
	dd if=bin/bootblock of=bin/ucore.img conv=notrunc
	dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc

说明```ucore.img```是一个包含10000个512字节的块,每个块用0填充的文件,然后将生成好的```bootlock```填入这个文件的第一个块,将```kernel```从这个文件的第二块开始填充。

##### [练习1.2] 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?

查看```sign.c```的代码,可以发现如下内容:

    char buf[512];
	// ...
    buf[510] = 0x55;
    buf[511] = 0xAA;
	// ...
    size = fwrite(buf, 1, 512, ofp);
    if (size != 512) {
        fprintf(stderr, "write '%s' error, size is %d.\n", argv[2], size);
        return -1;
    }
	// ...
    printf("build 512 bytes boot sector: '%s' success!\n", argv[2]);

这些内容说明,一个被系统认为符合规范的硬盘主引导扇区的大小应为512字节,且第510个字节是```0x55```,第511个字节是```0xAA```。



## [练习2]

##### [练习2.1] 从 CPU 加电后执行的第一条指令开始,单步跟踪 BIOS 的执行。

完成这个练习时,由于我在```Windows```下进行实验,```msys```的```terminal```是```rxvt```,因此需要将```Makefile```中的```TERMINAL```设置为```rxvt```,才能使编译通过。即使这样,我的```gdb```仍然无法连接至```qemu```,目前还在进一步调试中。


## [练习5] 
##### 实现函数调用堆栈跟踪函数 

根据汇编课上学习的知识,```ebp```指向caller的```ebp```,```ebp+4```指向caller的```eip```,```ebp+8```之后是参数。据此可完成实验。

最后一行:

	ebp:0x00007bf8 eip:0x00007d66 args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
	<unknow>: -- 0x00007d65 –-

说明```bootmain()```的```ebp```为```0x00007bf8```。



## [练习6]
##### 完善中断初始化和处理

##### [练习6.1] 中断向量表中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

查阅```mmu.h```可知,中断向量表中每一个表项的结构如下:

	struct gatedesc {
		unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
		unsigned gd_ss : 16;            // segment selector
		unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
		unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
		unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
		unsigned gd_s : 1;                // must be 0 (system)
		unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
		unsigned gd_p : 1;                // Present
		unsigned gd_off_31_16 : 16;        // high bits of offset in segment
	};

所以中断向量表的每一个表项占用8字节,其中2-3字节是ss(段选择子),0-1字节是位移低位,6-7字节是位移高位,两者联合便是中断处理程序的入口地址。

##### [练习6.2] 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。

具体实现见代码。需要说明的是,```T_SYSCALL```和```T_SWITCH_TOK```两个中断是用户态可以触发的,而且只有用户态触发才有意义,因此我将这两个中断的特权级设为了用户级,其它均设为了内核级。

##### [练习6.3] 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数

具体实现见代码。需要说明的是,我的全局变量```ticks```在到达```TICK_NUM```的整数倍之后就会清零，这样能避免其溢出。

