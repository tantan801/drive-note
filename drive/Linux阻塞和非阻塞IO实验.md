## 第五十二章 Linux 阻塞和非阻塞 IO 实验

阻塞和非阻塞 IO 是 Linux 驱动开发里面很常见的两种设备访问模式，在编写驱动的时候一定要考虑到阻塞和非阻塞。本章我们就来学习一下阻塞和非阻塞 IO，以及如何在驱动程序中处理阻塞与非阻塞，如何在驱动程序使用等待队列和 poll 机制。

---

### 52.1 阻塞和非阻塞 IO

#### 52.1.1 阻塞和非阻塞简介
这里的 "IO" 并不是我们学习 STM32 或者其他单片机的时候所说的 "GPIO"(也就是引脚)。这里的 IO 指的是 Input/Output，也就是输入 / 输出，是应用程序对驱动设备的输入 / 输出操作。 当应用程序对设备驱动进行操作的时候，如果不能获取到设备资源，那么阻塞式 IO 就会将应用程序对应的线程挂起，直到设备资源可以获取为止。对于非阻塞 IO，应用程序对应的线程不会挂起，它要么一直轮询等待，直到设备资源可以使用，要么就直接放弃。

阻塞式 IO 如图 1-1所示：

![阻塞式 IO 访问](./photo/阻塞与非阻塞IO/1-1%20阻塞io访问.jpeg)

应用程序调用 read 函数从设备中读取数据，当设备不可用或数据未准备好的时候就会进入到休眠态。等设备可用的时候就会从休眠态唤醒，然后从设备中读取数据返回给应用程序。

非阻塞 IO 如图1-2所示：

![非阻塞 IO 访问](./photo/阻塞与非阻塞IO/1-2%20非阻塞IO访问.jpeg)

可以看出，应用程序使用非阻塞访问方式从设备读取数据，当设备不可用或数据未准备好的时候会立即向内核返回一个错误码，表示数据读取失败。应用程序会再次重新读取数据，这样一直往复循环，直到数据读取成功。

应用程序可以使用如下所示示例代码来实现阻塞访问：

**示例代码 52.1.1.1 应用程序阻塞读取数据**

```c
1 int fd;
2 int data = 0;
3
4 fd = open("/dev/xxx_dev", O_RDWR); /* 阻塞方式打开 */
5 ret = read(fd, &data, sizeof(data)); /* 读取数据 */
```

从示例代码 52.1.1.1 可以看出，对于设备驱动文件的默认读取方式就是阻塞式的，所以我们前面所有的例程测试 APP 都是采用阻塞 IO。

如果应用程序要采用非阻塞的方式来访问驱动设备文件，可以使用如下所示代码： 

**示例代码 52.1.1.2 应用程序非阻塞读取数据**

```c
1 int fd;
2 int data = 0;
3
4 fd = open("/dev/xxx_dev", O_RDWR | O_NONBLOCK); /* 非阻塞方式打开 */
5 ret = read(fd, &data, sizeof(data)); /* 读取数据 */
```

4 行使用 open 函数打开 "/dev/xxx_dev" 设备文件的时候添加了参数 "O_NONBLOCK"，表示以非阻塞方式打开设备，这样从设备中读取数据的时候就是非阻塞方式的了。

---

#### 52.1.2 等待队列

##### 1、等待队列头
阻塞访问最大的好处就是当设备文件不可操作的时候进程可以进入休眠态，这样可以将 CPU 资源让出来。但是，当设备文件可以操作的时候就必须唤醒进程，一般在中断函数里面完成唤醒工作。Linux 内核提供了等待队列 (wait queue) 来实现阻塞进程的唤醒工作，如果我们要在驱动中使用等待队列，必须创建并初始化一个等待队列头，等待队列头使用结构体 wait_queue_head_t 表示，wait_queue_head_t 结构体定义在文件 include/linux/wait.h 中，结构体内容如下所示：

**示例代码 52.1.2.1 wait_queue_head_t 结构体**

```c
39 struct __wait_queue_head {
40     spinlock_t lock;
41     struct list_head task_list;
42 };
43 typedef struct __wait_queue_head wait_queue_head_t;
```

定义好等待队列头以后需要初始化，使用init_waitqueue_head函数初始化等待队列头，函数原型如下

```c
void init_waitqueue_head(wait_queue_head_t *q)
```

参数 q 就是要初始化的等待队列头。 也可以使用宏 DECLARE_WAIT_QUEUE_HEAD 来一次性完成等待队列头的定义的初始化。

##### 2、等待队列项 
等待队列头就是一个等待队列的头部，每个访问设备的进程都是一个队列项，当设备不可用的时候就要将这些进程对应的等待队列项添加到等待队列里面。结构体wait_queue_t表示等待队列项，结构体内容如下：

**示例代码 52.1.2.2 wait_queue_t 结构体**

```c
struct __wait_queue {
    unsigned int flags;
    void *private;
    wait_queue_func_t func;
    struct list_head task_list;
};
typedef struct __wait_queue wait_queue_t;
```

使用宏DECLARE_WAITQUEUE定义并初始化一个等待队列项，宏的内容如下：

```c
DECLARE_WAITQUEUE(name, tsk)
```

