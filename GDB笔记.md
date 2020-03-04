### GDB学习

#### 一、基本操作

r	run，启动被调试进程

​	带参数启动：(gdb) set args a b c

​							(gdb) r

​	或者直接用：(gdb) r a b c

c	continue，继续执行到下一个断点

n	next，执行下一行，不进入子函数

s	step，执行下一行，会进入子函数

b	breakpoints，下断点

p	print，打印变量值

l	list，显示当前的代码	

i	info，显示信息，eg：i b(info breakpoints列出所有断点),  i threads(列出所有线程)

f	file，加载新程序作为待调试程序，eg：f /root/a.out 会加载a.out作为待调试程序

bt	backtrace,打印当前线程栈信息

u	up，栈向上一层

d	down，栈向下一层	

t	threads，根据id切换线程：t 2





#### 二、断点

##### 1 .按函数下

b transactions_controller_impl::push_receive_transaction

##### 2.按文件行数下

b /root/eos/unittests/transaction_executer_tests.cpp:677

##### 3.带条件的断点

b sgnt_broadcasted.size()==9

##### 4.组合断点：在某行，下一个带条件的断点

b /root/eos/unittests/transaction_executer_tests.cpp:316 if  tmp_cnt_bc_sgnts==9 && line==702

##### 5.对某类异常下断点

b std::__throw_bad_function_call	//对std::bad_function异常下断点

##### 删除断点

d 1 // delete 1 # 按编号删除一个断点

d 	//delete # 删除所有断点



#### 三、捕捉异常

eg：catch throw std::bad_function

```
如下用法与WSL中的gdb参数用法不一致，只供参考
(gdb) help catch
Set catchpoints to catch events.
Raised signals may be caught:
        catch signal              - all signals
        catch signal <signame>    - a particular signal
Raised exceptions may be caught:
        catch throw               - all exceptions, when thrown
        catch throw <exceptname>  - a particular exception, when thrown
        catch catch               - all exceptions, when caught
        catch catch <exceptname>  - a particular exception, when caught
Thread or process events may be caught:
        catch thread_start        - any threads, just after creation
        catch thread_exit         - any threads, just before expiration
        catch thread_join         - any threads, just after joins
Process events may be caught:
        catch start               - any processes, just after creation
        catch exit                - any processes, just before expiration
        catch fork                - calls to fork()
        catch vfork               - calls to vfork()
        catch exec                - calls to exec()
Dynamically-linked library events may be caught:
        catch load                - loads of any library
        catch load <libname>      - loads of a particular library
        catch unload              - unloads of any library
        catch unload <libname>    - unloads of a particular library
The act of your program's execution stopping may also be caught:
        catch stop

C++ exceptions may be caught:
        catch throw               - all exceptions, when thrown
        catch catch               - all exceptions, when caught
Ada exceptions may be caught:
        catch exception           - all exceptions, when raised
        catch exception <name>    - a particular exception, when raised
        catch exception unhandled - all unhandled exceptions, when raised
        catch assert              - all failed assertions, when raised

Do "help set follow-fork-mode" for info on debugging your program
after a fork or vfork is caught.

Do "help breakpoints" for info on other commands dealing with breakpoints.
```



#### 四、只调试当前线程

​	set scheduler-locking on



#### 五、自动化

加载完符号后，-x参数文件里面的命令会按顺序自动执行

gdb /root/eos_build/unittests/unit_test -x /root/eos/gdb_1.cmd



#### 六、查看内存段

**1：(gdb) p (*buff->data())@8**   输出buff->data()指向的地址的8个字节
$19 = "L\000\000\000\r\246\026\270"



**2：我们可以使用examine命令（缩写为x）来查看内存地址中的值。examine命令的语法如下所示：**

**x/<n/f/u> <addr>**



**(gdb) x/10db buff->data()**  16进制打印buff->data()指向的地址，输出10字节
0x7fffac001480:	76	0	0	0	13	-90	22	-72
0x7fffac001488:	13	79



**(gdb) x/8wx 0x7fbf948a9a20** 16进制打印8个uint32

0x7fbf948a9a20:	0xbed5eba0	0x00007f68	0x3707f9f0	0x07007f6a
0x7fbf948a9a30:	0x3707fdb8	0x00007f6a	0x01d6cde0	0x00000000



#### 七、info的使用

**(gdb) i args**:		当前frame函数入参

**(gdb) i locals**:	  函数内部变量

**(gdb) i threads**: 

**(gdb) i address r** 

-- Describe where symbol SYM is stored
Symbol "r" is multi-location:
  Range 0xf154b6-0xf154ba: a complex DWARF expression:
     0: DW_OP_breg5 0 [$rdi]

  Range 0xf154ba-0xf15597: a complex DWARF expression:
     0: DW_OP_breg6 -64 [$rbp]
     2: DW_OP_deref

**(gdb) info all-registers**

-- List of all registers and their contents

**(gdb) i frame**  当前栈帧信息
Stack level 0, frame at 0x7fc5fcfc6bd0:
 rip = 0x7fc648751207 in tcache_get (malloc.c:2943); saved rip = 0x7fc6490f4258
 inlined into frame 1
 source language c.
 Arglist at unknown address.
 Locals at unknown address, Previous frame's sp in rsp


#### 八、窗口分割显示：layout的使用

layout：用于分割窗口，可以一边查看代码，一边测试。主要有以下几种用法：
layout src：显示源代码窗口
layout asm：显示汇编窗口
layout regs：显示源代码/汇编和寄存器窗口
layout split：显示源代码和汇编窗口
layout next：显示下一个layout
layout prev：显示上一个layout
Ctrl + L：刷新窗口
Ctrl + x，再按1：单窗口模式，显示一个窗口
Ctrl + x，再按2：双窗口模式，显示两个窗口
Ctrl + x，再按a：回到传统模式，即退出layout，回到执行layout之前的调试窗口。