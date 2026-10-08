## 第五十一章 Linux中断实验

不管是裸机实验还是 Linux 下的驱动实验，中断都是频繁使用的功能，Linux 内核提供了完善的中断框架，我们只需要申请中断，然后注册中断处理函数即可，使用非常方便，不需要一系列复杂的寄存器配置。本章我们就来学习一下如何在 Linux 下使用中断。

---

### 51.1 Linux中断简介

#### 51.1.1 Linux 中断 API 函数

先来回顾一下裸机实验里面中断的处理方法：
①、使能中断，初始化相应的寄存器。
②、注册中断服务函数，也就是向 irqTable 数组的指定标号处写入中断服务函数。
③、中断发生以后进入 IRQ 中断服务函数，在 IRQ 中断服务函数在数组 irqTable 里面查找具体的中断处理函数，找到以后执行相应的中断处理函数。

在 Linux 内核中也提供了大量的中断相关的 API 函数，我们来看一下这些跟中断有关的 API 函数：

##### 1）中断号

每个中断都有一个中断号，通过中断号即可区分不同的中断，有的资料也把中断号叫做中断线。在 Linux 内核中使用一个 int 变量表示中断号。

##### 2）request_irq 函数

在 Linux 内核中要想使用某个中断是需要申请的，`request_irq` 函数用于申请中断，`request_irq` 函数可能会导致睡眠，因此不能在中断上下文或者其他禁止睡眠的代码段中使用 `request_irq` 函数。`request_irq` 函数会激活 (使能) 中断，所以不需要我们手动去使能中断，`request_irq` 函数原型如下：

```c
int request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags, const char *name, void *dev)
```

函数参数和返回值含义如下：
- irq：要申请中断的中断号。
- handler：中断处理函数，当中断发生以后就会执行此中断处理函数。
- flags：中断标志，可以在文件 `include/linux/interrupt.h` 里面查看所有的中断标志，这里我们介绍几个常用的中断标志：

| 标志 | 描述 |
|------|------|
| IRQF_SHARED | 多个设备共享一个中断线，共享的所有中断都必须指定此标志。如果使用共享中断的话，request_irq 函数的 dev 参数就是唯一区分他们的标志。 |
| IRQF_ONESHOT | 单次中断，中断执行一次就结束。 |
| IRQF_TRIGGER_NONE | 无触发。 |
| IRQF_TRIGGER_RISING | 上升沿触发。 |
| IRQF_TRIGGER_FALLING | 下降沿触发。 |
| IRQF_TRIGGER_HIGH | 高电平触发。 |
| IRQF_TRIGGER_LOW | 低电平触发。 |

比如 I.MX6U-ALPHA 开发板上的 KEY0 使用 GPIO1_IO18，按下 KEY0 以后为低电平，因此可以设置为下降沿触发，也就是将 flags 设置为 `IRQF_TRIGGER_FALLING`。表中的这些标志可以通过 `|` 来实现多种组合。

- name：中断名字，设置以后可以在 `/proc/interrupts` 文件中看到对应的中断名字。
- dev：如果将 flags 设置为 `IRQF_SHARED` 的话，dev 用来区分不同的中断，一般情况下将 dev 设置为设备结构体，dev 会传递给中断处理函数 `irq_handler_t` 的第二个参数。

返回值：0 中断申请成功，其他负值 中断申请失败，如果返回 `-EBUSY` 的话表示中断已经被申请了。

##### 3）free_irq 函数

使用中断的时候需要通过 `request_irq` 函数申请，使用完成以后就要通过 `free_irq` 函数释放掉相应的中断。如果中断不是共享的，那么 `free_irq` 会删除中断处理函数并且禁止中断。`free_irq` 函数原型如下所示：

```c
void free_irq(unsigned int irq, void *dev)
```

函数参数和返回值含义如下：
- irq：要释放的中断。
- dev：如果中断设置为共享 (`IRQF_SHARED`) 的话，此参数用来区分具体的中断。共享中断只有在释放最后中断处理函数的时候才会被禁止掉。

返回值：无。

##### 4）中断处理函数

使用 `request_irq` 函数申请中断的时候需要设置中断处理函数，中断处理函数格式如下所示：

```c
irqreturn_t (*irq_handler_t) (int, void *)
```

第一个参数是要中断处理函数要响应的中断号。第二个参数是一个指向 void 的指针，也就是个通用指针，需要与 `request_irq` 函数的 dev 参数保持一致。用于区分共享中断的不同设备，dev 也可以指向设备数据结构。中断处理函数的返回值为 `irqreturn_t` 类型，`irqreturn_t` 类型定义如下所示：

```c
enum irqreturn {
    IRQ_NONE = (0 << 0),
    IRQ_HANDLED = (1 << 0),
    IRQ_WAKE_THREAD = (1 << 1),
};
typedef enum irqreturn irqreturn_t;
```

可以看出 `irqreturn_t` 是个枚举类型，一共有三种返回值。一般中断服务函数返回值使用如下形式：

```c
return IRQ_RETVAL(IRQ_HANDLED)
```

##### 5）中断使能与禁止函数

常用的中断使能和禁止函数如下所示：

```c
void enable_irq(unsigned int irq)
void disable_irq(unsigned int irq)
```

`enable_irq` 和 `disable_irq` 用于使能和禁止指定的中断，irq 就是要禁止的中断号。`disable_irq` 函数要等到当前正在执行的中断处理函数执行完才返回，因此使用者需要保证不会产生新的中断，并且确保所有已经开始执行的中断处理程序已经全部退出。在这种情况下，可以使用另外一个中断禁止函数：

```c
void disable_irq_nosync(unsigned int irq)
```

`disable_irq_nosync` 函数调用以后立即返回，不会等待当前中断处理程序执行完毕。

上面三个函数都是使能或者禁止某一个中断，有时候我们需要关闭当前处理器的整个中断系统，也就是在学习 STM32 的时候常说的关闭全局中断，这个时候可以使用如下两个函数：

