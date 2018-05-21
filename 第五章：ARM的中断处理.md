#ARM的中断处理
##5.1. 中断初始化与注册 
###5.1.1. 中断与异常
 中断可分为同步（synchronous）中断和异步（asynchronous）中断：
 （1）同步中断又称为异常，是当指令执行时由CPU控制单元产生，之所以称为同步，是因为只有在一条指令执行完成后CPU才会发出中断。典型的异常有缺页和除0。
（2）异步中断是指由其他硬件设备依照CPU时钟信号随机产生。

####5.1.1.1 ARM异常与向量表
 当一个异常或中断发生时，处理器会把pc设置为一个特定的存储器地址。这个地址放在一个被称为向量表（vector table）的特定的地址范围内。向量表的入口是一些跳转指令，跳转到专门处理某个异常或中断的子程序。
 存储器映射地址0x00000000是为向量表保留的。在有些处理器中，向量表可以选择定位在存储空间的更高地址（从偏移量0xffff0000开始）。操作系统，如Linux和Microsoft的嵌入式操作系统，就可以利用这一特性。
 
 当一个异常或中断发生时，处理器挂起正常的执行转而从向量表（见表1-1）装载指令。每一个向量表入口包含一条指向一个特定子程序的跳转指令。 
 ![](http://i.imgur.com/I7TxUuc.jpg)
 
- 复位向量是处理器上电后执行的第一条指令的位置。这条指令使处理器转到初始化代码处。
- 未定义指令向量是在处理器不能对一条指令译码时使用的。
- 软件中断向量是执行SWI指令时被调用的。SWI指令经常被用作调用一个操作系统例程的机制。
- 预取指中止向量是发生在处理器试图从一个未获得正确访问权限的地址去取指时，实际上中止发生在“译码”阶段。
- 数据中止向量是与预取指中止类似，发生在一条指令试图访问未获得正确访问权限的数据存储器时。
- 中断请求向量是用于外部硬件中断处理器的正常执行流。只有当cpsr中的IRQ位未被屏蔽时才能发生。
- 快速中断请求向量类似于中断请求，是为要求更短的中断响应时间的硬件保留的。只有当cpsr中的FIQ位未被屏蔽时才能发生。
  
 >(注意：本章代码为linux-v4.4)
###5.1.2. 相关数据结构

 与中断相关的主要数据结构有三个，分别是：irq\_desc、irq\_chip和irqaction。下面将介绍一下这三个数据结构的实现以及它们之间的关系。

内核通过irq\_desc来描述一个中断，irq\_desc结构体在include/linux/irqdesc.h中定义：

	 46 struct irq_desc {
	 47         struct irq_common_data  irq_common_data;
	 48         struct irq_data         irq_data;
	 49         unsigned int __percpu   *kstat_irqs;
	 50         irq_flow_handler_t      handle_irq;
	 51 #ifdef CONFIG_IRQ_PREFLOW_FASTEOI
	 52         irq_preflow_handler_t   preflow_handler;
	 53 #endif
	 54         struct irqaction        *action;        /* IRQ action list */
	 55         unsigned int            status_use_accessors;
	 56         unsigned int            core_internal_state__do_not_mess_with_it;
	 57         unsigned int            depth;          /* nested irq disables */
	 58         unsigned int            wake_depth;     /* nested wake enables */
	 59         unsigned int            irq_count;      /* For detecting broken IRQs */
	 60         unsigned long           last_unhandled; /* Aging timer for unhandled count */
	 61         unsigned int            irqs_unhandled;
	 62         atomic_t                threads_handled;
	 63         int                     threads_handled_last;
	 64         raw_spinlock_t          lock;
	 65         struct cpumask          *percpu_enabled;
	 66 #ifdef CONFIG_SMP
	 67         const struct cpumask    *affinity_hint;
	 68         struct irq_affinity_notify *affinity_notify;
	 69 #ifdef CONFIG_GENERIC_PENDING_IRQ
	 70         cpumask_var_t           pending_mask;
	 71 #endif
	 72 #endif
	 73         unsigned long           threads_oneshot;
	 74         atomic_t                threads_active;
	 75         wait_queue_head_t       wait_for_threads;
	 76 #ifdef CONFIG_PM_SLEEP
	 77         unsigned int            nr_actions;
	 78         unsigned int            no_suspend_depth;
	 79         unsigned int            cond_suspend_depth;
	 80         unsigned int            force_resume_depth;
	 81 #endif
	 82 #ifdef CONFIG_PROC_FS
	 83         struct proc_dir_entry   *dir;
	 84 #endif
	 85         int                     parent_irq;
	 86         struct module           *owner;
	 87         const char              *name;
	 88 } ____cacheline_internodealigned_in_smp;

 其中，handle\_irq指向中断处理函数，每当发生中断时，体系结构相关的代码都会调用handle\_irq，该函数负责使用chip中提供的处理器相关的方法，来完成处理中断所必需的一些底层操作。用于不同中断类型的默认函数由内核提供，例如handle\_level\_irq和handle\_edge\_irq等。

中断控制器相关操作被封装到irq\_chip，该结构体在include/linux/irq.h中定义：

	346 struct irq_chip {
	347         const char      *name;
	348         unsigned int    (*irq_startup)(struct irq_data *data);
	349         void            (*irq_shutdown)(struct irq_data *data);
	350         void            (*irq_enable)(struct irq_data *data);
	351         void            (*irq_disable)(struct irq_data *data);
	352 
	353         void            (*irq_ack)(struct irq_data *data);
	354         void            (*irq_mask)(struct irq_data *data);
	355         void            (*irq_mask_ack)(struct irq_data *data);
	356         void            (*irq_unmask)(struct irq_data *data);
	357         void            (*irq_eoi)(struct irq_data *data);
	358 
	359         int             (*irq_set_affinity)(struct irq_data *data, const struct cpumask *dest, bool force);
	360         int             (*irq_retrigger)(struct irq_data *data);
	361         int             (*irq_set_type)(struct irq_data *data, unsigned int flow_type);
	362         int             (*irq_set_wake)(struct irq_data *data, unsigned int on);
	363 
	364         void            (*irq_bus_lock)(struct irq_data *data);
	365         void            (*irq_bus_sync_unlock)(struct irq_data *data);
	366 
	367         void            (*irq_cpu_online)(struct irq_data *data);
	368         void            (*irq_cpu_offline)(struct irq_data *data);
	369 
	370         void            (*irq_suspend)(struct irq_data *data);
	371         void            (*irq_resume)(struct irq_data *data);
	372         void            (*irq_pm_shutdown)(struct irq_data *data);
	373 
	374         void            (*irq_calc_mask)(struct irq_data *data);
	375 
	376         void            (*irq_print_chip)(struct irq_data *data, struct seq_file *p);
	377         int             (*irq_request_resources)(struct irq_data *data);
	378         void            (*irq_release_resources)(struct irq_data *data);
	379 
	380         void            (*irq_compose_msi_msg)(struct irq_data *data, struct msi_msg *msg);
	381         void            (*irq_write_msi_msg)(struct irq_data *data, struct msi_msg *msg);
	382 
	383         int             (*irq_get_irqchip_state)(struct irq_data *data, enum irqchip_irq_state which, bool *state);
	384         int             (*irq_set_irqchip_state)(struct irq_data *data, enum irqchip_irq_state which, bool state);
	385 
	386         int             (*irq_set_vcpu_affinity)(struct irq_data *data, void *vcpu_info);
	387 
	388         unsigned long   flags;
	389 };


action指向一个操作链表，在中断发生时执行。由中断通知设备驱动程序，可以将与之相关的处理函数放置在此处。irqaction结构体在include/linux/interrupt.h中定义：

	110 struct irqaction {
	111         irq_handler_t           handler;
	112         void                    *dev_id;
	113         void __percpu           *percpu_dev_id;
	114         struct irqaction        *next;
	115         irq_handler_t           thread_fn;
	116         struct task_struct      *thread;
	117         struct irqaction        *secondary;
	118         unsigned int            irq;
	119         unsigned int            flags;
	120         unsigned long           thread_flags;
	121         unsigned long           thread_mask;
	122         const char              *name;
	123         struct proc_dir_entry   *dir;
	124 } ____cacheline_internodealigned_in_smp;


 每个处理函数都对应该结构体的一个实例irqaction，该结构体中最重要的成员是handler，这是个函数指针。该函数指针就是通过request_irq来初始化的，在后续小节中将详细介绍。当系统产生中断，内核将调用该函数指针所指向的处理函数。name和dev\_id唯一地标识一个irqaction实例，name用于标识设备名，dev\_id用于指向设备的数据结构实例。

请注意irq\_desc中的handle\_irq不同于irqaction中的handler，这两个函数指针分别定义为：

	typedef void (*irq_flow_handler_t)(struct irq_desc *desc);
	typedef irqreturn_t (*irq_handler_t)(int, void *);
	

内核通过维护一个irq\_desc的全局数组来管理所有的中断。该数组在kernel/irq/irqdesc.c中定义：

	259 struct irq_desc irq_desc[NR_IRQS] __cacheline_aligned_in_smp = {
	260         [0 ... NR_IRQS-1] = {
	261                 .handle_irq     = handle_bad_irq,
	262                 .depth          = 1,
	263                 .lock           = __RAW_SPIN_LOCK_UNLOCKED(irq_desc->lock),
	264         }
	265 };

其中NR_IRQS表示支持中断的最大数，在arch/alpha/include/asm/irq.h文件中定义。

**IRQ相关数据结构关系如图：**

![](http://i.imgur.com/LFdOxTv.jpg)

###5.1.3. 中断初始化
 ARM Linux中断初始化主要通过三个函数完成：early\_trap\_init 、early\_irq\_init和init\_IRQ。


初始化以处理异常——early\_trap\_init函数定义在arch/arm/kernel/traps.c文件中，在setup_arch函数最后调用的，主要完成ARM异常向量表和异常处理程序的重定位。

	791 void __init early_trap_init(void *vectors_base)
	792 {
	793 #ifndef CONFIG_CPU_V7M
	794         unsigned long vectors = (unsigned long)vectors_base;
	795         extern char __stubs_start[], __stubs_end[];
	796         extern char __vectors_start[], __vectors_end[];
	797         unsigned i;
	798 
	799         vectors_page = vectors_base;
	800 
	801         /*
	802          * Poison the vectors page with an undefined instruction.  This
	803          * instruction is chosen to be undefined for both ARM and Thumb
	804          * ISAs.  The Thumb version is an undefined instruction with a
	805          * branch back to the undefined instruction.
	806          */
	807         for (i = 0; i < PAGE_SIZE / sizeof(u32); i++)
	808                 ((u32 *)vectors_base)[i] = 0xe7fddef1;
	809 
	810         /*
	811          * Copy the vectors, stubs and kuser helpers (in entry-armv.S)
	812          * into the vector page, mapped at 0xffff0000, and ensure these
	813          * are visible to the instruction stream.
	814          */
	815         memcpy((void *)vectors, __vectors_start, __vectors_end - __vectors_start);
	816         memcpy((void *)vectors + 0x1000, __stubs_start, __stubs_end - __stubs_start);
	817 
	818         kuser_init(vectors_base);
	819 
	820         flush_icache_range(vectors, vectors + PAGE_SIZE * 2);
	821 #else /* ifndef CONFIG_CPU_V7M */
	822         /*
	823          * on V7-M there is no need to copy the vector table to a dedicated
	824          * memory area. The address is configurable and so a table in the kernel
	825          * image can be used.
	826          */
	827 #endif
	828 } 

 为了调用异常代码，在early\_trap\_init()函数中将处理器及辅助器代码复制到异常向量表基址，并设置CPU域。

early\_irq\_init函数定义在kernel/irq/irqdesc.c文件中：

	259 int __init early_irq_init(void)
	260 {
	261         int i, initcnt, node = first_online_node;
	262         struct irq_desc *desc;
	263 
	264         init_irq_default_affinity();
	265 
	266         /* Let arch update nr_irqs and return the nr of preallocated irqs */
	267         initcnt = arch_probe_nr_irqs();
	268         printk(KERN_INFO "NR_IRQS:%d nr_irqs:%d %d\n", NR_IRQS, nr_irqs, initcnt);
	269 
	270         if (WARN_ON(nr_irqs > IRQ_BITMAP_BITS))
	271                 nr_irqs = IRQ_BITMAP_BITS;
	272 
	273         if (WARN_ON(initcnt > IRQ_BITMAP_BITS))
	274                 initcnt = IRQ_BITMAP_BITS;
	275 
	276         if (initcnt > nr_irqs)
	277                 nr_irqs = initcnt;
	278 
	279         for (i = 0; i < initcnt; i++) {
	280                 desc = alloc_desc(i, node, NULL);
	281                 set_bit(i, allocated_irqs);
	282                 irq_insert_desc(i, desc);
	283         }
	284         return arch_early_irq_init();
	285 }


init\_IRQ函数定义在arch/arm/kernel/irq.c文件中：

	 83 void __init init_IRQ(void)
	 84 {
	 85         int ret;
	 86 
	 87         if (IS_ENABLED(CONFIG_OF) && !machine_desc->init_irq)
	 88                 irqchip_init();
	 89         else
	 90                 machine_desc->init_irq();
	 91 
	 92         if (IS_ENABLED(CONFIG_OF) && IS_ENABLED(CONFIG_CACHE_L2X0) &&
	 93             (machine_desc->l2c_aux_mask || machine_desc->l2c_aux_val)) {
	 94                 if (!outer_cache.write_sec)
	 95                         outer_cache.write_sec = machine_desc->l2c_write_sec;
	 96                 ret = l2x0_of_init(machine_desc->l2c_aux_val,
	 97                                    machine_desc->l2c_aux_mask);
	 98                 if (ret)
	 99                         pr_err("L2C: failed to init: %d\n", ret);
	100         }
	101 
	102         uniphier_cache_init();
	103 }

###5.1.4 注册中断处理程序
本节将主要讲述如何注册中断处理程序。Linux中用于注册中断处理程序的函数是request\_irq，该函数定义在include/linux/interrupt.h：

	133 static inline int __must_check
	134 request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
	135             const char *name, void *dev)
	136 {
	137         return request_threaded_irq(irq, handler, NULL, flags, name, dev);
	138 }
	
		request_thread_irq()函数定义在kernel/irq/manage.c
	
	1604 int request_threaded_irq(unsigned int irq, irq_handler_t handler,
	1605                          irq_handler_t thread_fn, unsigned long irqflags,
	1606                          const char *devname, void *dev_id)
	1607 {
	1608         struct irqaction *action;
	1609         struct irq_desc *desc;
	1610         int retval;
	1611 
	1612         /*
	1613          * Sanity-check: shared interrupts must pass in a real dev-ID,
	1614          * otherwise we'll have trouble later trying to figure out
	1615          * which interrupt is which (messes up the interrupt freeing
	1616          * logic etc).
	1617          *
	1618          * Also IRQF_COND_SUSPEND only makes sense for shared interrupts and
	1619          * it cannot be set along with IRQF_NO_SUSPEND.
	1620          */
	1621         if (((irqflags & IRQF_SHARED) && !dev_id) ||
	1622             (!(irqflags & IRQF_SHARED) && (irqflags & IRQF_COND_SUSPEND)) ||
	1623             ((irqflags & IRQF_NO_SUSPEND) && (irqflags & IRQF_COND_SUSPEND)))
	1624                 return -EINVAL;
	1625 
	1626         desc = irq_to_desc(irq);
	1627         if (!desc)
	1628                 return -EINVAL;
	1629 
	1630         if (!irq_settings_can_request(desc) ||
	1631             WARN_ON(irq_settings_is_per_cpu_devid(desc)))
	1632                 return -EINVAL;
	1633 
	1634         if (!handler) {
	1635                 if (!thread_fn)
	1636                         return -EINVAL;
	1637                 handler = irq_default_primary_handler;
	1638         }
	1639 
	1640         action = kzalloc(sizeof(struct irqaction), GFP_KERNEL);
	1641         if (!action)
	1642                 return -ENOMEM;
	1643 
	1644         action->handler = handler;
	1645         action->thread_fn = thread_fn;
	1646         action->flags = irqflags;
	1647         action->name = devname;
	1648         action->dev_id = dev_id;
	1649 
	1650         chip_bus_lock(desc);
	1651         retval = __setup_irq(irq, desc, action);
	1652         chip_bus_sync_unlock(desc);
	1653 
	1654         if (retval) {
	1655                 kfree(action->secondary);
	1656                 kfree(action);
	1657         }
	1658 
	1659 #ifdef CONFIG_DEBUG_SHIRQ_FIXME
	1660         if (!retval && (irqflags & IRQF_SHARED)) {
	1661                 /*
	1662                  * It's a shared IRQ -- the driver ought to be prepared for it
	1663                  * to happen immediately, so let's make sure....
	1664                  * We disable the irq to make sure that a 'real' IRQ doesn't
	1665                  * run in parallel with our fake.
	1666                  */
	1667                 unsigned long flags;
	1668 
	1669                 disable_irq(irq);
	1670                 local_irq_save(flags);
	1671 
	1672                 handler(irq, dev_id);
	1673 
	1674                 local_irq_restore(flags);
	1675                 enable_irq(irq);
	1676         }
	1677 #endif
	1678         return retval;
	1679 }
	

##5.2 中断处理流程
当一个异常或中断发生时，处理器会将PC设置为特定地址，从而跳转到已经初始化好的异常向量表。因此，要理清中断处理流程，先从异常向量表开始。对于ARM Linux而言，异常向量表和异常处理程序都存在arch/arm/kernel/entry\_armv.S汇编文件中。


异常向量表的代码实现：

	1209 __vectors_start:
	1210         W(b)    vector_rst
	1211         W(b)    vector_und
	1212         W(ldr)  pc, __vectors_start + 0x1000
	1213         W(b)    vector_pabt
	1214         W(b)    vector_dabt
	1215         W(b)    vector_addrexcptn
	1216         W(b)    vector_irq
	1217         W(b)    vector_fiq

vector\_stub是一个宏（在arch/arm/kernel/entry\_armv.S中定义），为了分析更直观，我们将vector\_stub宏展开如下：

	1077 /*
	1078  * Interrupt dispatcher
	1079  */
	1080         vector_stub     irq, IRQ_MODE, 4
	1081 
	1082         .long   __irq_usr                       @  0  (USR_26 / USR_32)
	1083         .long   __irq_invalid                   @  1  (FIQ_26 / FIQ_32)
	1084         .long   __irq_invalid                   @  2  (IRQ_26 / IRQ_32)
	1085         .long   __irq_svc                       @  3  (SVC_26 / SVC_32)
	1086         .long   __irq_invalid                   @  4
	1087         .long   __irq_invalid                   @  5
	1088         .long   __irq_invalid                   @  6
	1089         .long   __irq_invalid                   @  7
	1090         .long   __irq_invalid                   @  8
	1091         .long   __irq_invalid                   @  9
	1092         .long   __irq_invalid                   @  a
	1093         .long   __irq_invalid                   @  b
	1094         .long   __irq_invalid                   @  c
	1095         .long   __irq_invalid                   @  d
	1096         .long   __irq_invalid                   @  e
	1097         .long   __irq_invalid                   @  f

如果发生中断前处于用户态则进入\_\_irq\_usr，其定义如下（arch/arm/kernel/entry\_armv.S)： 

	454         .align  5
	455 __irq_usr:
	456         usr_entry //保存中断上下文
	457         kuser_cmpxchg_check
	458         irq_handler
	459         get_thread_info tsk //获取当前进程的进程描述符中的成员变量thread_info的地址，并将该地址保存到寄存器tsk
	460         mov     why, #0
	461         b       ret_to_user_from_irq //中断处理完成，恢复中断上下文并返回中断产生的位置
	462  UNWIND(.fnend          )
	463 ENDPROC(__irq_usr)
	464 
	465         .ltorg
