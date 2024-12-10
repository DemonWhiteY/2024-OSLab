# lab4:进程管理

## 练习一

alloc_proc函数主要是分配并且初始化一个PCB用于管理新进程的信息。proc_struct 结构的信息如下：

```c++
struct proc_struct {//进程控制块
    enum proc_state state;                      // 进程状态
    int pid;                                    // 进程ID
    int runs;                                   // 运行时间
    uintptr_t kstack;                           // 内核栈位置
    volatile bool need_resched;                 // 是否需要调度
    struct proc_struct *parent;                 // 父进程
    struct mm_struct *mm;                       // 进程的虚拟内存
    struct context context;                     // 进程上下文
    struct trapframe *tf;                       // 当前中断帧的指针
    uintptr_t cr3;                              // 当前页表地址
    uint32_t flags;                             // 进程
    char name[PROC_NAME_LEN + 1];               // 进程名字
    list_entry_t list_link;                     // 进程链表    
    list_entry_t hash_link;                    
};
```

根据手册里的提示，将`state`设为`PROC_UNINIT`，`pid`设为`-1`，`cr3`设置为`boot_cr3`，其余需要初始化的变量中，指针设为`NULL`，变量设置为`0`，具体实现方式如下：

```c++
static struct proc_struct *
alloc_proc(void) {
    struct proc_struct *proc = kmalloc(sizeof(struct proc_struct));
    if (proc != NULL) {
        proc->state = PROC_UNINIT;//给进程设置为未初始化状态
        proc->pid = -1;//未初始化的进程，其pid为-1
        proc->runs = 0;//初始化时间片,刚刚初始化的进程，运行时间一定为零 
        proc->kstack = 0;//内核栈地址,该进程分配的地址为0，因为还没有执行，也没有被重定位，因为默认地址都是从0开始的。
        proc->need_resched = 0;//不需要调度
        proc->parent = NULL;//父进程为空
        proc->mm = NULL;//虚拟内存为空
        memset(&(proc->context), 0, sizeof(struct context));//初始化上下文
        proc->tf = NULL;//中断帧指针为空
        proc->cr3 = boot_cr3;//页目录为内核页目录表的基址
        proc->flags = 0; //标志位为0
        memset(proc->name, 0, PROC_NAME_LEN);//进程名为0
    }
    return proc;
}
```

### 问题解答

**成员变量含义和作用：**

- `struct context context`：保存进程执行的上下文，也就是关键的几个寄存器的值。用于进程切换中还原之前的运行状态。在通过`proc_run`切换到CPU上运行时，需要调用`switch_to`将原进程的寄存器保存，以便下次切换回去时读出，保持之前的状态。
- `struct trapframe *tf`：保存了进程的中断帧（32个通用寄存器、异常相关的寄存器）。在进程从用户空间跳转到内核空间时，系统调用会改变寄存器的值。我们可以通过调整中断帧来使的系统调用返回特定的值。比如可以利用`s0`和`s1`传递线程执行的函数和参数；在创建子线程时，会将中断帧中的`a0`设为`0`。

## 练习二：为新创建的内核线程分配资源

### 代码实现

- 调用alloc_proc，首先获得一块用户信息块。
  
  ```c++
  proc = alloc_proc();
  ```
  
  并且将新线程的父线程设置为`current`：
  
  ```c
  proc->parent = current;
  ```

- 为进程分配一个内核栈。
  
  ```c
  setup_kstack(proc);
  ```

- 复制原进程的内存管理信息到新进程（但内核线程不必做此事）
  
  ```c
  copy_mm(clone_flags, proc);
  ```

- 复制原进程上下文到新进程
  
  ```c
  copy_thread(proc, stack, tf);
  ```

- 将新进程添加到进程列表
  
  ```c
  int pid = get_pid();
  proc->pid = pid;
  hash_proc(proc);
  list_add(&proc_list, &(proc->list_link));
  ```
  
  并且更新当前线程数量：
  
  ```
  nr_process++;
  ```

- 唤醒新进程

```c
proc->state = PROC_RUNNABLE;
```

- 返回新进程号

```c
ret = proc->pid;
```

### 问题：请说明ucore是否做到给每个新fork的线程一个唯一的id？

`ucore`能够做到给每个新`fork`的线程一个唯一的`id`，练习中通过`get_pid()`分配线程的`id`。其实现主要是对于每一个分配出的`id`，对线程链表进行遍历，判断是否有线程的`id`与之冲突，如果有，则将`id`增1，保证不和其他线程冲突，重新遍历链表；如果没有，就对线程链表进行更新，将当前`id`添加到链表中，以防冲突。通过这种方式，可以保证`ucore`可以给每个新`fork`的线程一个唯一的`id`。

## 练习三

### 代码实现

参考`schedule`函数里面的禁止和启用中断的过程，实现如下：

```c
bool intr_flag;
struct proc_struct *prev = current, *next = proc;
local_intr_save(intr_flag);
{
    current = proc;
    lcr3(next->cr3);
    switch_to(&(prev->context), &(next->context));
}
local_intr_restore(intr_flag);
```

在本实验中，创建且运行了两个内核线程：

- idleproc：第0个内核进程，完成内核中各个子系统的初始化，之后立即调度，执行其他进程。
- initproc：第一个内核进程，用于完成实验的功能而调度的内核进程。

## Challenge

这两个函数分别是kern/sync.h中定义的中断前后使能信号保存和退出的函数。在当时Lab1中有所涉及，主要作用是首先通过定义两个宏函数local_intr_save和local_intr_restore。这两个宏函数会调用两个内联函数intr_save和intr_restore。其中：

- 当调用`local_intr_save`时，会读取`sstatus`寄存器，判断`SIE`位的值，如果该位为1，则说明中断是能进行的，这时需要调用`intr_disable`将该位置0，并返回1，将`intr_flag`赋值为1；如果该位为0，则说明中断此时已经不能进行，则返回0，将`intr_flag`赋值为0。以此保证之后的代码执行时不会发生中断。

- 当需要恢复中断时，调用`local_intr_restore`，需要判断`intr_flag`的值，如果其值为1，则需要调用`intr_enable`将`sstatus`寄存器的`SIE`位置1，否则该位依然保持0。以此来恢复调用`local_intr_save`之前的`SIE`的值。