```c
local_irq_enable()
local_irq_disable()
```

`local_irq_enable` 用于使能当前处理器中断系统，`local_irq_disable` 用于禁止当前处理器中断系统。

假如 A 任务调用 `local_irq_disable` 关闭全局中断 10S，当关闭了 2S 的时候 B 任务开始运行，B 任务也调用 `local_irq_disable` 关闭全局中断 3S，3 秒以后 B 任务调用 `local_irq_enable` 函数将全局中断打开了。此时才过去 `2+3=5` 秒的时间，然后全局中断就被打开了，此时 A 任务要关闭 10S 全局中断的愿望就破灭了，然后 A 任务就 “生气了”，结果很严重，可能系统都要被 A 任务整崩溃。

为了解决这个问题，B 任务不能直接简单粗暴的通过 `local_irq_enable` 函数来打开全局中断，而是将中断状态恢复到以前的状态，要考虑到别的任务的感受，此时就要用到下面两个函数：

```c
local_irq_save(flags)
local_irq_restore(flags)
```

这两个函数是一对，`local_irq_save` 函数用于禁止中断，并且将中断状态保存在 flags 中。`local_irq_restore` 用于恢复中断，将中断到 flags 状态。

---

#### 51.1.2 上半部与下半部

在有些资料中也将上半部和下半部称为顶半部和底半部，都是一个意思。我们在使用 `request_irq` 申请中断的时候注册的中断服务函数属于中断处理的上半部，只要中断触发，那么中断处理函数就会执行。

我们都知道中断处理函数一定要快点执行完毕，越短越好，但是现实往往是残酷的，有些中断处理过程就是比较费时间，我们必须要对其进行处理，缩小中断处理函数的执行时间。比如电容触摸屏通过中断通知 SOC 有触摸事件发生，SOC 响应中断，然后通过 IIC 接口读取触摸坐标值并将其上报给系统。但是我们都知道 IIC 的速度最高也只有 400Kbit/S，所以在中断中通过 IIC 读取数据就会浪费时间。我们可以将通过 IIC 读取触摸数据的操作暂后执行，中断处理函数仅仅响应中断，然后清除中断标志位即可。

这个时候中断处理过程就分为了两部分：
- 上半部：上半部就是中断处理函数，那些处理过程比较快，不会占用很长时间的处理就可以放在上半部完成。
- 下半部：如果中断处理过程比较耗时，那么就将这些比较耗时的代码提出来，交给下半部去执行，这样中断处理函数就会快进快出。

因此，Linux 内核将中断分为上半部和下半部的主要目的就是实现中断处理函数的快进快出，那些对时间敏感、执行速度快的操作可以放到中断处理函数中，也就是上半部。剩下的所有工作都可以放到下半部去执行，比如在上半部将数据拷贝到内存中，关于数据的具体处理就可以放到下半部去执行。

至于哪些代码属于上半部，哪些代码属于下半部并没有明确的规定，一切根据实际使用情况去判断，这个就很考验驱动编写人员的功底了。这里有一些可以借鉴的参考点：
①、如果要处理的内容不希望被其他中断打断，那么可以放到上半部。
②、如果要处理的任务对时间敏感，可以放到上半部。
③、如果要处理的任务与硬件有关，可以放到上半部。
④、除了上述三点以外的其他任务，优先考虑放到下半部。

上半部处理很简单，直接编写中断处理函数就行了，关键是下半部该怎么做呢？Linux 内核提供了多种下半部机制。

##### 1）软中断

一开始 Linux 内核提供了 “bottom half” 机制来实现下半部，简称 “BH”。后面引入了软中断和 tasklet 来替代 “BH” 机制，完全可以使用软中断和 tasklet 来替代 BH，从 2.5 版本的 Linux 内核开始 BH 已经被抛弃了。

Linux 内核使用结构体 `softirq_action` 表示软中断，`softirq_action` 结构体定义在文件 `include/linux/interrupt.h` 中，内容如下：

```c
struct softirq_action
{
    void (*action)(struct softirq_action *);
};
```

在 `kernel/softirq.c` 文件中一共定义了 10 个软中断，如下所示：

```c
static struct softirq_action softirq_vec[NR_SOFTIRQS];
```

`NR_SOFTIRQS` 是枚举类型，定义在文件 `include/linux/interrupt.h` 中，定义如下：

```c
enum {
    HI_SOFTIRQ=0,       /* 高优先级软中断 */
    TIMER_SOFTIRQ,      /* 定时器软中断 */
    NET_TX_SOFTIRQ,     /* 网络数据发送软中断 */
    NET_RX_SOFTIRQ,     /* 网络数据接收软中断 */
    BLOCK_SOFTIRQ,
    BLOCK_IOPOLL_SOFTIRQ,
    TASKLET_SOFTIRQ,    /* tasklet软中断 */
    SCHED_SOFTIRQ,      /* 调度软中断 */
    HRTIMER_SOFTIRQ,    /* 高精度定时器软中断 */
    RCU_SOFTIRQ,        /* RCU软中断 */
    NR_SOFTIRQS
};
```

可以看出，一共有 10 个软中断，因此 `NR_SOFTIRQS` 为 10，因此数组 `softirq_vec` 有 10 个元素。`softirq_action` 结构体中的 `action` 成员变量就是软中断的服务函数，数组 `softirq_vec` 是个全局数组，因此所有的 CPU (对于 SMP 系统而言) 都可以访问到，每个 CPU 都有自己的触发和控制机制，并且只执行自己所触发的软中断。但是各个 CPU 所执行的软中断服务函数确是相同的，都是数组 `softirq_vec` 中定义的 `action` 函数。

要使用软中断，必须先使用 `open_softirq` 函数注册对应的软中断处理函数，`open_softirq` 函数原型如下：

```c
void open_softirq(int nr, void (*action)(struct softirq_action *))
```

函数参数和返回值含义如下：
- nr：要开启的软中断，在上面枚举中选择一个。
- action：软中断对应的处理函数。