name 就是等待队列项的名字，tsk 表示这个等待队列项属于哪个任务 (进程)，一般设置为current，在 Linux 内核中 current 相当于一个全局变量，表示当前进程。因此宏DECLARE_WAITQUEUE就是给当前正在运行的进程创建并初始化了一个等待队列项。

##### 3、将队列项添加 / 移除等待队列头 
当设备不可访问的时候就需要将进程对应的等待队列项添加到前面创建的等待队列头中，只有添加到等待队列头中以后进程才能进入休眠态。当设备可以访问以后再将进程对应的等待队列项从等待队列头中移除即可，等待队列项添加 API 函数如下：

```c
void add_wait_queue(wait_queue_head_t *q, wait_queue_t *wait)
```

函数参数和返回值含义如下： q：要删除的等待队列项所处的等待队列头。 wait：要删除的等待队列项。 返回值：无。

##### 4、等待唤醒 
当设备可以使用的时候就要唤醒进入休眠态的进程，唤醒可以使用如下两个函数：

```c
void wake_up(wait_queue_head_t *q)
void wake_up_interruptible(wait_queue_head_t *q)
```

参数 q 就是要唤醒的等待队列头，这两个函数会将这个等待队列头中的所有进程都唤醒。 wake_up 函数可以唤醒处于TASK_INTERRUPTIBLE 和 TASK_UNINTERRUPTIBLE 状态的进程，而wake_up_interruptible 函数只能唤醒处于TASK_INTERRUPTIBLE 状态的进程。

##### 5、等待事件 
除了主动唤醒以外，也可以设置等待队列等待某个事件，当这个事件满足以后就自动唤醒等待队列中的进程，和等待事件有关的 API 函数如表 52.1.2.1 所示：

**表 52.1.2.1 等待事件 API 函数**

| 函数 | 描述 |
|------|------|
| wait_event(wq, condition) | 等待以 wq 为等待队列头的等待队列被唤醒，前提是 condition 条件必须满足 (为真)，否则一直阻塞。此函数会将进程设置为 TASK_UNINTERRUPTIBLE 状态 |
| wait_event_timeout(wq, condition, timeout) | 功能和 wait_event 类似，但是此函数可以添加超时时间，以 jiffies 为单位。此函数有返回值，如果返回 0 的话表示超时时间到，而且 condition 为假。为 1 的话表示 condition 为真，也就是条件满足了。 |
| wait_event_interruptible(wq, condition) | 与 wait_event 函数类似，但是此函数将进程设置为 TASK_INTERRUPTIBLE，就是可以被信号打断。 |
| wait_event_interruptible_timeout(wq, condition, timeout) | 与 wait_event_timeout 函数类似，此函数也将进程设置为 TASK_INTERRUPTIBLE，可以被信号打断。 |

---

#### 52.1.3 轮询
如果用户应用程序以非阻塞的方式访问设备，设备驱动程序就要提供非阻塞的处理方式，也就是轮询。poll、epoll 和 select 可以用于处理轮询，应用程序通过 select、epoll 或 poll 函数来查询设备是否可以操作，如果可以操作的话就从设备读取或者向设备写入数据。当应用程序调用 select、epoll 或 poll 函数的时候设备驱动程序中的 poll 函数就会执行，因此需要在设备驱动程序中编写 poll 函数。我们先来看一下应用程序中使用的 select、poll 和 epoll 这三个函数。

##### 1、select 函数
select 函数原型如下：

```c
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout)
```

函数参数和返回值含义如下： 

- nfds：所要监视的这三类文件描述集合中，最大文件描述符加 1。 
- readfds、writefds 和 exceptfds：这三个指针指向描述符集合，这三个参数指明了关心哪些描述符、需要满足哪些条件等等，这三个参数都是 fd_set 类型的，fd_set 类型变量的每一个位都代表了一个文件描述符。readfds 用于监视指定描述符集的读变化，也就是监视这些文件是否可以读取，只要这些集合里面有一个文件可以读取那么 select 就会返回一个大于 0 的值表示文件可以读取。如果没有文件可以读取，那么就会根据 timeout 参数来判断是否超时。可以将 readfds 设置为 NULL，表示不关心任何文件的读变化。writefds 和 readfds 类似，只是 writefds 用于监视这些文件是否可以进行写操作。exceptfds 用于监视这些文件的异常。

比如我们现在要从一个设备文件中读取数据，那么就可以定义一个 fd_set 变量，这个变量要传递给参数 readfds。当我们定义好一个 fd_set 变量以后可以使用如下所示几个宏进行操作：

```c
void FD_ZERO(fd_set *set)
void FD_SET(int fd, fd_set *set)
void FD_CLR(int fd, fd_set *set)
int FD_ISSET(int fd, fd_set *set)
```

FD_ZERO 用于将 fd_set 变量的所有位都清零，FD_SET 用于将 fd_set 变量的某个位置 1，也就是向 fd_set 添加一个文件描述符，参数 fd 就是要加入的文件描述符。FD_CLR 用于将 fd_set 变量的某个位清零，也就是将一个文件描述符从 fd_set 中删除，参数 fd 就是要删除的文件描述符。FD_ISSET 用于测试一个文件是否属于某个集合，参数 fd 就是要判断的文件描述符。

timeout: 超时时间，当我们调用 select 函数等待某些文件描述符可以设置超时时间，超时时间使用结构体 timeval 表示，结构体定义如下所示：