上面代码中的usr\_entry是一个宏定义，主要用于保护上下文到栈中：

	376         .macro  usr_entry, trace=1, uaccess=1
	377  UNWIND(.fnstart        )
	378  UNWIND(.cantunwind     )       @ do not unwind the user space
	379         sub     sp, sp, #S_FRAME_SIZE  //堆栈被定义为递减式满堆栈，所以首先让sp向下移动#S_FRAME_SIZE,
	准备向栈中存放数据。
	380  ARM(   stmib   sp, {r1 - r12}  )
	381  THUMB( stmia   sp, {r0 - r12}  )
	382 
	383  ATRAP( mrc     p15, 0, r7, c1, c0, 0)
	384  ATRAP( ldr     r8, .LCcralign)
	385 
	386         ldmia   r0, {r3 - r5}
	387         add     r0, sp, #S_PC           @ here for interlock avoidance
	388         mov     r6, #-1                 @  ""  ""     ""        ""
	389 
	390         str     r3, [sp]                @ save the "real" r0 copied
	391                                         @ from the exception stack
	392 
	393  ATRAP( ldr     r8, [r8, #0])
	394 
	395         @
	396         @ We are now ready to fill in the remaining blanks on the stack:
	397         @
	398         @  r4 - lr_<exception>, already fixed up for correct return/restart
	399         @  r5 - spsr_<exception>
	400         @  r6 - orig_r0 (see pt_regs definition in ptrace.h)
	401         @
	402         @ Also, separately save sp_usr and lr_usr
	403         @
	404         stmia   r0, {r4 - r6}
	405  ARM(   stmdb   r0, {sp, lr}^                   )
	406  THUMB( store_user_sp_lr r0, r1, S_SP - S_PC    )
	407 
	408         .if \uaccess
	409         uaccess_disable ip
	410         .endif
	411 
	412         @ Enable the alignment trap while in kernel mode
	413  ATRAP( teq     r8, r7)
	414  ATRAP( mcrne   p15, 0, r8, c1, c0, 0)
	415 
	416         @
	417         @ Clear FP to mark the first stack frame
	418         @
	419         zero_fp
	420 
	421         .if     \trace
	422 #ifdef CONFIG_TRACE_IRQFLAGS
	423         bl      trace_hardirqs_off
	424 #endif
	425         ct_user_exit save = 0
	426         .endif
	427         .endm

上面的这段代码主要是填充结构体pt_regs ，在arch/arm/include/asm/ptrace.h中定义：

	 16 struct pt_regs {
	 17         unsigned long uregs[18];
	 18 };

保存中断上下文后则进入中断处理程序—irq\_handler，定义在arch/arm/kernel/entry\_armv.S文件中:

	 38 /*
	 39  * Interrupt handling.
	 40  */
	 41         .macro  irq_handler
	 42 #ifdef CONFIG_MULTI_IRQ_HANDLER
	 43         ldr     r1, =handle_arch_irq
	 44         mov     r0, sp
	 45         badr    lr, 9997f
	 46         ldr     pc, [r1]
	 47 #else
	 48         arch_irq_handler_default
	 49 #endif
	 50 9997:
	 51         .endm

回看\_\_irq\_usr在完成了irq\_handler中断处理函数后，要完成从中断异常处理程序返回到中断点的工作。如果中断产生于用户空间，则调用ret\_to\_user\_from\_irq来恢复中断现场并返回用户空间继续运行：

	 Linux/arch/arm/kernel/entry-common.S
	 95 ENTRY(ret_to_user_from_irq)
	 96         ldr     r1, [tsk, #TI_FLAGS] //获取thread_info->flags
	 97         tst     r1, #_TIF_WORK_MASK //判断是否有待处理的work
	 98         bne     slow_work_pending //如果有，则进入work_pending进一步处理，主要是完成用户进程抢占相关处理。
	 99 no_work_pending: //如果没有work待处理，则准备恢复中断现场，返回用户空间。
	100         asm_trace_hardirqs_on save = 0
	101 
	102         /* perform architecture specific actions before user return */
	103         arch_ret_to_user r1, lr //调用体系结构相关的代码
	104         ct_user_enter save = 0
	105 
	106         restore_user_regs fast = 0, offset = 0 //用restore_user_regs
	107 ENDPROC(ret_to_user_from_irq)

如果中断产生于内核空间，则调用svc\_exit来恢复中断现场：

	arch/arm/kernel/entry-header.S
	200         .macro  svc_exit, rpsr, irq = 0
	201         .if     \irq != 0
	202         @ IRQs already off
	203 #ifdef CONFIG_TRACE_IRQFLAGS
	204         @ The parent context IRQs must have been enabled to get here in
	205         @ the first place, so there is no point checking the PSR I bit.
	206         bl      trace_hardirqs_on
	207 #endif
	208         .else
	209         @ IRQs off again before pulling preserved data off the stack
	210         disable_irq_notrace
	211 #ifdef CONFIG_TRACE_IRQFLAGS
	212         tst     \rpsr, #PSR_I_BIT
	213         bleq    trace_hardirqs_on
	214         tst     \rpsr, #PSR_I_BIT
	215         blne    trace_hardirqs_off
	216 #endif
	217         .endif
	218         uaccess_restore
	219 
	220 #ifndef CONFIG_THUMB2_KERNEL
	221         @ ARM mode SVC restore
	222         msr     spsr_cxsf, \rpsr
	223 #if defined(CONFIG_CPU_V6) || defined(CONFIG_CPU_32v6K)
	224         @ We must avoid clrex due to Cortex-A15 erratum #830321
	225         sub     r0, sp, #4                      @ uninhabited address
	226         strex   r1, r2, [r0]                    @ clear the exclusive monitor
	227 #endif
	228         ldmia   sp, {r0 - pc}^                  @ load r0 - pc, cpsr
	229 #else
	230         @ Thumb mode SVC restore
	231         ldr     lr, [sp, #S_SP]                 @ top of the stack
	232         ldrd    r0, r1, [sp, #S_LR]             @ calling lr and pc
	233 
	234         @ We must avoid clrex due to Cortex-A15 erratum #830321
	235         strex   r2, r1, [sp, #S_LR]             @ clear the exclusive monitor
	236 
	237         stmdb   lr!, {r0, r1, \rpsr}            @ calling lr and rfe context
	238         ldmia   sp, {r0 - r12}
	239         mov     sp, lr
	240         ldr     lr, [sp], #4
	241         rfeia   sp!
	242 #endif
	243         .endm
	

在arch/arm/kernel/irq.c文件中找到asm\_do\_IRQ函数定义：

	 74 /*
	 75  * asm_do_IRQ is the interface to be used from assembly code.
	 76  */
	 77 asmlinkage void __exception_irq_entry
	 78 asm_do_IRQ(unsigned int irq, struct pt_regs *regs)
	 79 {
	 80         handle_IRQ(irq, regs);
	 81 }
	
	
		arch/arm/kernel/irq.c
	 63 /*
	 64  * handle_IRQ handles all hardware IRQ's.  Decoded IRQs should
	 65  * not come via this function.  Instead, they should provide their
	 66  * own 'handler'.  Used by platform code implementing C-based 1st
	 67  * level decoding.
	 68  */
	 69 void handle_IRQ(unsigned int irq, struct pt_regs *regs)
	 70 {
	 71         __handle_domain_irq(NULL, irq, false, regs);
	 72 }
	
		kernel/irq/irqdesc.c
	356 /**
	357  * __handle_domain_irq - Invoke the handler for a HW irq belonging to a domain
	358  * @domain:     The domain where to perform the lookup
	359  * @hwirq:      The HW irq number to convert to a logical one
	360  * @lookup:     Whether to perform the domain lookup or not
	361  * @regs:       Register file coming from the low-level handling code
	362  *
	363  * Returns:     0 on success, or -EINVAL if conversion has failed
	364  */
	365 int __handle_domain_irq(struct irq_domain *domain, unsigned int hwirq,
	366                         bool lookup, struct pt_regs *regs)
	367 {
	368         struct pt_regs *old_regs = set_irq_regs(regs); // 保存新的寄存器集合指针到全局cpu变量，
	                                                           //方便后续处理程序访问寄存器集合。
	369         unsigned int irq = hwirq;
	370         int ret = 0;
	371 
	372         irq_enter();
	373 
	374 #ifdef CONFIG_IRQ_DOMAIN
	375         if (lookup)
	376                 irq = irq_find_mapping(domain, hwirq);
	377 #endif
	378 
	379         /*
	380          * Some hardware gives randomly wrong interrupts.  Rather
	381          * than crashing, do something sensible.
	382          */
	383         if (unlikely(!irq || irq >= nr_irqs)) {                // 判断中断号
	384                 ack_bad_irq(irq);
	385                 ret = -EINVAL;
	386         } else {
	387                 generic_handle_irq(irq); // 调用中断处理函数
	388         }
	389 
	390         irq_exit();
	391         set_irq_regs(old_regs);
	392         return ret;
	393 }

generic\_handle\_irq是体系结构无关函数，用来调用desc->handle\_irq，该函数指针在中断初始化时指向了处理函数。
其中，set\_irq\_regs将指向寄存器结构体的指针保存在一个全局的CPU变量中，后续的程序可以通过该变量访问寄存器结构体。所以在进入中断处理前，先将全局CPU变量中保存的旧指针保留下来，等到中断处理结束后再将其恢复。irq\_enter负责更新一些统计量：

	     kernel/softirq.c
	322 /*
	323  * Enter an interrupt context.
	324  */
	325 void irq_enter(void)
	326 {
	327         rcu_irq_enter();
	328         if (is_idle_task(current) && !in_interrupt()) {
	329                 /*
	330                  * Prevent raise_softirq from needlessly waking up ksoftirqd
	331                  * here, as softirq will be serviced on return from interrupt.
	332                  */
	333                 local_bh_disable();
	334                 tick_irq_enter();
	335                 _local_bh_enable();
	336         }
	337 
	338         __irq_enter();
	339 }
	

宏\_\_irq\_enter()定义如下：

	 29 /*
	 30  * It is safe to do non-atomic ops on ->hardirq_context,
	 31  * because NMI handlers may not preempt and the ops are
	 32  * always balanced, so the interrupted value of ->hardirq_context
	 33  * will always be restored.
	 34  */
	 35 #define __irq_enter()                                   \
	 36         do {                                            \
	 37                 account_irq_enter_time(current);        \
	 38                 preempt_count_add(HARDIRQ_OFFSET);      \
	 39                 trace_hardirq_enter();                  \
	 40         } while (0)

preempt\_count\_add(HARDIRQ_OFFSET)使表示中断处理程序嵌套层次的计数器加1.

完成中断处理函数调用后，返回到asm\_do\_IRQ继续执行，其中最重要的是执行中断退出函数irq\_exit：

	377 /*
	378  * Exit an interrupt context. Process softirqs if needed and possible:
	379  */
	380 void irq_exit(void)
	381 {
	382 #ifndef __ARCH_IRQ_EXIT_IRQS_DISABLED
	383         local_irq_disable();
	384 #else
	385         WARN_ON_ONCE(!irqs_disabled());
	386 #endif
	387 
	388         account_irq_exit_time(current);
	389         preempt_count_sub(HARDIRQ_OFFSET);
	390         if (!in_interrupt() && local_softirq_pending())
	391                 invoke_softirq();
	392 
	393         tick_irq_exit();
	394         rcu_irq_exit();
	395         trace_hardirq_exit(); /* must be last! */
	396 }

irq\_exit函数首先将preempt\_count\_sub计数器减去HARDIRQ\_OFFSET，用来标识退出硬中断，这与irq\_enter函数中的preempt\_count\_add相对应。

之后irq\_exit会通过宏in\_interrupt()判断当前是否处在中断（interrupt）中。

处理完软中断后,返回asm\_do\_IRQ，该函数最后通过set\_irq\_regs(old\_regs)将寄存器集合指针恢复到发生中断之前的值。asm\_do\_IRQ结束后会返回入口点，再次回到汇编语言代码。

##5.3 软中断softirq 
内核出于性能等方面的考虑，将中断分为两部分：上半部处理中断中需要及时响应且处理时间较短的部分，下半部用于处理对响应时间要求不高且处理时间可能较长的部分。本节将主要介绍中断下半部的三种实现方式之一：软中断。

软中断是在编译期间静态分配的，由数据结构softirq\_action表示,定义在include/linux/interrupt.h：

	436 struct softirq_action
	437 {
	438         void    (*action)(struct softirq_action *);
	439 };

softirq\_action结构体非常简单，仅包含一个指向软中断处理函数的指针。为了管理软中断，内核维护着一个softirq\_action结构体数组softirq\_vec[NR\_SOFTIRQS]，称之为软中断向量表：

	 56 static struct softirq_action softirq_vec[NR_SOFTIRQS] __cacheline_aligned_in_smp;

其中NR\_SOFTIRQS是一个枚举类型内核支持的最大软中断数。

	408 enum
	409 {
	410         HI_SOFTIRQ=0,
	411         TIMER_SOFTIRQ,
	412         NET_TX_SOFTIRQ,
	413         NET_RX_SOFTIRQ,
	414         BLOCK_SOFTIRQ,
	415         BLOCK_IOPOLL_SOFTIRQ,
	416         TASKLET_SOFTIRQ,
	417         SCHED_SOFTIRQ,
	418         HRTIMER_SOFTIRQ, /* Unused, but kept as tools rely on the
	419                             numbering. Sigh! */
	420         RCU_SOFTIRQ,    /* Preferable RCU should always be the last softirq */
	421 
	422         NR_SOFTIRQS
	423 };

通过上面的枚举类型可知当前内核版本支持10种软中断，其中HI\_SOFTIRQ和TASKLET\_SOFTIRQ用于实现tasklet，TIMER\_SOFTIRQ和HRTIMER\_SOFTIRQ用于实现定时器，NET\_TX\_SOFTIRQ和NET\_RX\_SOFTIRQ用于实现网络设备的发送和接受操作，BLOCK\_SOFTIRQ和BLOCK\_IOPOLL\_SOFTIRQ用于实现块设备操作，SCHED\_SOFTIRQ用于调度器。

内核中通过调用open\_softirq函数注册软中断处理程序，该函数接收两个参数：软中断索引号（枚举类型）和处理程序。根据索引号寻址到软中断向量表softirq\_vec的指定元素，使其action成员指向处理程序。

	433 void open_softirq(int nr, void (*action)(struct softirq_action *))
	434 {
	435         softirq_vec[nr].action = action;
	436 }

例如，网络系统中注册接收和发送数据包的软中断：
	     net/core/dev.c
	7794         open_softirq(NET_TX_SOFTIRQ, net_tx_action);
	7795         open_softirq(NET_RX_SOFTIRQ, net_rx_action);

一个已经注册的软中断必须“触发”后才能在“适当的时机”被执行，raise\_softirq函数和\_\_raise\_softirq\_irqoff宏用于触发软中断，将一个软中断设置为挂起状态（pending）。raise_softirq最终也是通过调用\_\_raise\_softirq\_irqoff来实现的：
	
	398 /*
	399  * This function must run with irqs disabled!
	400  */
	401 inline void raise_softirq_irqoff(unsigned int nr)
	402 {
	403         __raise_softirq_irqoff(nr);
	404 
	405         /*
	406          * If we're in an interrupt or softirq, we're done
	407          * (this also catches softirq-disabled code). We will
	408          * actually run the softirq once we return from
	409          * the irq or softirq.
	410          *
	411          * Otherwise we wake up ksoftirqd to make sure we
	412          * schedule the softirq soon.
	413          */
	414         if (!in_interrupt())
	415                 wakeup_softirqd();
	416 }
	
	418 void raise_softirq(unsigned int nr)
	419 {
	420         unsigned long flags;
	421 
	422         local_irq_save(flags); //禁止中断
	423         raise_softirq_irqoff(nr); //触发软中断之前先要禁止中断
	424         local_irq_restore(flags); //开启中断
	425 }

\_\_raise\_softirq\_irqoff被定义为宏，以上述的枚举类型作为参数：

			arch/arm/include/asm/hardirq.h
	 10 typedef struct {
	 11         unsigned int __softirq_pending; //软中断挂起位图，每bit代表一种软中断
	 12 #ifdef CONFIG_SMP
	 13         unsigned int ipi_irqs[NR_IPI];
	 14 #endif
	 15 } ____cacheline_aligned irq_cpustat_t;
	
	  kernel/softirq.c
	 51 #ifndef __ARCH_IRQ_STAT
	 52 irq_cpustat_t irq_stat[NR_CPUS]  ____cacheline_aligned;  //cpu拥有一个软中断状态结构 体，
	                                    //所以即使是相同类型的软中断也可以在其他cpu上同时执行。
	 53 EXPORT_SYMBOL(irq_stat);
	 54 #endif
	
	  include/linux/irq_cpustat.h
	 19 #ifndef __ARCH_IRQ_STAT
	 20 extern irq_cpustat_t irq_stat[];                /* defined in asm/hardirq.h */
	 21 #define __IRQ_STAT(cpu, member) (irq_stat[cpu].member)
	 22 #endif
	 23 
	 24   /* arch independent irq_stat fields */
	 25 #define local_softirq_pending() \
	 26         __IRQ_STAT(smp_processor_id(), __softirq_pending)

 软中断并不是一旦触发就被执行，而是在之后的某个“适当的时机”执行，在Linux内核中共有三个“时机”：
 
 1）从一个硬中断返回时，即在irq\_exit中有机会执行软中断。
 
 2）在ksoftirqd内核线程中被执行，稍后分析。
 
 3）在那些显式检查或执行待处理软中断的代码中。如网络系统中的netif\_rx\_ni函数，local\_bh\_enable以及local\_bh\_enable\_ip等函数。
 
最终软中断都是通过do\_softirq或\_\_do\_softirq函数被执行的，其中do\_softirq也是通过调用\_\_do\_softirq来实现的。在内核需要执行软中断时，先通过local\_softirq\_pending函数检查是否有软中断挂起，如果有则调用do\_softirq来执行软中断。

	304 asmlinkage __visible void do_softirq(void)
	305 {
	306         __u32 pending;
	307         unsigned long flags;
	308 
	    /*如果in_interrupt()返回值为非0，
	     *则表示在中断上下文中调用了do_softirq，或者是在禁止了软中断后调用了do_softirq。
	     *这两种情况都不允许执行软中断，所以直接返回。
	     */
	309         if (in_interrupt())
	310                 return;
	311 
	312         local_irq_save(flags); //禁止本地CPU中断，并保存中断状态到flags临时变量中
	313 
	314         pending = local_softirq_pending(); //将本地软中断挂起状态位图存入局部变量pending
	315 
	316         if (pending) //如果有软中断挂起等待处理
	317                 do_softirq_own_stack();
	318 
	319         local_irq_restore(flags);
	320 }
	
	158 void do_softirq_own_stack(void)
	159 {
	160         struct thread_info *curctx;
	161         union irq_ctx *irqctx;
	162         u32 *isp;
	163 
	164         curctx = current_thread_info();
	165         irqctx = softirq_ctx[smp_processor_id()];
	166         irqctx->tinfo.task = curctx->task;
	167 
	168         /* build the stack frame on the softirq stack */
	169         isp = (u32 *) ((char *)irqctx + sizeof(struct thread_info));
	170 
	171         asm volatile (
	172                 "MOV   D0.5,%0\n"
	173                 "SWAP  A0StP,D0.5\n"
	174                 "CALLR D1RtP,___do_softirq\n"
	175                 "MOV   A0StP,D0.5\n"
	176                 :
	177                 : "r" (isp)
	178                 : "memory", "cc", "D1Ar1", "D0Ar2", "D1Ar3", "D0Ar4",
	179                   "D1Ar5", "D0Ar6", "D0Re0", "D1Re0", "D0.4", "D1RtP",
	180                   "D0.5"
	181                 );
	182 }
	
	230 asmlinkage __visible void __do_softirq(void)
	231 {
	232         unsigned long end = jiffies + MAX_SOFTIRQ_TIME;
	233         unsigned long old_flags = current->flags;
	234         int max_restart = MAX_SOFTIRQ_RESTART;
	235         struct softirq_action *h;
	236         bool in_hardirq;
	237         __u32 pending;
	238         int softirq_bit;
	239 
	240         /*
	241          * Mask out PF_MEMALLOC s current task context is borrowed for the
	242          * softirq. A softirq handled such as network RX might set PF_MEMALLOC
	243          * again if the socket is related to swap
	244          */
	245         current->flags &= ~PF_MEMALLOC;
	246 
	247         pending = local_softirq_pending();
	248         account_irq_enter_time(current);
	249 
	250         __local_bh_disable_ip(_RET_IP_, SOFTIRQ_OFFSET);
	251         in_hardirq = lockdep_softirq_start();
	252 
	253 restart:
	254         /* Reset the pending bitmask before enabling irqs */
	255         set_softirq_pending(0);
	256 
	257         local_irq_enable(); //开启本地CPU中断
	258 
	259         h = softirq_vec; //指向软中断向量表
	260 
	261         while ((softirq_bit = ffs(pending))) {  //遍历所有类型的软中断
	262                 unsigned int vec_nr;
	263                 int prev_count;
	264 
	265                 h += softirq_bit - 1;
	266 
	267                 vec_nr = h - softirq_vec;
	268                 prev_count = preempt_count();
	269 
	270                 kstat_incr_softirqs_this_cpu(vec_nr);
	271 
	272                 trace_softirq_entry(vec_nr);
	273                 h->action(h);
	274                 trace_softirq_exit(vec_nr);
	275                 if (unlikely(prev_count != preempt_count())) {
	276                         pr_err("huh, entered softirq %u %s %p with preempt_count %08x, exited with %08x?\n",
	277                                vec_nr, softirq_to_name[vec_nr], h->action,
	278                                prev_count, preempt_count());
	279                         preempt_count_set(prev_count);
	280                 }
	281                 h++; //指向软中断向量表的下一项
	282                 pending >>= softirq_bit;
	283         }
	284 
	285         rcu_bh_qs();
	286         local_irq_disable(); //禁止本地CPU中断
	287 
	288         pending = local_softirq_pending(); //再次获取软中断挂起状态位图存入pending
	
	  	/*如果pending不为0，表示有软中断等待处理，并且循环计数器max_restart不为0时，
		*回到restart 标号处运行，遍历并处理挂起的软中断.
		*/
	289         if (pending) {
	290                 if (time_before(jiffies, end) && !need_resched() &&
	291                     --max_restart)
	292                         goto restart;
		/*如果完成了max_restart次循环处理软中断后，仍有待处理的软中断（pending为非0），
		*调用wakeup_softirqd唤醒软中断处理内核线程处理本地CPU上的软中断。
		*/
	293 
	294                 wakeup_softirqd();
	295         }
	296 
	297         lockdep_softirq_end(in_hardirq);
	298         account_irq_exit_time(current);
	299         __local_bh_enable(SOFTIRQ_OFFSET); //最后开启中断下半部
	300         WARN_ON_ONCE(in_interrupt());
	301         tsk_restore_flags(current, old_flags, PF_MEMALLOC);
	302 }


最后，软中断的内核线程（ksoftirqd）。这实际上是内核为了解决大量软中断重复触发“霸占”CPU导致用户进程处于饥饿状态的一种折中方案。借助内核线程，可以保证在软中断负担很重的时候用户进程不会因为得不到处理时间而处于饥饿状态；同时也保证过量的软中断终究会得到处理。即使在空闲系统上，这种方案也表现良好，软中断处理得非常迅速，因为仅存的内核线程肯定会马上调度。

内核中有三个地方调用wakeup\_softirqd唤醒ksoftirqd内核线程：

1）在\_\_do_softirq函数中

2）在raise\_softirq\_irqoff函数中

3）在invoke\_softirq函数中

##5.4 tasklet 
判断一个类型的软中断处理函数是否可以执行的条件是软中断挂起位图\_\_softirq\_pending的相应bit是否被置1。在多处理器系统中，每个处理器拥有各自的软中断挂起位图\_\_softirq\_pending变量。由于执行软中断过程中是开启本地处理器的硬件中断并禁止本地处理器的软中断。因此，如果同一个软中断在执行过程中又被触发，那么另外一个处理器就有机会同时执行该软中断处理程序，这有助于提高软中断效率，但对于软中断处理函数的设计必须是完全可重入且线程安全的，对临界区做必要的加锁保护，从而增加了代码设计复杂度。

tasklet是另一种中断下半部的实现方式。tasklet是通过软中断实现的，其本质上就是软中断，不过它的代码设计复杂度要比软中断低，接口更简单。因此除非对性能要求很高像网络系统那样，否则实现中断下半部的方式一般选择tasklet。

tasklet\_struct结构体用于描述tasklet，其定义在include/linux/interrupt.h如下：

	487 struct tasklet_struct
	488 {
	489         struct tasklet_struct *next; //指向链表中下一个tasklet结构体
	490         unsigned long state;  //tasklet状态
	491         atomic_t count; //引用计数器
	492         void (*func)(unsigned long); //tasklet处理函数
	493         unsigned long data; //tasklet处理函数的参数
	494 };

其中state有2种可能的状态，分别是TASKLET\_STATE\_SCHED和TASKLET\_STATE\_RUN。TASKLET\_STATE\_SCHED表示tasklet已被调度等待执行；TASKLET_STATE_RUN表示该tasklet正在执行，该状态仅在多处理器系统中有效。

count是原子的引用计数器，用于禁止已调度的tasklet。如果count值不为0，则tasklet被禁止，不允许执行；只有当它为0时，tasklet才被激活，并且被设置为TASKLET\_STATE\_SCHED状态时，该tasklet才能够执行。

func函数指针指向tasklet的处理函数，data是给处理函数传递的参数。

内核中有两种创建tasklet的方式：静态创建和动态创建。内核提供了如下两个宏用于静态创建tasklet：

	496 #define DECLARE_TASKLET(name, func, data) \
	497 struct tasklet_struct name = { NULL, 0, ATOMIC_INIT(0), func, data }
	498 
	499 #define DECLARE_TASKLET_DISABLED(name, func, data) \
	500 struct tasklet_struct name = { NULL, 0, ATOMIC_INIT(1), func, data }

这两个宏的唯一区别就是count引用计数的初值设置。前一个宏把count设置为0，表示该tasklet处于激活状态；后一个宏把count设置为1，表示该tasklet处于禁止状态。

在内核中也可以通过一个tasklet指针动态创建一个tasklet结构，并使用如下接口来初始化一个tasklet：

	557 void tasklet_init(struct tasklet_struct *t,
	558                   void (*func)(unsigned long), unsigned long data)
	559 {
	560         t->next = NULL;
	561         t->state = 0;
	562         atomic_set(&t->count, 0);
	563         t->func = func;
	564         t->data = data;
	565 }

开发人员在定义并初始化一个tasklet实例后，必须将其注册到系统后才有机会被执行。内核中维护了两个per-CPU型的tasklet管理链表tasklet\_vec和tasklet\_hi\_vec，分别定义如下：

	441 struct tasklet_head {
	442         struct tasklet_struct *head;
	443         struct tasklet_struct **tail;
	444 };
	445 
	446 static DEFINE_PER_CPU(struct tasklet_head, tasklet_vec);
	447 static DEFINE_PER_CPU(struct tasklet_head, tasklet_hi_vec);

软中断时提到的枚举类型,其中HI_SOFTIRQ和TASKLET_SOFTIRQ用于实现tasklet，所以tasklet\_hi\_vec与tasklet\_vec的唯一区别就在于tasklet\_hi\_vec对应HI\_SOFTIRQ而tasklet\_vec对应TASKLET\_SOFTIRQ。在执行软中断时，HI\_SOFTIRQ优先于TASKLET\_SOFTIRQ执行。所以开发人员可以根据优先级的需求来决定自己创建的tasklet注册到tasklet\_hi\_vec还是tasklet\_vec中。

由于tasklet\_hi\_vec和tasklet\_vec非常类似，所以接下来仅分析与tasklet\_vec相关的内容。内核中提供了注册tasklet的接口：

	533 static inline void tasklet_schedule(struct tasklet_struct *t)
	534 {
	535         if (!test_and_set_bit(TASKLET_STATE_SCHED, &t->state))
	536                 __tasklet_schedule(t);
	537 }
	
	449 void __tasklet_schedule(struct tasklet_struct *t)
	450 {
	451         unsigned long flags;
	         // 为了避免多个内核控制路径共同修改tasklet_vec，必须禁止本地cpu中断
	452   
	453         local_irq_save(flags);
	454         t->next = NULL;
	455         *__this_cpu_read(tasklet_vec.tail) = t;
	456         __this_cpu_write(tasklet_vec.tail, &(t->next));
	         //激活TASKLET_SOFTIRQ软中断，确保下次调用do_softirq时执行该tasklet
	457         raise_softirq_irqoff(TASKLET_SOFTIRQ);
	458         local_irq_restore(flags); //开启本地cpu中断
	459 }

tasklet\_schedule首先检查待注册tasklet的状态字段是否已经被设置为TASKLET\_STATE\_SCHED。如果是，则表示该tasklet已经被注册，不需要重新注册；如果不是，则将该tasklet的状态字段设置为TASKLET\_STATE\_SCHED并调用\_\_tasklet\_schedule将tasklet结构体实例添加到tasklet\_vec管理链表中。

由于tasklet是基于软中断实现的，因此要执行tasklet需要一个软中断处理函数来调用tasklet的处理函数。在内核启动过程中调用了softirq\_init函数来完成一些软中断相关的初始化工作：

	634 void __init softirq_init(void)
	635 {
	636         int cpu;
	637 
	638         for_each_possible_cpu(cpu) {
	639                 per_cpu(tasklet_vec, cpu).tail =
	640                         &per_cpu(tasklet_vec, cpu).head; //初始化tasklet_vec链表
	641                 per_cpu(tasklet_hi_vec, cpu).tail =
	642                         &per_cpu(tasklet_hi_vec, cpu).head; //初始化tasklet_hi_vec链表
	643         }
	644 
	645         open_softirq(TASKLET_SOFTIRQ, tasklet_action); //注册tasklet的软中断处理函数
	646         open_softirq(HI_SOFTIRQ, tasklet_hi_action); //注册高优先级tasklet的软中断处理函数
	647 }


以tasklet\_action为例，分析如何执行已注册到tasklet\_vec中的tasklet。

	485 static void tasklet_action(struct softirq_action *a)
	486 {
	487         struct tasklet_struct *list;
	     /*由于软中断处理函数是在开中断环境下执行的，所以为了避免多个内核控制路径修改
	      *tasklet_vec，此处要禁止本地cpu中断。
	      */
	488 
	489         local_irq_disable();
	490         list = __this_cpu_read(tasklet_vec.head); //使用局部变量list保存tasklet_vec链表表头指针  
	491         __this_cpu_write(tasklet_vec.head, NULL); //将tasklet_vec链表清空 
	492         __this_cpu_write(tasklet_vec.tail, this_cpu_ptr(&tasklet_vec.head));
	493         local_irq_enable(); //开启本地cpu中断
	494 
	495         while (list) {      //遍历tasklet_vec链表
	496                 struct tasklet_struct *t = list;
	497 
	498                 list = list->next;
	499 
	500                 if (tasklet_trylock(t)) {  //判断count是否为0，如果为0，则表示该tasklet没有被禁止可以执行。
	501                         if (!atomic_read(&t->count)) {
	502                                 if (!test_and_clear_bit(TASKLET_STATE_SCHED,
	503                                                         &t->state))
	504                                         BUG();
	505                                 t->func(t->data); //执行tasklet处理函数
	506                                 tasklet_unlock(t); //清除tasklet的TASKLET_STATE_RUN状态
	507                                 continue; //跳到循环开始处继续执行下一个tasklet
	508                         }
			//如果count不为0，则表示该tasklet被禁止，执行下面的代码
	509                         tasklet_unlock(t); 
	510                 }
	511 
	512                 local_irq_disable(); //禁止本地cpu中断，原因同上。
	
	       /*将已被注册但没有得到执行的tasklet重新放到已清空的tasklet_vec链表中，以
	        *免造成tasklet丢失。
	        */
	513                 t->next = NULL;
	514                 *__this_cpu_read(tasklet_vec.tail) = t;
	515                 __this_cpu_write(tasklet_vec.tail, &(t->next));
	516                 __raise_softirq_irqoff(TASKLET_SOFTIRQ); //触发TASKLET_SOFTIRQ软中断
	517                 local_irq_enable(); //开始本地cpu中断
	518         }
	519 }


##5.5 工作队列
工作队列是另一种中断下半部实现方式，它与软中断和tasklet最主要区别在于其不在中断上下文执行而是在进程上下文执行。因此它拥有在进程上下文执行的所有优势，例如允许被重新调度以及睡眠等。一般的，如果推后执行的任务需要睡眠则选择工作队列来实现，否则选择软中断或tasklet。

工作队列子系统可以让你的驱动程序创建一个专门的工作者线程来处理需要推后的工作。不过，工作队列子系统提供了一个默认的工作者线程来处理这些工作。因此，工作队列最基本的表现形式就转变成了一个把需要推后执行的任务交给特定的通用线程这样一种接口。
###5.5.1 相关数据结构
首先介绍一下与工作队列相关的几个数据结构。内核将工作队列抽象成数据结构workqueue\_struct，定义如下：

	237 struct workqueue_struct {
	238         struct list_head        pwqs;           /* WR: all pwqs of this wq */
	239         struct list_head        list;           /* PR: list of all workqueues */
	240 
	241         struct mutex            mutex;          /* protects this wq */
	242         int                     work_color;     /* WQ: current work color */
	243         int                     flush_color;    /* WQ: current flush color */
	244         atomic_t                nr_pwqs_to_flush; /* flush in progress */
	245         struct wq_flusher       *first_flusher; /* WQ: first flusher */
	246         struct list_head        flusher_queue;  /* WQ: flush waiters */
	247         struct list_head        flusher_overflow; /* WQ: flush overflow list */
	248 
	249         struct list_head        maydays;        /* MD: pwqs requesting rescue */
	250         struct worker           *rescuer;       /* I: rescue worker */
	251 
	252         int                     nr_drainers;    /* WQ: drain in progress */
	253         int                     saved_max_active; /* WQ: saved pwq max_active */
	254 
	255         struct workqueue_attrs  *unbound_attrs; /* PW: only for unbound wqs */
	256         struct pool_workqueue   *dfl_pwq;       /* PW: only for unbound wqs */
	257 
	258 #ifdef CONFIG_SYSFS
	259         struct wq_device        *wq_dev;        /* I: for sysfs interface */
	260 #endif
	261 #ifdef CONFIG_LOCKDEP
	262         struct lockdep_map      lockdep_map;
	263 #endif
	264         char                    name[WQ_NAME_LEN]; /* I: workqueue name */
	265 
	266         /*
	267          * Destruction of workqueue_struct is sched-RCU protected to allow
	268          * walking the workqueues list without grabbing wq_pool_mutex.
	269          * This is used to dump all workqueues from sysrq.
	270          */
	271         struct rcu_head         rcu;
	272 
	273         /* hot fields used during command issue, aligned to cacheline */
	274         unsigned int            flags ____cacheline_aligned; /* WQ: WQ_* flags */
	275         struct pool_workqueue __percpu *cpu_pwqs; /* I: per-cpu pwqs */
	276         struct pool_workqueue __rcu *numa_pwq_tbl[]; /* PWR: unbound pwqs indexed by node */
	277 };

每一个work对应的数据结构为work\_struct，定义在include/linux/workqueue.h如下：

	100 struct work_struct {
	101         atomic_long_t data; /*传递给处理函数的参数*/
	102         struct list_head entry; /*将多个待处理工作节点形成链表*/
	103         work_func_t func; /*处理函数*/
	104 #ifdef CONFIG_LOCKDEP
	105         struct lockdep_map lockdep_map;
	106 #endif
	107 };


###5.5.2 使用工作队列
内核对外提供了3种创建工作队列的接口（宏定义）：

	413 #define create_workqueue(name)                                          \
	414         alloc_workqueue("%s", WQ_MEM_RECLAIM, 1, (name))
	415 #define create_freezable_workqueue(name)                                \
	416         alloc_workqueue("%s", WQ_FREEZABLE | WQ_UNBOUND | WQ_MEM_RECLAIM, \
	417                         1, (name))
	418 #define create_singlethread_workqueue(name)                             \
	419         alloc_ordered_workqueue("%s", WQ_MEM_RECLAIM, name)

通过对宏展开发现，这3个接口最终都是调用\_\_alloc\_workqueue\_key函数，其定义在kernel/workqueue.c如下：

	3787 struct workqueue_struct *__alloc_workqueue_key(const char *fmt,
	3788                                                unsigned int flags,
	3789                                                int max_active,
	3790                                                struct lock_class_key *key,
	3791                                                const char *lock_name, ...)
	3792 {
	3793         size_t tbl_size = 0;
	3794         va_list args;
	3795         struct workqueue_struct *wq;
	3796         struct pool_workqueue *pwq;
	3797 
	3798         /* see the comment above the definition of WQ_POWER_EFFICIENT */
	3799         if ((flags & WQ_POWER_EFFICIENT) && wq_power_efficient)
	3800                 flags |= WQ_UNBOUND;
	3801 
	3802         /* allocate wq and format name */
	3803         if (flags & WQ_UNBOUND)
	3804                 tbl_size = nr_node_ids * sizeof(wq->numa_pwq_tbl[0]);
	3805 
	3806         wq = kzalloc(sizeof(*wq) + tbl_size, GFP_KERNEL); /*处理函数*/
	3807         if (!wq)
	3808                 return NULL;
	3809 
	3810         if (flags & WQ_UNBOUND) {
	3811                 wq->unbound_attrs = alloc_workqueue_attrs(GFP_KERNEL); 
	3812                 if (!wq->unbound_attrs)
	3813                         goto err_free_wq;
	3814         }
	3815 
	3816         va_start(args, lock_name);
	3817         vsnprintf(wq->name, sizeof(wq->name), fmt, args);
	3818         va_end(args);
	3819 
	3820         max_active = max_active ?: WQ_DFL_ACTIVE;
	3821         max_active = wq_clamp_max_active(max_active, flags, wq->name);
	3822 
	3823         /* init wq */
	3824         wq->flags = flags;
	3825         wq->saved_max_active = max_active;
	3826         mutex_init(&wq->mutex);
	3827         atomic_set(&wq->nr_pwqs_to_flush, 0);
	3828         INIT_LIST_HEAD(&wq->pwqs);
	3829         INIT_LIST_HEAD(&wq->flusher_queue);
	3830         INIT_LIST_HEAD(&wq->flusher_overflow);
	3831         INIT_LIST_HEAD(&wq->maydays);
	3832 
	3833         lockdep_init_map(&wq->lockdep_map, lock_name, key, 0);
	3834         INIT_LIST_HEAD(&wq->list);
	3835 
	3836         if (alloc_and_link_pwqs(wq) < 0)
	3837                 goto err_free_wq;
	3838 
	3839         /*
	3840          * Workqueues which may be used during memory reclaim should
	3841          * have a rescuer to guarantee forward progress.
	3842          */
	3843         if (flags & WQ_MEM_RECLAIM) {
	3844                 struct worker *rescuer;
	3845 
	3846                 rescuer = alloc_worker(NUMA_NO_NODE);
	3847                 if (!rescuer)
	3848                         goto err_destroy;
	3849 
	3850                 rescuer->rescue_wq = wq;
	3851                 rescuer->task = kthread_create(rescuer_thread, rescuer, "%s",
	3852                                                wq->name);
	3853                 if (IS_ERR(rescuer->task)) {
	3854                         kfree(rescuer);
	3855                         goto err_destroy;
	3856                 }
	3857 
	3858                 wq->rescuer = rescuer;
	3859                 kthread_bind_mask(rescuer->task, cpu_possible_mask);
	3860                 wake_up_process(rescuer->task);
	3861         }
	3862 
	3863         if ((wq->flags & WQ_SYSFS) && workqueue_sysfs_register(wq))
	3864                 goto err_destroy;
	3865 
	3866         /*
	3867          * wq_pool_mutex protects global freeze state and workqueues list.
	3868          * Grab it, adjust max_active and add the new @wq to workqueues
	3869          * list.
	3870          */
	3871         mutex_lock(&wq_pool_mutex);
	3872 
	3873         mutex_lock(&wq->mutex);
	3874         for_each_pwq(pwq, wq)
	3875                 pwq_adjust_max_active(pwq);
	3876         mutex_unlock(&wq->mutex);
	3877 
	3878         list_add_tail_rcu(&wq->list, &workqueues);
	3879 
	3880         mutex_unlock(&wq_pool_mutex);
	3881 
	3882         return wq;
	3883 
	3884 err_free_wq:
	3885         free_workqueue_attrs(wq->unbound_attrs);
	3886         kfree(wq);
	3887         return NULL;
	3888 err_destroy:
	3889         destroy_workqueue(wq);
	3890         return NULL;
	3891 }

###5.5.3 销毁工作队列
当驱动程序不再需要先前创建的工作队列时，就需要使用destroy\_workqueue函数来销毁工作队列，其主要工作是释放在\_\_alloc\_workqueue\_key函数中分配的资源

	3900 void destroy_workqueue(struct workqueue_struct *wq)
	3901 {
	3902         struct pool_workqueue *pwq;
	3903         int node;
	3904 
	3905         /* drain it before proceeding with destruction */
	3906         drain_workqueue(wq);
	3907 
	3908         /* sanity checks */
	3909         mutex_lock(&wq->mutex);
	3910         for_each_pwq(pwq, wq) {
	3911                 int i;
	3912 
	3913                 for (i = 0; i < WORK_NR_COLORS; i++) {
	3914                         if (WARN_ON(pwq->nr_in_flight[i])) {
	3915                                 mutex_unlock(&wq->mutex);
	3916                                 return;
	3917                         }
	3918                 }
	3919 
	3920                 if (WARN_ON((pwq != wq->dfl_pwq) && (pwq->refcnt > 1)) ||
	3921                     WARN_ON(pwq->nr_active) ||
	3922                     WARN_ON(!list_empty(&pwq->delayed_works))) {
	3923                         mutex_unlock(&wq->mutex);
	3924                         return;
	3925                 }
	3926         }
	3927         mutex_unlock(&wq->mutex);
	3928 
	3929         /*
	3930          * wq list is used to freeze wq, remove from list after
	3931          * flushing is complete in case freeze races us.
	3932          */
	3933         mutex_lock(&wq_pool_mutex);
	3934         list_del_rcu(&wq->list); //将工作队列从链表中删除
	3935         mutex_unlock(&wq_pool_mutex);
	3936 
	3937         workqueue_sysfs_unregister(wq);
	3938 
	3939         if (wq->rescuer)
	3940                 kthread_stop(wq->rescuer->task);
	3941 
	3942         if (!(wq->flags & WQ_UNBOUND)) {
	3943                 /*
	3944                  * The base ref is never dropped on per-cpu pwqs.  Directly
	3945                  * schedule RCU free.
	3946                  */
	3947                 call_rcu_sched(&wq->rcu, rcu_free_wq);
	3948         } else {
	3949                 /*
	3950                  * We're the sole accessor of @wq at this point.  Directly
	3951                  * access numa_pwq_tbl[] and dfl_pwq to put the base refs.
	3952                  * @wq will be freed when the last pwq is released.
	3953                  */
	3954                 for_each_node(node) { //遍历每个node
	3955                         pwq = rcu_access_pointer(wq->numa_pwq_tbl[node]);
	3956                         RCU_INIT_POINTER(wq->numa_pwq_tbl[node], NULL);
	3957                         put_pwq_unlocked(pwq);
	3958                 }
	3959 
	3960                 /*
	3961                  * Put dfl_pwq.  @wq may be freed any time after dfl_pwq is
	3962                  * put.  Don't access it afterwards.
	3963                  */
	3964                 pwq = wq->dfl_pwq;
	3965                 wq->dfl_pwq = NULL;
	3966                 put_pwq_unlocked(pwq);
	3967         }
	3968 }