返回值：没有返回值。

注册好软中断以后需要通过 `raise_softirq` 函数触发，`raise_softirq` 函数原型如下：

```c
void raise_softirq(unsigned int nr)
```

参数 nr：要触发的软中断。返回值：没有返回值。

软中断必须在编译的时候静态注册，Linux 内核使用 `softirq_init` 函数初始化软中断，`softirq_init` 函数定义在 `kernel/softirq.c` 文件里面，函数内容如下：

```c
void __init softirq_init(void)
{
    int cpu;
    for_each_possible_cpu(cpu) {
        per_cpu(tasklet_vec, cpu).tail = &per_cpu(tasklet_vec, cpu).head;
        per_cpu(tasklet_hi_vec, cpu).tail = &per_cpu(tasklet_hi_vec, cpu).head;
    }
    open_softirq(TASKLET_SOFTIRQ, tasklet_action);
    open_softirq(HI_SOFTIRQ, tasklet_hi_action);
}
```

从代码可以看出，`softirq_init` 函数默认会打开 `TASKLET_SOFTIRQ` 和 `HI_SOFTIRQ`。

##### 2）tasklet

tasklet 是利用软中断来实现的另外一种下半部机制，在软中断和 tasklet 之间，建议大家使用 tasklet。Linux 内核使用 `tasklet_struct` 结构体来表示 tasklet：

```c
struct tasklet_struct
{
    struct tasklet_struct *next;        /* 下一个tasklet */
    unsigned long state;                 /* tasklet状态 */
    atomic_t count;                      /* 计数器，记录对tasklet的引用数 */
    void (*func)(unsigned long);         /* tasklet执行的函数 */
    unsigned long data;                  /* 函数func的参数 */
};
```

`func` 函数就是 tasklet 要执行的处理函数，用户定义函数内容，相当于中断处理函数。

如果要使用 tasklet，必须先定义一个 tasklet，然后使用 `tasklet_init` 函数初始化 tasklet，`tasklet_init` 函数原型如下：

```c
void tasklet_init(struct tasklet_struct *t,void (*func)(unsigned long), unsigned long data);
```

参数：
- t：要初始化的 tasklet
- func：tasklet 的处理函数。
- data：要传递给 func 函数的参数

返回值：没有返回值。

也可以使用宏 `DECLARE_TASKLET` 来一次性完成 tasklet 的定义和初始化，`DECLARE_TASKLET` 定义在 `include/linux/interrupt.h` 文件中，定义如下：

```c
DECLARE_TASKLET(name, func, data)
```

其中 name 为要定义的 tasklet 名字，这个名字就是一个 `tasklet_struct` 类型的变量，func 就是 tasklet 的处理函数，data 是传递给 func 函数的参数。

在上半部，也就是中断处理函数中调用 `tasklet_schedule` 函数就能使 tasklet 在合适的时间运行，`tasklet_schedule` 函数原型如下：

```c
void tasklet_schedule(struct tasklet_struct *t)
```

参数 t：要调度的 tasklet，也就是 `DECLARE_TASKLET` 宏里面的 name。返回值：没有返回值。

tasklet 参考使用示例：

```c
/* 定义tasklet */
struct tasklet_struct testtasklet;
/* tasklet处理函数 */
void testtasklet_func(unsigned long data)
{
    /* tasklet具体处理内容 */
}
/* 中断处理函数 */
irqreturn_t test_handler(int irq, void *dev_id)
{
    ......
    /* 调度tasklet */
    tasklet_schedule(&testtasklet);
    ......
}
/* 驱动入口函数 */
static int __init xxxx_init(void)
{
    ......
    /* 初始化tasklet */
    tasklet_init(&testtasklet, testtasklet_func, data);
    /* 注册中断处理函数 */
    request_irq(xxx_irq, test_handler, 0, "xxx", &xxx_dev);
    ......
}
```

##### 3）工作队列

工作队列是另外一种下半部执行方式，工作队列在进程上下文执行，工作队列将要推后的工作交给一个内核线程去执行，因为工作队列工作在进程上下文，因此工作队列允许睡眠或重新调度。因此如果你要推后的工作可以睡眠那么就可以选择工作队列，否则的话就只能选择软中断或 tasklet。

Linux 内核使用 `work_struct` 结构体表示一个工作，内容如下 (省略掉条件编译)：

```c
struct work_struct {
    atomic_long_t data;
    struct list_head entry;
    work_func_t func; /* 工作队列处理函数 */
};
```

这些工作组织成工作队列，工作队列使用 `workqueue_struct` 结构体表示，内容如下 (省略掉条件编译)：

```c
struct workqueue_struct {
    struct list_head pwqs;
    struct list_head list;
    struct mutex mutex;
    int work_color;
    int flush_color;
    atomic_t nr_pwqs_to_flush;
    struct wq_flusher *first_flusher;
    struct list_head flusher_queue;
    struct list_head flusher_overflow;
    struct list_head maydays;
    struct worker *rescuer;
    int nr_drainers;
    int saved_max_active;
    struct workqueue_attrs *unbound_attrs;
    struct pool_workqueue *dfl_pwq;
    char name[WQ_NAME_LEN];
    struct rcu_head rcu;
    unsigned int flags ____cacheline_aligned;
    struct pool_workqueue __percpu *cpu_pwqs;
    struct pool_workqueue __rcu *numa_pwq_tbl[];
};
```

Linux 内核使用工作者线程 (worker thread) 来处理工作队列中的各个工作，Linux 内核使用 `worker` 结构体表示工作者线程：

```c
struct worker {
    union {
        struct list_head entry;
        struct hlist_node hentry;
    };
    struct work_struct *current_work;
    work_func_t current_func;
    struct pool_workqueue *current_pwq;
    bool desc_valid;
    struct list_head scheduled;
    struct task_struct *task;
    struct worker_pool *pool;
    struct list_head node;
    unsigned long last_active;
    unsigned int flags;
    int id;
    char desc[WORKER_DESC_LEN];
    struct workqueue_struct *rescue_wq;
};
```