```c
struct timeval {
    long tv_sec; /* 秒 */
    long tv_usec; /* 微秒 */
};
```

当 timeout 为 NULL 的时候就表示无限期的等待。

返回值：0，表示超时发生，但是没有任何文件描述符可以进行操作；-1，发生错误；其他值，可以进行操作的文件描述符个数。

使用 select 函数对某个设备驱动文件进行读非阻塞访问的操作示例如下所示： 

**示例代码 52.1.3.1 select 函数非阻塞读访问示例**

```c
1 void main(void)
2 {
3     int ret, fd; /* 要监视的文件描述符 */
4     fd_set readfds; /* 读操作文件描述符集 */
5     struct timeval timeout; /* 超时结构体 */
6
7     fd = open("dev_xxx", O_RDWR | O_NONBLOCK); /* 非阻塞式访问 */
8
9     FD_ZERO(&readfds); /* 清除readfds */
10    FD_SET(fd, &readfds); /* 将fd添加到readfds里面 */
11
12    /* 构造超时时间 */
13    timeout.tv_sec = 0;
14    timeout.tv_usec = 500000; /* 500ms */
15
16    ret = select(fd + 1, &readfds, NULL, NULL, &timeout);
17    switch (ret) {
18        case 0: /* 超时 */
19            printf("timeout!\r\n");
20            break;
21        case -1: /* 错误 */
22            printf("error!\r\n");
23            break;
24        default: /* 可以读取数据 */
25            if(FD_ISSET(fd, &readfds)) { /* 判断是否为fd文件描述符 */
26                /* 使用read函数读取数据 */
27            }
28            break;
29    }
30 }
```

##### 2、poll 函数 
在单个线程中，select 函数能够监视的文件描述符数量有最大的限制，一般为 1024，可以修改内核将监视的文件描述符数量改大，但是这样会降低效率！这个时候就可以使用 poll 函数，poll 函数本质上和 select 没有太大的差别，但是 poll 函数没有最大文件描述符限制，Linux 应用程序中 poll 函数原型如下所示：

```c
int poll(struct pollfd *fds, nfds_t nfds, int timeout)
```

函数参数和返回值含义如下： 
- fds：要监视的文件描述符集合以及要监视的事件，为一个数组，数组元素都是结构体 pollfd 类型的，pollfd 结构体如下所示：

```c
struct pollfd {
    int fd; /* 文件描述符 */
    short events; /* 请求的事件 */
    short revents; /* 返回的事件 */
};
```

fd 是要监视的文件描述符，如果 fd 无效的话那么 events 监视事件也就无效，并且 revents 返回 0。 events 是要监视的事件，可监视的事件类型如下所示：
- POLLIN 有数据可以读取。
- POLLPRI 有紧急的数据需要读取。
- POLLOUT 可以写数据。
- POLLERR 指定的文件描述符发生错误。
- POLLHUP 指定的文件描述符挂起。
- POLLNVAL 无效的请求。
- POLLRDNORM 等同于 POLLIN

revents 是返回参数，也就是返回的事件，由 Linux 内核设置具体的返回事件。 
- nfds：poll 函数要监视的文件描述符数量。 
- timeout：超时时间，单位为 ms。

返回值：返回 revents 域中不为 0 的 pollfd 结构体个数，也就是发生事件或错误的文件描述符数量；0，超时；-1，发生错误，并且设置 errno 为错误类型。

使用 poll 函数对某个设备驱动文件进行读非阻塞访问的操作示例如下所示： 

**示例代码 52.1.3.2 poll 函数读非阻塞访问示例**

```c
1 void main(void)
2 {
3     int ret;
4     int fd; /* 要监视的文件描述符 */
5     struct pollfd fds;
6
7     fd = open(filename, O_RDWR | O_NONBLOCK); /* 非阻塞式访问 */
8
9     /* 构造结构体 */
10    fds.fd = fd;
11    fds.events = POLLIN; /* 监视数据是否可以读取 */
12
13    ret = poll(&fds, 1, 500); /* 轮询文件是否可操作，超时500ms */
14    if (ret) { /* 数据有效 */
15        ......
16        /* 读取数据 */
17        ......
18    } else if (ret == 0) { /* 超时 */
19        ......
20    } else if (ret < 0) { /* 错误 */
21        ......
22    }
23 }
```

##### 3、epoll 函数 
传统的 selcet 和 poll 函数都会随着所监听的 fd 数量的增加，出现效率低下的问题，而且 poll 函数每次必须遍历所有的描述符来检查就绪的描述符，这个过程很浪费时间。为此，epoll 应运而生，epoll 就是为处理大并发而准备的，一般常常在网络编程中使用 epoll 函数。 

应用程序需要先使用 epoll_create 函数创建一个 epoll 句柄，epoll_create 函数原型如下：

```c
int epoll_create(int size)
```

函数参数和返回值含义如下： size：从 Linux2.6.8 开始此参数已经没有意义了，随便填写一个大于 0 的值就可以。 返回值：epoll 句柄，如果为 - 1 的话表示创建失败。

epoll 句柄创建成功以后使用 epoll_ctl 函数向其中添加要监视的文件描述符以及监视的事件，epoll_ctl 函数原型如下所示：

```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)
```

