---
layout: post
title:  "Linux 2.6.32 进程调度"
date:   2015-11-22 20:33:25
categories: linux 
tags: linux
---

#	1.linux2.6.32使用的trap gate和interrupt gate

Intel X86体系中有中断和异常之分，

*	中断发生在程序执行的任意时刻，一般来自外部硬件产生得事件，例如磁盘，网卡等I/O设备中断，中断一般和执行指令没有直接的关系，可以理解中断是Asynchronous(异步)的。
*	异常一般是执行指令时由处理器捕捉到的错误，分为Fault, Trap和Aborts。另外一种异常是软中断/系统调用，用来和系统内核交互。 异常和执行指令有直接关系，比如执行了错误的指令，调用int系统调用等。从这层意义上理解可以说异常是同步(Synchronous)的。   
	-- Faults — correctable; offending instruction is retried  
	-- Traps — often for debugging; instruction is not retried   
	-- Aborts — major error (hardware failure) 
   

Linux2.6.32中主要使用了x86提供的中断门和陷阱们，按照linux2.6.32的使用方法，IDT中断描述符有3种 

*	interupte_gate dpl:00(仅对内核态开放) type:01110(interrupt gate IF=0) 

_所有的IRQ(0x20-0x2f) 包括timer hd_  
_几乎所有的异常(divide error, page fault)_  

*	system_intr_gate dpl:11(用户态开放) type:01110 (interrupt gate IF=0)   

_0x03 int3_  
_0x04 overflow_   
 
*	system_trap_gate dpl:11(用户态开放) type:01111 (trap gate IF=1)   

_0x80 system_call_

     arch/x86/kernel/irqinit.c
     arch/x86/kernel/traps.c

	    set_intr_gate(0, &divide_error);
		set_intr_gate_ist(1, &debug, DEBUG_STACK);
		set_intr_gate_ist(2, &nmi, NMI_STACK);
		/* int3 can be called from all */
		set_system_intr_gate_ist(3, &int3, DEBUG_STACK);
		/* int4 can be called from all */
		set_system_intr_gate(4, &overflow);
		set_intr_gate(5, &bounds);
		set_intr_gate(6, &invalid_op);
		set_intr_gate(7, &device_not_available);
	#ifdef CONFIG_X86_32
		set_task_gate(8, GDT_ENTRY_DOUBLEFAULT_TSS);
	#else
		set_intr_gate_ist(8, &double_fault, DOUBLEFAULT_STACK);
	#endif
		set_intr_gate(9, &coprocessor_segment_overrun);
		set_intr_gate(10, &invalid_TSS);
		set_intr_gate(11, &segment_not_present);
		set_intr_gate_ist(12, &stack_segment, STACKFAULT_STACK);
		set_intr_gate(13, &general_protection);
		set_intr_gate(14, &page_fault);
		set_intr_gate(15, &spurious_interrupt_bug);
		set_intr_gate(16, &coprocessor_error);
		set_intr_gate(17, &alignment_check);
	#ifdef CONFIG_X86_MCE
		set_intr_gate_ist(18, &machine_check, MCE_STACK);
	#endif
		set_intr_gate(19, &simd_coprocessor_error);

		/* Reserve all the builtin and the syscall vector: */
		for (i = 0; i < FIRST_EXTERNAL_VECTOR; i++)
			set_bit(i, used_vectors);

	#ifdef CONFIG_IA32_EMULATION
		set_system_intr_gate(IA32_SYSCALL_VECTOR, ia32_syscall);
		set_bit(IA32_SYSCALL_VECTOR, used_vectors);
	#endif

IF=0表示中断当前中断线上的屏蔽，会在所有处理器上屏蔽掉当前同一中断线，这样可以防止同一中断线上接收另一个新的中断。通常情况下，所有其他的中断都是打开的，所以这些不同中断线上的其他中断都能被处理，但当前中断线总是被禁止的。由此可见，同一个中断处理程序绝不会被同时调用产生同一个处理中断函数被嵌套的情况。而且，用这种方式也控制了整个中断处理栈的深度，内核最多可以支持15个IRQ同时在线，也就是说最大的内核中断处理栈的深度为15。