每个 worker 都有一个工作队列，工作者线程处理自己工作队列中的所有工作。在实际的驱动开发中，我们只需要定义工作 (`work_struct`) 即可，关于工作队列和工作者线程我们基本不用去管。

简单创建工作很简单，直接定义一个 `work_struct` 结构体变量即可，然后使用 `INIT_WORK` 宏来初始化工作，`INIT_WORK` 宏定义如下：

```c
#define INIT_WORK(_work, _func)
```

`_work` 表示要初始化的工作，`_func` 是工作对应的处理函数。

也可以使用 `DECLARE_WORK` 宏一次性完成工作的创建和初始化，宏定义如下：

```c
#define DECLARE_WORK(n, f)
```

n 表示定义的工作 (`work_struct`)，f 表示工作对应的处理函数。

和 tasklet 一样，工作也是需要调度才能运行的，工作的调度函数为 `schedule_work`，函数原型如下所示：

```c
bool schedule_work(struct work_struct *work)
```

参数 work：要调度的工作。返回值：0 成功，其他值 失败。

工作队列参考使用示例：

```c
/* 定义工作(work) */
struct work_struct testwork;
/* work处理函数 */
void testwork_func_t(struct work_struct *work)
{
    /* work具体处理内容 */
}
/* 中断处理函数 */
irqreturn_t test_handler(int irq, void *dev_id)
{
    ......
    /* 调度work */
    schedule_work(&testwork);
    ......
}
/* 驱动入口函数 */
static int __init xxxx_init(void)
{
    ......
    /* 初始化work */
    INIT_WORK(&testwork, testwork_func_t);
    /* 注册中断处理函数 */
    request_irq(xxx_irq, test_handler, 0, "xxx", &xxx_dev);
    ......
}
```

---

#### 51.1.3 设备树中断信息节点

如果使用设备树的话就需要在设备树中设置好中断属性信息，Linux 内核通过读取设备树中的中断属性信息来配置中断。对于中断控制器而言，设备树绑定信息参考文档 `Documentation/devicetree/bindings/arm/gic.txt`。

打开 `imx6ull.dtsi` 文件，其中的 `intc` 节点就是 I.MX6ULL 的中断控制器节点，节点内容如下所示：

```dts
intc: interrupt-controller@00a01000 {
    compatible = "arm,cortex-a7-gic";
    #interrupt-cells = <3>;
    interrupt-controller;
    reg = <0x00a01000 0x1000>,
          <0x00a02000 0x100>;
};
```

- 第 2 行，compatible 属性值为 `"arm,cortex-a7-gic"`，在 Linux 内核源码中搜索 `"arm,cortex-a7-gic"` 即可找到 GIC 中断控制器驱动文件。
- 第 3 行，`#interrupt-cells` 和 `#address-cells`、`#size-cells` 一样。表示此中断控制器下设备的 cells 大小，对于设备而言，会使用 `interrupts` 属性描述中断信息，`#interrupt-cells` 描述了 `interrupts` 属性的 cells 大小，也就是一条信息有几个 cells。每个 cells 都是 32 位整形值，对于 ARM 处理的 GIC 来说，一共有 3 个 cells，这三个 cells 的含义如下：
  1. 第一个 cells：中断类型，0 表示 SPI 中断，1 表示 PPI 中断。
  2. 第二个 cells：中断号，对于 SPI 中断来说中断号的范围为 0~987，对于 PPI 中断来说中断号的范围为 0~15。
  3. 第三个 cells：标志，bit `[3:0]` 表示中断触发类型，为 1 的时候表示上升沿触发，为 2 的时候表示下降沿触发，为 4 的时候表示高电平触发，为 8 的时候表示低电平触发。bit `[15:8]` 为 PPI 中断的 CPU 掩码。
- 第 4 行，`interrupt-controller` 节点为空，表示当前节点是中断控制器。

对于 gpio 来说，gpio 节点也可以作为中断控制器，比如 `imx6ull.dtsi` 文件中的 `gpio5` 节点内容如下所示：

```dts
gpio5: gpio@020ac000 {
    compatible = "fsl,imx6ul-gpio", "fsl,imx35-gpio";
    reg = <0x020ac000 0x4000>;
    interrupts = <>, <>;
    gpio-controller;
    #gpio-cells = <2>;
    interrupt-controller;
    #interrupt-cells = <2>;
};
```

- 第 4 行，`interrupts` 描述中断源信息，对于 gpio5 来说一共有两条信息，中断类型都是 SPI，触发电平都是 `IRQ_TYPE_LEVEL_HIGH`。不同之处在于中断源，一个是 74，一个是 75。GPIO5 一共用了 2 个中断号，一个是 74，一个是 75。其中 74 对应 GPIO5_IO00~GPIO5_IO15 这低 16 个 IO，75 对应 GPIO5_IO16~GPIO5_IO31 这高 16 位 IO。
- 第 8 行，`interrupt-controller` 表明了 gpio5 节点也是个中断控制器，用于控制 gpio5 所有 IO 的中断。
- 第 9 行，将 `#interrupt-cells` 修改为 2。

打开 `imx6ull-alientek-emmc.dts` 文件，找到如下所示内容：

```dts
fxls8471@1e {
    compatible = "fsl,fxls8471";
    reg = <0x1e>;
    position = <0>;
    interrupt-parent = <&gpio5>;
    interrupts = <0 8>;
};
```

`fxls8471` 是 NXP 官方的 6ULL 开发板上的一个磁力计芯片，`fxls8471` 有一个中断引脚链接到了 I.MX6ULL 的 SNVS_TAMPER0 引脚上，这个引脚可以复用为 GPIO5_IO00。
- 第 5 行，`interrupt-parent` 属性设置中断控制器，这里使用 gpio5 作为中断控制器。
- 第 6 行，`interrupts` 设置中断信息，0 表示 GPIO5_IO00，8 表示低电平触发。