函数参数和返回值含义如下： 
- epfd：要操作的 epoll 句柄，也就是使用 epoll_create 函数创建的 epoll 句柄。 
- op：表示要对 epfd (epoll 句柄) 进行的操作，可以设置为：
  - EPOLL_CTL_ADD 向 epfd 添加文件参数 fd 表示的描述符。
  - EPOLL_CTL_MOD 修改参数 fd 的 event 事件。
  - EPOLL_CTL_DEL 从 epfd 中删除 fd 描述符。
- fd：要监视的文件描述符。 
- event：要监视的事件类型，为 epoll_event 结构体类型指针，epoll_event 结构体类型如下所示

```c
struct epoll_event {
    uint32_t events; /* epoll事件 */
    epoll_data_t data; /* 用户数据 */
};
```

结构体 epoll_event 的 events 成员变量表示要监视的事件，可选的事件如下所示：
- EPOLLIN 有数据可以读取。
- EPOLLOUT 可以写数据。
- EPOLLPRI 有紧急的数据需要读取。
- EPOLLERR 指定的文件描述符发生错误。
- EPOLLHUP 指定的文件描述符挂起。
- EPOLLET 设置 epoll 为边沿触发，默认触发模式为水平触发。
- EPOLLONESHOT 一次性的监视，当监视完成以后还需要再次监视某个 fd，那么就需要将 fd 重新添加到 epoll 里面。

上面这些事件可以进行 "或" 操作，也就是说可以设置监视多个事件。 返回值：0，成功；-1，失败，并且设置 errno 的值为相应的错误码。

一切都设置好以后应用程序就可以通过 epoll_wait 函数来等待事件的发生，类似 select 函数。epoll_wait 函数原型如下所示：

```c
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout)
```

函数参数和返回值含义如下： 
- epfd：要等待的 epoll。 
- events：指向 epoll_event 结构体的数组，当有事件发生的时候 Linux 内核会填写 events，调用者可以根据 events 判断发生了哪些事件。 
- maxevents：events 数组大小，必须大于 0。 
- timeout：超时时间，单位为 ms。

返回值：0，超时；-1，错误；其他值，准备就绪的文件描述符数量。

epoll 更多的是用在大规模的并发服务器上，因为在这种场合下 select 和 poll 并不适合。当设计到的文件描述符 (fd) 比较少的时候就适合用 selcet 和 poll，本章我们就使用 sellect 和 poll 这 两个函数。

---

#### 52.1.4 Linux 驱动下的 poll 操作函数
当应用程序调用 select 或 poll 函数来对驱动程序进行非阻塞访问的时候，驱动程序 file_operations 操作集中的 poll 函数就会执行。所以驱动程序的编写者需要提供对应的 poll 函数，poll 函数原型如下所示：

```c
unsigned int (*poll) (struct file *filp, struct poll_table_struct *wait)
```

函数参数和返回值含义如下： 
- filp：要打开的设备文件 (文件描述符)。 
- wait：结构体 poll_table_struct 类型指针，由应用程序传递进来的。一般将此参数传递给 poll_wait 函数。

返回值：向应用程序返回设备或者资源状态，可以返回的资源状态如下：
- POLLIN 有数据可以读取。
- POLLPRI 有紧急的数据需要读取。
- POLLOUT 可以写数据。
- POLLERR 指定的文件描述符发生错误。
- POLLHUP 指定的文件描述符挂起。
- POLLNVAL 无效的请求。
- POLLRDNORM 等同于 POLLIN，普通数据可读

我们需要在驱动程序的 poll 函数中调用 poll_wait 函数，poll_wait 函数不会引起阻塞，只是将应用程序添加到 poll_table 中，poll_wait 函数原型如下：

```c
void poll_wait(struct file * filp, wait_queue_head_t * wait_address, poll_table *p)
```

参数 wait_address 是要添加到 poll_table 中的等待队列头，参数 p 就是 poll_table，就是 file_operations 中 poll 函数的 wait 参数。

---

### 52.2 阻塞 IO 实验
在上一章 Linux 中断实验中，我们直接在应用程序中通过 read 函数不断的读取按键状态，当按键有效的时候就打印出按键值。这种方法有个缺点，那就是 imx6uirqApp 这个测试应用程序拥有很高的 CPU 占用率，大家可以在开发板中加载上一章的驱动程序模块 imx6uirq.ko，然后以后台运行模式打开 imx6uirqApp 这个测试软件，命令如下：

```shell
./imx6uirqApp /dev/imx6uirq &
```

测试驱动是否正常工作，如果驱动工作正常的话输入 "top" 命令查看 imx6uirqApp 这个应用程序的 CPU 使用率。

从结果可以看出，imx6uirqApp 这个应用程序的 CPU 使用率高达 99.6%，这仅仅是一个读取按键值的应用程序，这么高的 CPU 使用率显然是有问题的！原因就在于我们是直接在 while 循环中通过 read 函数读取按键值，因此 imx6uirqApp 这个软件会一直运行，一直读取按键值，CPU 使用率肯定就会很高。 最好的方法就是在没有有效的按键事件发生的时候，imx6uirqApp 这个应用程序应该处于休眠状态，当有按键事件发生以后 imx6uirqApp 这个应用程序才运行，打印出按键值，这样就会降低 CPU 使用率，本小节我们就使用阻塞 IO 来实现此功能。