时钟片的任务切换（timer_interrupt)就是这样的一个中断，所以每次一定只有一个时间片处理中断发生，将CPU调度成功。

系统调用是通过trap_gate做的，很容易被中断掉，典型的是陷入系统调用以后被磁盘中断掉，注意Linux2.4之前都是内核非抢占的，陷于system_call的内核态系统调用不会被时钟中断调度出去。

#	2.关于中断/异常的嵌套执行

*	首先，0.12中也是运行中断/异常的嵌套执行的，比如系统调用陷入内核，磁盘I/O完成转而去执行hd_interrupt。这样形成一个int 0x80 + hd_interrupt的内核控制路径。
*	2.6.32当然也允许中断内核控制路径嵌套执行。

#	3.linux2.6.32和linux0.12主要的不同点

*	除了软中断/系统调用以外的中断(IRQ0-IRQ15)和异常(0-32)都使用了中断门，也就是说针对这些中断和异常的处理都是仅有一个处理函数被允许，不可以嵌套同一个处理函数；而在0.12中仅仅是IRQ0-15是使用中断门的，异常(0-32)使用的是trap gate。
*	抢占式内核的概念:运行在内核态的软中断/系统调用，会被中断处理程序打断，常见的是I/O设备如磁盘，软盘，网卡，这种打断在linux0.12中本身就是可以支持的，这是中断处理程序本身应有之意，并不是抢占式内核的含义。这里抢占式内核的含义是，当时钟中断来临时，检查当前运行在内核态的软中断/系统调用，如果配置可以被抢占，并且此时系统中确实有更高优先级的任务需要执行，则调用schedule()将当前进程保持在内核态的软中断/系统调用，而转而去执行另外的任务。等到下一次调度时候，本次被打断的任务继续在内核态执行。而在0.12中，时钟中断来临只会对任务进行计时，如果任务运行在内核态并不会做重新调度。
*	引入了中断上下文，异常(包括软中断/系统调用)是在进程的上下文中执行，而硬件引起的中断是在中断上下文中执行，和进程无关。中断上下文也被称作原子上下文，该上下文中得执行代码不可阻塞，不可以进程切换。（问题： 时钟中断是如何进行进程切换的？）0.12中无论是异常还是中断都是在被打断的进程上下文中执行,
*	引入上半部和下半部，将耗时比较长的任务放到下半部去执行。0.12没有下半部的概念，中断都是在一个处理函数中执行的。
*	2.6.32抢占的情形  
用户抢占：  
1) 从系统调用(read/write/fork/exec等)返回用户空间时(注意是在中断处理程序执行完后进行schedule的)   
2) 从中断处理程序(hd/network/timer等)返回用户空间时(注意是在中断处理程序执行完后进行schedule的) 

内核抢占：   
1）中断处理程序(hd/network/timer等)正在执行，且返回内核空间之前；    
2）释放锁的内核代码立即检查可否进行内核抢占；    
3）显示调用schedule；    
4）任务阻塞(调用schedule)   

*	0.12抢占的情形   
用户抢占：  
1）从系统调用(read/write/fork/exec等)返回用户空间时（注意是在中断处理程序执行完后进行schedule的)   
2）从时钟中断程序(timer)返回用户空间时(注意是在中断处理程序do_timer中进行schedule的) 

内核抢占：  
1）显示调用schedule；    
2）任务阻塞(调用schedule) 
*	和0.12相比，用户抢占的情况基本相同，新的内核增加了除timer以外的其他中断处理程序返回用户空间时抢占的可能；在内核抢占方面增加了从中断处理程序返回内核空间时的抢占，以及内核代码中再一次具有抢占性时刻的抢占，从而使得内核态任务的抢占成为可能。

#	4.几个问题
问题1： 时钟中断是如何进行进程切换的？不是说中断处理是不可以阻塞，不可以进程切换的么？
回答： 时钟中断处理程序tick_periodic作为硬件中断的一种，并不进行进程切换，只进行计时操作。当中断程序执行完以后再根据情况进行schdule。这和0.12中直接在do_timer中做schedule是不同的。


问题2：软中断/系统调用+异常(do_no_page)最多两层嵌套？