简单总结一下与中断有关的设备树属性信息：
①、`#interrupt-cells`，指定中断源的信息 cells 个数。
②、`interrupt-controller`，表示当前节点为中断控制器。
③、`interrupts`，指定中断号，触发方式等。
④、`interrupt-parent`，指定父中断，也就是中断控制器。

---

#### 51.1.4 获取中断号

编写驱动的时候需要用到中断号，中断信息已经写到了设备树里面，因此可以通过 `irq_of_parse_and_map` 函数从 `interupts` 属性中提取到对应的设备号，函数原型如下：

```c
unsigned int irq_of_parse_and_map(struct device_node *dev, int index)
```

函数参数和返回值含义如下：
- dev：设备节点。
- index：索引号，`interrupts` 属性可能包含多条中断信息，通过 index 指定要获取的信息。

返回值：中断号。

如果使用 GPIO 的话，可以使用 `gpio_to_irq` 函数来获取 gpio 对应的中断号，函数原型如下：

```c
int gpio_to_irq(unsigned int gpio)
```

函数参数和返回值含义如下：
- gpio：要获取的 GPIO 编号。

返回值：GPIO 对应的中断号。

---

### 51.2 硬件原理图分析

参考之前的按键输入

---

### 51.3 实验程序编写

本实验对应的例程路径为：开发板光盘-> 2、Linux驱动例程-> 13_irq。
本章实验我们驱动I.MX6U-ALPHA 开发板上的 KEY0 按键，不过我们采用中断的方式，并且采用定时器来实现按键消抖，应用程序读取按键值并且通过终端打印出来。通过本章我们可以学习到Linux内核中断的使用方法，以及对Linux内核定时器的回顾。

---

#### 51.3.1 修改设备树文件

本章实验使用到了按键KEY0，按键KEY0使用中断模式，因此需要在“key”节点下添加中断相关属性，添加完成以后的“key”节点内容如下所示：

**示例代码 51.3.1.1 key 节点信息**

```dts
key {
    #address-cells = <1>;
    #size-cells = <1>;
    compatible = "atkalpha-key";
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_key>;
    key-gpio = <&gpio1 18 GPIO_ACTIVE_LOW>; /* KEY0 */
    interrupt-parent = <&gpio1>;
    interrupts = <18 IRQ_TYPE_EDGE_BOTH>; /* FALLING RISING */
    status = "okay";
};
```

第 8 行，设置 `interrupt-parent` 属性值为 “gpio1”，因为 KEY0 所使用的 GPIO 为 GPIO1_IO18，也就是设置 KEY0 的 GPIO 中断控制器为 gpio1。
第 9 行，设置 `interrupts` 属性，也就是设置中断源，第一个 cells 的 18 表示 GPIO1 组的 18 号 IO。`IRQ_TYPE_EDGE_BOTH` 定义在文件 `include/linux/irq.h` 中，定义如下：

**示例代码 51.3.1.2 中断线状态**

```c
enum {
    IRQ_TYPE_NONE = 0x00000000,
    IRQ_TYPE_EDGE_RISING = 0x00000001,
    IRQ_TYPE_EDGE_FALLING = 0x00000002,
    IRQ_TYPE_EDGE_BOTH = (IRQ_TYPE_EDGE_FALLING | IRQ_TYPE_EDGE_RISING),
    IRQ_TYPE_LEVEL_HIGH = 0x00000004,
    IRQ_TYPE_LEVEL_LOW = 0x00000008,
    IRQ_TYPE_LEVEL_MASK = (IRQ_TYPE_LEVEL_LOW | IRQ_TYPE_LEVEL_HIGH),
    ......
};
```

从示例代码 51.3.1.2 中可以看出，`IRQ_TYPE_EDGE_BOTH` 表示上升沿和下降沿同时有效，相当于 KEY0 按下和释放都会触发中断。
设备树编写完成以后使用 `make dtbs` 命令重新编译设备树，然后使用新编译出来的 `imx6ull-alientek-emmc.dtb` 文件启动 Linux 系统。


#### 51.3.2 按键中断驱动程序编写
新建名为 "13_irq" 的文件夹，然后在 13_irq 文件夹里面创建 vscode 工程，工作区命名为 "imx6uirq"。工程创建好以后新建 imx6uirq.c 文件，在 imx6uirq.c 里面输入如下内容：

**示例代码 51.3.2.1 imx6uirq.c 文件代码**