---

#### 52.2.1 硬件原理图分析
本章实验硬件原理图参考key小节即可。

---

#### 52.2.2 实验程序编写

##### 1、驱动程序编写
本实验对应的例程路径为：开发板光盘 -> 2、Linux 驱动例程 -> 14_blockio。 

本章实验我们在上一章的 "13_irq" 实验的基础上完成，主要是对其添加阻塞访问相关的代码。新建名为 "14_blockio" 的文件夹，然后在 14_blockio 文件夹里面创建 vscode 工程，工作区命名为 "blockio"。将 "13_irq" 实验中的 imx6uirq.c 复制到 14_blockio 文件夹中，并重命名 为 blockio.c。接下来我们就修改 blockio.c 这个文件，在其中添加阻塞相关的代码，完成以后的 blockio.c 内容如下所示 (因为是在上一章实验的 imx6uirq.c 文件的基础上修改的，为了减少篇幅，下面的代码有省略)：

```c
#include <linux/types.h>
#include <linux/kernel.h>
......
#include <linux/interrupt.h>
#include <asm/mach/map.h>
#include <asm/uaccess.h>
#include <asm/io.h>

/***************************************************************
Copyright © ALIENTEK Co., Ltd. 1998-2029. All rights reserved.
文件名 : block.c
作者 : 左忠凯
版本 : V1.0
描述 : 阻塞IO访问
其他 : 无
论坛 : www.openedv.com
日志 : 初版V1.0 2019/7/26 左忠凯创建
***************************************************************/
#define IMX6UIRQ_CNT 1 /* 设备号个数 */
#define IMX6UIRQ_NAME "blockio" /* 名字 */
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
    atomic_t releasekey; /* 标记是否完成一次完成的按键 */
    struct timer_list timer; /* 定义一个定时器*/
    struct irq_keydesc irqkeydesc[KEY_NUM]; /* 按键描述数组 */
    unsigned char curkeynum; /* 当前按键号 */

    wait_queue_head_t r_wait; /* 读等待队列头 */
};

struct imx6uirq_dev imx6uirq; /* irq设备 */

/* @description : 中断服务函数，开启定时器
 * 定时器用于按键消抖。
 * @param - irq : 中断号
 * @param - dev_id : 设备结构。
 * @return : 中断执行结果
 */
static irqreturn_t key0_handler(int irq, void *dev_id)
{
    struct imx6uirq_dev *dev = (struct imx6uirq_dev*)dev_id;

    dev->curkeynum = 0;
    dev->timer.data = (volatile long)dev_id;
    mod_timer(&dev->timer, jiffies + msecs_to_jiffies(10));
    return IRQ_RETVAL(IRQ_HANDLED);
}

/* @description : 定时器服务函数，用于按键消抖，定时器到了以后
 * 再次读取按键值，如果按键还是处于按下状态就表示按键有效。
 * @param - arg : 设备结构变量
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
        atomic_set(&dev->releasekey, 1);
    }

    /* 唤醒进程 */
    if(atomic_read(&dev->releasekey)) { /* 完成一次按键过程 */
        /* wake_up(&dev->r_wait); */
        wake_up_interruptible(&dev->r_wait);
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
    char name[10];
    int ret = 0;
    ......
    /* 创建定时器 */
    init_timer(&imx6uirq.timer);
    imx6uirq.timer.function = timer_function;

    /* 初始化等待队列头 */
    init_waitqueue_head(&imx6uirq.r_wait);
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
 * @param - filp : 要打开的设备文件(文件描述符)
 * @param - buf : 返回给用户空间的数据缓冲区
 * @param - cnt : 要读取的数据长度
 * @param - offt : 相对于文件首地址的偏移
 * @return : 读取的字节数，如果为负值，表示读取失败
 */
static ssize_t imx6uirq_read(struct file *filp, char __user *buf, size_t cnt, loff_t *offt)
{
    int ret = 0;
    unsigned char keyvalue = 0;
    unsigned char releasekey = 0;
    struct imx6uirq_dev *dev = (struct imx6uirq_dev *) filp->private_data;

#if 0
    /* 加入等待队列，等待被唤醒,也就是有按键按下 */
    ret = wait_event_interruptible(dev->r_wait, atomic_read(&dev->releasekey));
    if (ret) {
        goto wait_error;
    }
#endif

    DECLARE_WAITQUEUE(wait, current); /* 定义一个等待队列 */
    if(atomic_read(&dev->releasekey) == 0) { /* 没有按键按下 */
        add_wait_queue(&dev->r_wait, &wait); /* 添加到等待队列头 */
        __set_current_state(TASK_INTERRUPTIBLE);/* 设置任务状态 */
        schedule(); /* 进行一次任务切换 */
        if(signal_pending(current)) { /* 判断是否为信号引起的唤醒 */
            ret = -ERESTARTSYS;
            goto wait_error;
        }
        __set_current_state(TASK_RUNNING); /*设置为运行状态 */
        remove_wait_queue(&dev->r_wait, &wait); /*将等待队列移除 */
    }
    keyvalue = atomic_read(&dev->keyvalue);
    releasekey = atomic_read(&dev->releasekey);
    ......
    return 0;

wait_error:
    set_current_state(TASK_RUNNING); /* 设置任务为运行态 */
    remove_wait_queue(&dev->r_wait, &wait); /* 将等待队列移除 */
    return ret;

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
    ......
    /* 5、初始化按键 */
    atomic_set(&imx6uirq.keyvalue, INVAKEY);
    atomic_set(&imx6uirq.releasekey, 0);
    keyio_init();
    return 0;
}

/* 论坛:www.openedv.com
 * @description : 驱动出口函数
 * @param : 无
 * @return : 无
 */
static void __exit imx6uirq_exit(void)
{
    unsigned i = 0;
    /* 删除定时器 */
    del_timer_sync(&imx6uirq.timer); /* 删除定时器 */
    ......
    class_destroy(imx6uirq.class);
}

module_init(imx6uirq_init);
module_exit(imx6uirq_exit);
MODULE_LICENSE("GPL");
```

