# lab3 #
-------------------------------------------------------------------- 
## 思考题 ##
### 思考3.1-init 的逆序插入 ###
　　因为每次都是空闲链表的头取一个空进程，逆序插入使得先插入的先被使用，形成空闲的队列。
### 思考3.2-地址空间初始化 ###
　　参考内存布局：　　　
	![MMU.H](\lab3-graph\mmu.h.PNG)
　　对照代码：
　　![copy_pgdir](\lab3-graph\copy_pgdir.PNG)
　　发现：UTOP以上的虚拟地址分为两部分：内核区的代码，以及用户的页表和各种进程控制块的信息。这些部分应该被所有的进程所共享，因此需要拷贝内核区的相应位置的页目录信息即可。
　　参考了pmap.c中mip_vm_init函数，对于UTOP与ULIM有了新的认识
	![usage_UPAGE_UENV.PNG](\lab3-graph\usage_UPAGE_UENV.PNG)
　　这里表明，内核的pgdir中虚拟地址UPAGES与UENVS对应的区域分别映射了内存中页结构与进程控制块结构的物理地址，因为用户区的pgdir在这一部分拷贝了内核区的boot_pgdir,使得用户进程可以通过两级页表的方式访问所有的页结构和所有的进程控制块。（P.S USER_VTP的作用还不能理解，因为没有相关的使用）  
　　正因为用户区的访问时通过二级页表的访问来实现的，因此pgdir[PDX(UVPT)]需要记录用户页目录的物理地址，根据定义应该存在env_cr3中
### 思考3.3-user_data 的作用 ###
　　在本次实验的场景中，user_data是用来传入env的地址的。
　　![user_data](\lab3-graph\user_data.PNG)
　　而uerdata在load_elf中是这样使用的
　　![apply_userdata](\lab3-graph\apply_userdata.PNG)
　　作为回调函数的一个参数，在调用load_elf时通过userdata的形式传入，在load_elf使用函数load_icode_mapper时，作为参数传入。如此，增加了代码的灵活性，便于维护。  
　　相关的应用有，[例子](http://zhidao.baidu.com/question/504520449.html)：matlab GUI中有用相关方式来传递参数
### 思考3.4-位置的含义 ###
　　首先查看enter_point的赋值，在load_elf中  
![enter_p_00](\lab3-graph\enter_p_00.PNG)
![enter_p_01](\lab3-graph\enter_p_01.PNG)  
　　entry_point被赋予二进制文件相应位置的值，这个值应该不依赖二进制加载的具体位置，在编译之初就定好了，因此应该是统一的。为了验证，输出entry_point的数值（如图),是统一的虚拟地址 0x803fff94,对应于内核的栈顶左右的位置。   
![enter_p_result](\lab3-graph\enter_p_result.PNG)  
　　这里，有一点比较奇怪：输出显示，镜像只加载了一次，因此两个进程的PC应该是不同的物理地址，所以，尽管地址高于0x8000,0000,仍旧需要查表来找到物理地址，也就是同一个虚拟地址对应了两个不同的物理地址，为了验证，输出了中断，结果显示如图，确实产生了缺页中断，验证了猜想
![entry_int_check](\lab3-graph\entry_int_check.PNG)  
### 思考3.5-进程上下文的PC 值 ###   
　　根据观察发现进程的相关运行信息，上下文信息，存储在内存的TIMESTACK区域，其中包括寄存器中的相关信息等等。   
![TIMESTACK_MEANING_2](\lab3-graph\TIMESTACK_MEANING_2.PNG)  
### 思考3.6-TIMESTACK 的含义 ###   
　　我认为TIMESTACK是用来存储当前进程运行状态的地方。参考的是env_destory函数，在销毁一个进程的时候，如果该进程在运行中，就覆盖掉该进程对应存储的信息。目前，我认为：KERNEL_SP存储的是内核进程的状态。在删除当前运行的进程的时候，控制权交给内核，因此把内核代码的上下文恢复，删除旧的进程的上下文信息
![TIMESTACK_MEANING_1](\lab3-graph\TIMESTACK_MEANING_1.PNG)  
　　另外一个证据是中断的PC与TIMESTACK中存储的EPC相同，表明TIMESTACK存存储了中断产生时候的EPC寄存器中的数值   
![TIMESTACK_MEANING_2](\lab3-graph\TIMESTACK_MEANING_2.PNG)  
### 思考3.7-不公的调度 ###　　　
　　不公平产生，是因为每次分配给进程的时间，不仅包含了进程执行的时间，还包含了调度所用的时间。该调度进程2时，计数器为0，经过一次自加就可以调度进程2；然而，当该调度进程1时，计数器需要遍历所有的进程控制块数组后，才能回到开头，调度进程1。因此进程1的执行时间被耽误了，导致了不公现象。   
![unfair](\lab3-graph\unfair.PNG) 
　　解决的方案是：如果只有两个进程，计数器为奇数调度进程1，为偶数调度进程2，如此就可以实现公平   
##疑问与思考    
　　目前，不能理解第一次准备启动进程A时，KERENL_SP 中的epc 为什么会存400？
![Q_ker_epc_origin](\lab3-graph\Q_ker_epc_origin.PNG) 　　　

　　本次实验，阅读了比较多的汇编代码，初步了解了Ｃ语言与汇编语言之间的调用跳转方法，感觉收获很大。但是感觉对于TIMESTACK和KERNEL_SP的理解依旧不充分。另外，意识到了权限位设定的重要性。之前用户栈的权限位忘记设为可写，导致程序发生错误，花费了许多时间调试。