```c
#include <linux/types.h>
#include <linux/kernel.h>
#include <linux/delay.h>
#include <linux/ide.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/errno.h>
#include <linux/gpio.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/of.h>
#include <linux/of_address.h>
#include <linux/of_gpio.h>
#include <linux/input.h>
#include <linux/semaphore.h>
#include <linux/timer.h>
#include <linux/irq.h>
#include <linux/interrupt.h>
#include <asm/mach/map.h>
#include <asm/uaccess.h>
#include <asm/io.h>

/***************************************************************
Copyright © ALIENTEK Co., Ltd. 1998-2029. All rights reserved.
文件名 : imx6uirq.c
作者 : 左忠凯
版本 : V1.0
描述 : Linux中断驱动实验
其他 : 无
论坛 : www.openedv.com
日志 : 初版V1.0 2019/7/26 左忠凯创建
***************************************************************/
#define IMX6UIRQ_CNT 1 /* 设备号个数 */
#define IMX6UIRQ_NAME "imx6uirq" /* 名字 */
#define KEY0VALUE 0X01 /* KEY0按键值 */
#define INVAKEY 0XFF /* 无效的按键值 */
#define KEY_NUM 1 /* 按键数量 */

/* 中断IO描述结构体 */
struct irq_keydesc {
    int gpio; /* gpio */
    int irqnum; /* 中断号 */
    unsigned char value; /* 按键对应的键值 */
    char name[10]; /* 名字 */
    irqreturn_t (*handler)(int, void *); /* 中断服务函数 */
};

/* imx6uirq设备结构体 */
struct imx6uirq_dev{
    dev_t devid; /* 设备号 */
    struct cdev cdev; /* cdev */
    struct class *class; /* 类 */
    struct device *device; /* 设备 */
    int major; /* 主设备号 */
    int minor; /* 次设备号 */
    struct device_node *nd; /* 设备节点 */
    atomic_t keyvalue; /* 有效的按键键值 */
    atomic_t releasekey; /* 标记是否完成一次完成的按键*/
    struct timer_list timer; /* 定义一个定时器*/
    struct irq_keydesc irqkeydesc[KEY_NUM]; /* 按键描述数组 */
    unsigned char curkeynum; /* 当前的按键号 */
};

struct imx6uirq_dev imx6uirq; /* irq设备 */

/* @description : 中断服务函数，开启定时器，延时10ms，
 * 定时器用于按键消抖。
 * @param - irq : 中断号
 * @param - dev_id : 设备结构。
 * @return : 中断执行结果
 */
static irqreturn_t key0_handler(int irq, void *dev_id)
{
    struct imx6uirq_dev *dev = (struct imx6uirq_dev *)dev_id;

    dev->curkeynum = 0;
    dev->timer.data = (volatile long)dev_id;
    mod_timer(&dev->timer, jiffies + msecs_to_jiffies(10));
    return IRQ_RETVAL(IRQ_HANDLED);
}

/* @description : 定时器服务函数，用于按键消抖，定时器到了以后
 * 再次读取按键值，如果按键还是处于按下状态就表示按键有效。
 * @param – arg : 设备结构变量
 * @return : 无
 */
void timer_function(unsigned long arg)
{
    unsigned char value;
    unsigned char num;
    struct irq_keydesc *keydesc;
    struct imx6uirq_dev *dev = (struct imx6uirq_dev *)arg;

    num = dev->curkeynum;
    keydesc = &dev->irqkeydesc[num];

    value = gpio_get_value(keydesc->gpio); /* 读取IO值 */
    if(value == 0){ /* 按下按键 */
        atomic_set(&dev->keyvalue, keydesc->value);
    }
    else{ /* 按键松开 */
        atomic_set(&dev->keyvalue, 0x80 | keydesc->value);
        atomic_set(&dev->releasekey, 1); /* 标记松开按键 */
    }
}

/*
 * @description : 按键IO初始化
 * @param : 无
 * @return : 无
 */
static int keyio_init(void)
{
    unsigned char i = 0;
    int ret = 0;

    imx6uirq.nd = of_find_node_by_path("/key");
    if (imx6uirq.nd== NULL){
        printk("key node not find!\r\n");
        return -EINVAL;
    }

    /* 提取GPIO */
    for (i = 0; i < KEY_NUM; i++) {
        imx6uirq.irqkeydesc[i].gpio = of_get_named_gpio(imx6uirq.nd, "key-gpio", i);
        if (imx6uirq.irqkeydesc[i].gpio < 0) {
            printk("can't get key%d\r\n", i);
        }
    }

    /* 初始化key所使用的IO，并且设置成中断模式 */
    for (i = 0; i < KEY_NUM; i++) {
        memset(imx6uirq.irqkeydesc[i].name, 0, sizeof(imx6uirq.irqkeydesc[i].name));
        sprintf(imx6uirq.irqkeydesc[i].name, "KEY%d", i);
        gpio_request(imx6uirq.irqkeydesc[i].gpio, imx6uirq.irqkeydesc[i].name);
        gpio_direction_input(imx6uirq.irqkeydesc[i].gpio);
        imx6uirq.irqkeydesc[i].irqnum = irq_of_parse_and_map(imx6uirq.nd, i);
#if 0
        imx6uirq.irqkeydesc[i].irqnum = gpio_to_irq(imx6uirq.irqkeydesc[i].gpio);
#endif
        printk("key%d:gpio=%d, irqnum=%d\r\n",i, imx6uirq.irqkeydesc[i].gpio,
        imx6uirq.irqkeydesc[i].irqnum);
    }
    /* 申请中断 */
    imx6uirq.irqkeydesc[0].handler = key0_handler;
    imx6uirq.irqkeydesc[0].value = KEY0VALUE;

    for (i = 0; i < KEY_NUM; i++) {
        ret = request_irq(imx6uirq.irqkeydesc[i].irqnum, imx6uirq.irqkeydesc[i].handler, IRQF_TRIGGER_FALLING|IRQF_TRIGGER_RISING, imx6uirq.irqkeydesc[i].name, &imx6uirq);
        if(ret < 0){
            printk("irq %d request failed!\r\n", imx6uirq.irqkeydesc[i].irqnum);
            return -EFAULT;
        }
    }

    /* 创建定时器 */
    init_timer(&imx6uirq.timer);
    imx6uirq.timer.function = timer_function;
    return 0;
}

/*
 * @description : 打开设备
 * @param – inode : 传递给驱动的inode
 * @param - filp : 设备文件，file结构体有个叫做private_data的成员变量
 * 一般在open的时候将private_data指向设备结构体。
 * @return : 0 成功;其他 失败
 */
static int imx6uirq_open(struct inode *inode, struct file *filp)
{
    filp->private_data = &imx6uirq; /* 设置私有数据 */
    return 0;
}

/*
 * @description : 从设备读取数据
 * @param – filp : 要打开的设备文件(文件描述符)
 * @param – buf : 返回给用户空间的数据缓冲区
 * @param - cnt : 要读取的数据长度
 * @param – offt : 相对于文件首地址的偏移
 * @return : 读取的字节数，如果为负值，表示读取失败
 */
static ssize_t imx6uirq_read(struct file *filp, char __user *buf, size_t cnt, loff_t *offt)
{
    int ret = 0;
    unsigned char keyvalue = 0;
    unsigned char releasekey = 0;
    struct imx6uirq_dev *dev = (struct imx6uirq_dev *) filp->private_data;

    keyvalue = atomic_read(&dev->keyvalue);
    releasekey = atomic_read(&dev->releasekey);

    if (releasekey) { /* 有按键按下 */
        if (keyvalue & 0x80) {
            keyvalue &= ~0x80;
            ret = copy_to_user(buf, &keyvalue, sizeof(keyvalue));
        } else {
            goto data_error;
        }
        atomic_set(&dev->releasekey, 0); /* 按下标志清零 */
    } else {
        goto data_error;
    }
    return 0;

data_error:
    return -EINVAL;
}

/* 设备操作函数 */
static struct file_operations imx6uirq_fops = {
    .owner = THIS_MODULE,
    .open = imx6uirq_open,
    .read = imx6uirq_read,
};

/*
 * @description : 驱动入口函数
 * @param : 无
 * @return : 无
 */
static int __init imx6uirq_init(void)
{
    /* 1、构建设备号 */
    if (imx6uirq.major) {
        imx6uirq.devid = MKDEV(imx6uirq.major, 0);
        register_chrdev_region(imx6uirq.devid, IMX6UIRQ_CNT, IMX6UIRQ_NAME);
    } else {
        alloc_chrdev_region(&imx6uirq.devid, 0, IMX6UIRQ_CNT, IMX6UIRQ_NAME);
        imx6uirq.major = MAJOR(imx6uirq.devid);
        imx6uirq.minor = MINOR(imx6uirq.devid);
    }

    /* 2、注册字符设备 */
    cdev_init(&imx6uirq.cdev, &imx6uirq_fops);
    cdev_add(&imx6uirq.cdev, imx6uirq.devid, IMX6UIRQ_CNT);

    /* 3、创建类 */
    imx6uirq.class = class_create(THIS_MODULE, IMX6UIRQ_NAME);
    if (IS_ERR(imx6uirq.class)) {
        return PTR_ERR(imx6uirq.class);
    }

    /* 4、创建设备 */
    imx6uirq.device = device_create(imx6uirq.class, NULL, imx6uirq.devid, NULL, IMX6UIRQ_NAME);
    if (IS_ERR(imx6uirq.device)) {
        return PTR_ERR(imx6uirq.device);
    }

    /* 5、初始化按键 */
    atomic_set(&imx6uirq.keyvalue, INVAKEY);
    atomic_set(&imx6uirq.releasekey, 0);
    keyio_init();
    return 0;
}

/*
 * @description : 驱动出口函数
 * @param : 无
 * @return : 无
 */
static void __exit imx6uirq_exit(void)
{
    unsigned int i = 0;
    /* 删除定时器 */
    del_timer_sync(&imx6uirq.timer);

    /* 释放中断 */
    for (i = 0; i < KEY_NUM; i++) {
        free_irq(imx6uirq.irqkeydesc[i].irqnum, &imx6uirq);
        gpio_free(imx6uirq.irqkeydesc[i].gpio);
    }
    cdev_del(&imx6uirq.cdev);
    unregister_chrdev_region(imx6uirq.devid, IMX6UIRQ_CNT);
    device_destroy(imx6uirq.class, imx6uirq.devid);
    class_destroy(imx6uirq.class);
}

module_init(imx6uirq_init);
module_exit(imx6uirq_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("tantan");
```