第 32 行，修改设备文件名字为 "blockio"，当驱动程序加载成功以后就会在根文件系统中出现一个名为 "/dev/blockio" 的文件。
第 61 行，在设备结构体中添加一个等待队列头 r_wait，因为在 Linux 驱动中处理阻塞 IO 需要用到等待队列。
第 107~110 行，定时器中断处理函数执行，表示有按键按下，先在 107 行判断一下是否是一次有效的按键，如果是的话就通过 wake_up 或者 wake_up_interruptible 函数来唤醒等待队列 r_wait。
第 168 行，调用 init_waitqueue_head 函数初始化等待队列头 r_wait。
第 200~206 行，采用等待事件来处理 read 的阻塞访问，wait_event_interruptible 函数等待 releasekey 有效，也就是有按键按下。如果按键没有按下的话进程就会进入休眠状态，因为采用 了 wait_event_interruptible 函数，因此进入休眠态的进程可以被信号打断。
第 208~218 行，首先使用 DECLARE_WAITQUEUE 宏定义一个等待队列，如果没有按键 按下的话就使用 add_wait_queue 函数将当前任务的等待队列添加到等待队列头 r_wait 中。随后 调用__set_current_state 函数设置当前进程的状态为 TASK_INTERRUPTIBLE，也就是可以被信 号打断。接下来调用 schedule 函数进行一次任务切换，当前进程就会进入到休眠态。如果有按 键按下，那么进入休眠态的进程就会唤醒，然后接着从休眠点开始运行。在这里也就是从第 213 行开始运行，首先通过 signal_pending 函数判断一下进程是不是由信号唤醒的，如果是由信号 唤醒的话就直接返回 - ERESTARTSYS 这个错误码。如果不是由信号唤醒的 (也就是被按键唤醒 的) 那么就在 217 行调用__set_current_state 函数将任务状态设置为 TASK_RUNNING，然后在 218 行调用 remove_wait_queue 函数将进程从等待队列中删除。

使用等待队列实现阻塞访问重点注意两点： ①、将任务或者进程加入到等待队列头， ②、在合适的点唤醒等待队列，一般都是中断处理函数里面。

##### 2、编写测试 APP 
本节实验的测试 APP 直接使用第 51.3.3 小节所编写的 imx6uirqApp.c，将 imx6uirqApp.c 复制到本节实验文件夹下，并且重命名为 blockioApp.c，不需要修改任何内容。

---

#### 52.2.3 运行测试

##### 1、编译驱动程序和测试 APP

①、编译驱动程序 

编写 Makefile 文件，本章实验的 Makefile 文件和第四十章实验基本一样，只是将 obj-m 变量的值改为 blockio.o，Makefile 内容如下所示：

**示例代码 52.2.3.1 Makefile 文件**

```makefile
KERNELDIR := /home/zuozhongkai/linux/IMX6ULL/linux/temp/linux-imx-rel_imx_4.1.15_2.1.0_ga_alientek
CURRENT_PATH := $(shell pwd)
obj-m := blockio.o

build:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) modules
clean:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) clean
```

第 4 行，设置 obj-m 变量的值为 blockio.o。
输入如下命令编译出驱动模块文件：

```shell
make -j32
```

编译成功以后就会生成一个名为 "blockio.ko" 的驱动模块文件。

②、编译测试 APP 

输入如下命令编译测试 blockioApp.c 这个测试程序：

```shell
arm-linux-gnueabihf-gcc blockioApp.c -o blockioApp
```

编译成功以后就会生成 blockioApp 这个应用程序。

##### 2、运行测试 
将上一小节编译出来 blockio.ko 和 blockioApp 这两个文件拷贝到 rootfs/lib/modules/4.1.15 目录中，重启开发板，进入到目录 lib/modules/4.1.15 中，输入如下命令加载 blockio.ko 驱动模块：

```shell
depmod //第一次加载驱动的时候需要运行此命令
modprobe blockio.ko //加载驱动
```

驱动加载成功以后使用如下命令打开 blockioApp 这个测试 APP，并且以后台模式运行：

```shell
./blockioApp /dev/blockio &
```

按下开发板上的 KEY0 按键，测试 APP 就会打印出按键值。

输入 "top" 命令，查看 blockioAPP 这个应用 APP 的 CPU 使用率。 从图可以看出，当我们在按键驱动程序里面加入阻塞访问以后，blockioApp 这个应用程序的 CPU 使用率从 99.6% 降低到了 0.0%。大家注意，这里的 0.0% 并不是说 blockioApp 这个应用程序不使用 CPU 了，只是因为使用率太小了，CPU 使用率可能为 0.00001%，但是 top 只能显示出小数点后一位，因此就显示成了 0.0%。

我们可以使用 "kill" 命令关闭后台运行的应用程序，比如我们关闭掉 blockioApp 这个后台 运行的应用程序。首先输出Ctrl+C关闭 top 命令界面，进入到命令行模式。然后使用ps 命令查看一下 blockioApp 这个应用程序的 PID，使用kill -9 PID即可 "杀死" 指定 PID 的进程，示例：

```shell
kill -9 149
```
---

### 52.3 非阻塞IO实验

#### 52.3.1 硬件原理图分析 
本章实验硬件原理图参考key小节即可。

---

#### 52.3.2 实验程序编写

##### 1、驱动程序编写
本实验对应的例程路径为：开发板光盘 -> 2、Linux 驱动例程 -> 15_noblockio。

本章实验我们在 52.2 小节中的 "14_blockio" 实验的基础上完成，上一小节实验我们已经在驱动中添加了阻塞 IO 的代码，本小节我们继续完善驱动，加入非阻塞 IO 驱动代码。新建名为 "15_noblockio" 的文件夹，然后在 15_noblockio 文件夹中创建 vscode 工程，工作区命名为 "noblockio"。将 "14_blockio" 实验中的 blockio.c 复制到 15_noblockio 文件夹中，并重命名为 noblockio.c。接下来我们就修改 noblockio.c 这个文件，在其中添加非阻塞相关的代码，完成以后的 noblockio.c 内容如下所示 (因为是在上一小节实验的 blockio.c 文件的基础上修改的，为了减少篇幅，下面的代码有省略)：

**示例代码 52.3.3.1 noblockio.c 文件 (有省略)**

```c
#include ......
......
#include
#include
#include
#include
#include

/***************************************************************
Copyright © ALIENTEK Co., Ltd. 1998-2029. All rights reserved.
文件名 : noblock.c
作者 : 左忠凯
版本 : V1.0
描述 : 非阻塞IO访问
其他 : 无
论坛 : www.openedv.com
日志 : 初版V1.0 2019/7/26 左忠凯创建
***************************************************************/
#define IMX6UIRQ_CNT 1 /* 设备号个数 */
#define IMX6UIRQ_NAME "noblockio" /* 名字 */
......

/*
 * @description : 从设备读取数据
 * @param - filp : 要打开的设备文件(文件描述符)
 * @param - buf : 返回给用户空间的数据缓冲区
 * @param - cnt : 要读取的数据长度
 * @param - offt : 相对于文件首地址的偏移
 * @return : 读取的字节数，如果为负值，表示读取失败
 */
static ssize_t imx6uirq_read(struct file *filp, char __user *buf, size_t cnt, loff_t *offt)
{
	int ret = 0;
	unsigned char keyvalue = 0;
	unsigned char releasekey = 0;
	struct imx6uirq_dev *dev = (struct imx6uirq_dev *) filp->private_data;

	if (filp->f_flags & O_NONBLOCK) { /* 非阻塞访问 */
		if(atomic_read(&dev->releasekey) == 0) /* 没有按键按下 */
			return -EAGAIN;
	} else { /* 阻塞访问 */
		/* 加入等待队列，等待被唤醒,也就是有按键按下 */
		ret = wait_event_interruptible(dev->r_wait, atomic_read(&dev->releasekey));
		if (ret) {
			goto wait_error;
		}
	}
......
wait_error:
	return ret;
data_error:
	return -EINVAL;
}

/*
 * @description : poll函数，用于处理非阻塞访问
 * @param - filp : 要打开的设备文件(文件描述符)
 * @param - wait : 等待列表(poll_table)
 * @return : 设备或者资源状态，
 */
unsigned int imx6uirq_poll(struct file *filp, struct poll_table_struct *wait)
{
	unsigned int mask = 0;
	struct imx6uirq_dev *dev = (struct imx6uirq_dev *) filp->private_data;

	poll_wait(filp, &dev->r_wait, wait);

	if(atomic_read(&dev->releasekey)) { /* 按键按下 */
		mask = POLLIN | POLLRDNORM; /* 返回PLLIN */
	}
	return mask;
}

/* 设备操作函数 */
static struct file_operations imx6uirq_fops = {
	.owner = THIS_MODULE,
	.open = imx6uirq_open,
	.read = imx6uirq_read,
	.poll = imx6uirq_poll,
};

/*
 * @description : 驱动入口函数
 * @param : 无
 * @return : 无
 */
static int __init imx6uirq_init(void)
{
......
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
	unsigned i = 0;
	/* 删除定时器 */
	del_timer_sync(&imx6uirq.timer); /* 删除定时器 */
......
	class_destroy(imx6uirq.class);
}

module_init(imx6uirq_init);
module_exit(imx6uirq_exit);
MODULE_LICENSE("GPL");
```