第 38~43 行，结构体 irq_keydesc 为按键的中断描述结构体，gpio 为按键 GPIO 编号，irqnum 为按键 IO 对应的中断号，value 为按键对应的键值，name 为按键名字，handler 为按键中断服务函数。使用 irq_keydesc 结构体即可描述一个按键中断。
第 47~60 行，结构体 imx6uirq_dev 为本例程设备结构体，第 55 行的 keyvalue 保存按键值，第 56 行的 releasekey 表示按键是否被释放，如果按键被释放表示发生了一次完整的按键过程。第 57 行的 timer 为按键消抖定时器，使用定时器进行按键消抖的原理已经在 19.1 小节讲解过了。第 58 行的数组 irqkeydesc 为按键信息数组，数组元素个数就是开发板上的按键个数，I.MX6U-ALPHA 开发板上只有一个按键，因此 irqkeydesc 数组只有一个元素。第 59 行的 curkeynum 表示当前按键。
第 62 行，定义设备结构体变量 imx6uirq。 第 70~78 行，key0_handler 函数，按键 KEY0 中断处理函数，参数 dev_id 为设备结构体，也就是 imx6uirq。第 74 行设置 curkeynum=0，表示当前按键为 KEY0，第 76 行使用 mod_timer 函数启动定时器，定时器周期为 10ms。
第 85~103，timer_function 函数，定时器定时处理函数，参数 arg 是设备结构体，也就是 imx6uirq，在此函数中读取按键值。第 95 行通过 gpio_get_value 函数读取按键值。如果为 0 的话就表示按键被按下去了，按下去的话就设置 imx6uirq 结构体的 keyvalue 成员变量为按键的键值，比如 KEY0 按键的话按键值就是 KEY0VALUE=0。如果按键值为 1 的话表示按键被释放了，按键释放了的话就将 imx6uirq 结构体的 keyvalue 成员变量的最高位置 1，表示按键值有效，也就是将 keyvalue 与 0x80 进行或运算，表示按键松开了，并且设置 imx6uirq 结构体的 releasekey 成员变量为 1，表示按键释放，一次有效的按键过程发生。
第 110~159 行，keyio_init 函数，按键 IO 初始化函数，在驱动入口函数里面会调用 keyio_init 来初始化按键 IO。第 131~142 行轮流初始化所有的按键，包括申请 IO、设置 IO 为输入模式、从设备树中获取 IO 的中断号等等。第 136 行通过 irq_of_parse_and_map 函数从设备树中获取按键 IO 对应的中断号。也可以使用 gpio_to_irq 函数将某个 IO 设置为中断状态，并且返回其中断号。第 144 和 145 行设置 KEY0 按键对应的按键中断处理函数为 key0_handler、KEY0 的按键值为 KEY0VALUE。第 147~153 行轮流调用 request_irq 函数申请中断号，设置中断触发模式为 IRQF_TRIGGER_FALLING 和 IRQF_TRIGGER_RISING，也就是上升沿和下降沿都可以触发中断。最后，第 156 行初始化定时器，并且设置定时器的定时处理函数。
第 168~172 行，imx6uirq_open 函数，对应应用程序的 open 函数。 第 182~207 行，imx6uirq_read 函数，对应应用程序的 read 函数。此函数向应用程序返回按键值。首先判断 imx6uirq 结构体的 releasekey 成员变量值是否为 1，如果为 1 的话表示有一次有效按键发生，否则的话就直接返回 - EINVAL。当有按键事件发生的话就要向应用程序发送按键值，首先判断按键值的最高位是否为 1，如果为 1 的话就表示按键值有效。如果按键值有效的话就将最高位清除，得到真实的按键值，然后通过 copy_to_user 函数返回给应用程序。向应用程序发送按键值完成以后就将 imx6uirq 结构体的 releasekey 成员变量清零，准备下一次按键操作。
第 210~214 行，按键中断驱动操作函数集 imx6uirq_fops。 第 221~253 行，驱动入口函数，第 250 和 251 行分别初始化 imx6uirq 结构体中的原子变量 keyvalue 和 releasekey，第 252 行调用 keyio_init 函数初始化按键所使用的 IO。 第 261~276 行，驱动出口函数，第 265 行调用 del_timer_sync 函数删除定时器，第 268~270 行轮流释放申请的所有按键中断。