第 32 行，修改设备文件名字为 "noblockio"，当驱动程序加载成功以后就会在根文件系统中出现一个名为 "/dev/noblockio" 的文件。 第 202~204 行，判断是否为非阻塞式读取访问，如果是的话就判断按键是否有效，也就是判断一下有没有按键按下，如果没有的话就返回 - EAGAIN。 第 241~252 行，imx6uirq_poll 函数就是 file_operations 驱动操作集中的 poll 函数，当应用程序调用 select 或者 poll 函数的时候 imx6uirq_poll 函数就会执行。第 246 行调用 poll_wait 函数将等待队列头添加到 poll_table 中，第 248~250 行判断按键是否有效，如果按键有效的话就向应用程序返回 POLLIN 这个事件，表示有数据可以读取。 第 259 行，设置 file_operations 的 poll 成员变量为 imx6uirq_poll。

##### 2、编写测试 APP
新建名为 noblockioApp.c 测试 APP 文件，然后在其中输入如下所示内容：

**示例代码 52.3.3.2 noblockioApp.c 文件代码**

```c
#include "stdio.h"
#include "unistd.h"
#include "sys/types.h"
#include "sys/stat.h"
#include "fcntl.h"
#include "stdlib.h"
#include "string.h"
#include "poll.h"
#include "sys/select.h"
#include "sys/time.h"
#include "linux/ioctl.h"

/***************************************************************
Copyright © ALIENTEK Co., Ltd. 1998-2029. All rights reserved.
文件名 : noblockApp.c
作者 : 左忠凯
版本 : V1.0
描述 : 非阻塞访问测试APP
其他 : 无
使用方法 ：./blockApp /dev/blockio 打开测试App
论坛 : www.openedv.com
日志 : 初版V1.0 2019/9/8 左忠凯创建
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
	struct pollfd fds;
	fd_set readfds;
	struct timeval timeout;
	unsigned char data;

	if (argc != 2) {
		printf("Error Usage!\r\n");
		return -1;
	}

	filename = argv[1];
	fd = open(filename, O_RDWR | O_NONBLOCK); /* 非阻塞访问 */
	if (fd < 0) {
		printf("Can't open file %s\r\n", filename);
		return -1;
	}

#if 0
	/* 构造结构体 */
	fds.fd = fd;
	fds.events = POLLIN;

	while (1) {
		ret = poll(&fds, 1, 500);
		if (ret) { /* 数据有效 */
			ret = read(fd, &data, sizeof(data));
			if(ret < 0) {
				/* 读取错误 */
			} else {
				if(data)
					printf("key value = %d \r\n", data);
			}
		} else if (ret == 0) { /* 超时 */
			/* 用户自定义超时处理 */
		} else if (ret < 0) { /* 错误 */
			/* 用户自定义错误处理 */
		}
	}
#endif

	while (1) {
		FD_ZERO(&readfds);
		FD_SET(fd, &readfds);
		/* 构造超时时间 */
		timeout.tv_sec = 0;
		timeout.tv_usec = 500000; /* 500ms */
		ret = select(fd + 1, &readfds, NULL, NULL, &timeout);
		switch (ret) {
		case 0: /* 超时 */
			/* 用户自定义超时处理 */
			break;
		case -1: /* 错误 */
			/* 用户自定义错误处理 */
			break;
		default: /* 可以读取数据 */
			if(FD_ISSET(fd, &readfds)) {
				ret = read(fd, &data, sizeof(data));
				if (ret < 0) {
					/* 读取错误 */
				} else {
					if (data)
						printf("key value=%d\r\n", data);
				}
			}
			break;
		}
	}

	close(fd);
	return ret;
}
```

第 52~73 行，这段代码使用 poll 函数来实现非阻塞访问，在 while 循环中使用 poll 函数不断的轮询，检查驱动程序是否有数据可以读取，如果可以读取的话就调用 read 函数读取按键数据。 第 75~101 行，这段代码使用 select 函数来实现非阻塞访问。

---

#### 52.3.3 运行测试

##### 1、编译驱动程序和测试 APP

①、编译驱动程序

编写 Makefile 文件，本章实验的 Makefile 文件和第四十章实验基本一样，只是将 obj-m 变量的值改为 noblockio.o，Makefile 内容如下所示：

**示例代码 52.3.3.1 Makefile 文件**

```makefile
KERNELDIR := /home/zuozhongkai/linux/IMX6ULL/linux/temp/linux-imx-rel_imx_4.1.15_2.1.0_ga_alientek
......
obj-m := noblockio.o
......
clean:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) clean
```

第 4 行，设置 obj-m 变量的值为 noblockio.o。
输入如下命令编译出驱动模块文件：

```shell
make -j32
```

编译成功以后就会生成一个名为 "noblockio.ko" 的驱动模块文件。

②、编译测试 APP

输入如下命令编译测试 noblockioApp.c 这个测试程序：

```shell
arm-linux-gnueabihf-gcc noblockioApp.c -o noblockioApp
```

编译成功以后就会生成 noblcokioApp 这个应用程序。

##### 2、运行测试
将上一小节编译出来 noblockio.ko 和 noblockioApp 这两个文件拷贝到 rootfs/lib/modules/4.1.15 目录中，重启开发板，进入到目录 lib/modules/4.1.15 中，输入如下命令加载驱动模块：

```shell
depmod //第一次加载驱动的时候需要运行此命令
modprobe noblockio.ko //加载驱动
```

驱动加载成功以后使用如下命令打开 noblockioApp 这个测试 APP，并且以后台模式运行：

```shell
./noblockioApp /dev/noblockio &
```