---

#### 51.3.3 编写测试 APP
测试 APP 要实现的内容很简单，通过不断的读取 /dev/imx6uirq 文件来获取按键值，当按键按下以后就会将获取到的按键值输出在终端上，新建名为 imx6uirqApp.c 的文件，然后输入如下所示内容：

**示例代码 51.3.3.1 imx6uirqApp.c 文件代码**

```c
#include "stdio.h"
#include "unistd.h"
#include "sys/types.h"
#include "sys/stat.h"
#include "fcntl.h"
#include "stdlib.h"
#include "string.h"
#include "linux/ioctl.h"
/***************************************************************
Copyright © ALIENTEK Co., Ltd. 1998-2029. All rights reserved.
文件名 : imx6uirqApp.c
作者 : 左忠凯
版本 : V1.0
描述 : 定时器测试应用程序
其他 : 无
使用方法 ：./imx6uirqApp /dev/imx6uirq 打开测试App
论坛 : www.openedv.com
日志 初版V1.0 2019/7/26 左忠凯创建
***************************************************************/

/*
 * @description : main主程序
 * @param - argc : argv数组元素个数
 * @param - argv : 具体参数
 * @return : 0 成功;其他 失败
 */
int main(int argc, char *argv[])
{
    int fd;
    int ret = 0;
    char *filename;
    unsigned char data;
    if (argc != 2) {
        printf("Error Usage!\r\n");
        return -1;
    }

    filename = argv[1];
    fd = open(filename, O_RDWR);
    if (fd < 0) {
        printf("Can't open file %s\r\n", filename);
        return -1;
    }

    while (1) {
        ret = read(fd, &data, sizeof(data));
        if (ret < 0) { /* 数据读取错误或者无效 */

        } else { /* 数据读取正确 */
            if (data) /* 读取到数据 */
            printf("key value = %#X\r\n", data);
        }
    }
    close(fd);
    return ret;
}
```

第 45~53 行的 while 循环用于不断的读取按键值，如果读取到有效的按键值就将其输出到终端上。

---

### 51.4 运行测试

#### 51.4.1 编译驱动程序和测试 APP

1、编译驱动程序 
编写 Makefile 文件，本章实验的 Makefile 文件和第四十章实验基本一样，只是将 obj-m 变量的值改为 imx6uirq.o，Makefile 内容如下所示：

**示例代码 51.4.1.1 Makefile 文件**

```makefile
KERNELDIR := /home/zuozhongkai/linux/IMX6ULL/linux/temp/linux-imx-rel_imx_4.1.15_2.1.0_ga_alientek
CURRENT_PATH := $(shell pwd)
obj-m := imx6uirq.o

build:
    $(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) modules
clean:
    $(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) clean
```

第 4 行，设置 obj-m 变量的值为 imx6uirq.o。
输入如下命令编译出驱动模块文件：

```shell
make -j32
```

编译成功以后就会生成一个名为 "imx6uirq.ko" 的驱动模块文件。

2、编译测试 APP
输入如下命令编译测试 imx6uirqApp.c 这个测试程序：

```shell
arm-linux-gnueabihf-gcc imx6uirqApp.c -o imx6uirqApp
```

编译成功以后就会生成 imx6uirqApp 这个应用程序。

---

#### 51.4.2 运行测试
将上一小节编译出来 imx6uirq.ko 和 imx6uirqApp 这两个文件拷贝到 rootfs/lib/modules/4.1.15 目录中，重启开发板，进入到目录 lib/modules/4.1.15 中，输入如下命令加载 imx6uirq.ko 驱动模块：

```shell
depmod //第一次加载驱动的时候需要运行此命令
modprobe imx6uirq.ko //加载驱动
```

驱动加载成功以后可以通过查看 /proc/interrupts 文件来检查一下对应的中断有没有被注册上，输入如下命令：

```shell
cat /proc/interrupts
```

从结果可以看出 imx6uirq.c 驱动文件里面的 KEY0 中断已经存在了，触发方式为边沿 (Edge)，中断号为 49。
接下来使用如下命令来测试中断：

```shell
./imx6uirqApp /dev/imx6uirq
```

按下开发板上的 KEY0 键，终端就会输出按键值，从打印信息可以看出，按键值获取成功，并且不会有按键抖动导致的误判发生，说明按键消抖工作正常。
如果要卸载驱动的话输入如下命令即可：

```shell
rmmod imx6uirq.ko
```