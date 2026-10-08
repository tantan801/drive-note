## 第四十七章 Linux 并发与竞争

---

### 47.1 并发与竞争简介

#### 1）并发 & 竞争核心概念

- **并发**：多个任务（线程/中断）**同时访问**同一个共享资源
- **竞争**：并发访问共享资源，导致数据被篡改、错乱
  > 示例：两个进程同时打印 → 内容交叉混杂；多个进程同时读写全局变量 → 变量值随机错误

- **共享资源**：多个任务都能读写的数据，**保护的是数据，不是代码！**
  - ✅ **需要保护**：全局变量、设备结构体、硬件寄存器
  - ❌ **不需要保护**：函数内部局部变量（每个任务有独立栈空间）

- **临界区**：操作共享资源的那一段代码，要求**同一时刻只能有一个任务执行**
- **原子访问**：操作不可拆分，一旦开始必须一次性执行完毕，中途不能被打断

#### 2）Linux 并发产生的 4 个根源

| 根源 | 说明 | 示例 |
|------|------|------|
| 多线程并发 | 多进程/内核线程同时执行，同时进入驱动访问共享数据 | 多个应用同时调用 LED 驱动 |
| 内核抢占 | Linux 2.6 后支持，低优先级任务执行途中可被高优先级任务打断 | 低优先级线程操作寄存器时，被高优先级线程抢占 |
| 中断并发 | 中断优先级远高于线程，线程运行随时会被硬件中断打断；中断服务函数若操作同一共享数据 → 竞争 | 线程刚拿到锁，被中断打断，中断也申请该锁 → 死锁 |
| SMP 多核并发 | 多核 SOC 真正**并行同时执行代码**，多核同时访问同一份内存 | 两个 CPU 核心同时写同一个全局变量 |

> **并发竞争 BUG 特点**：概率性出现，难以复现，极难调试。
> 
> **开发原则**：写驱动阶段就必须考虑并发保护，绝对不要写完驱动再补！

---

### 47.2 原子操作

#### 47.2.1 原子操作简介

原子操作 = **硬件保证的不可分割操作**，专门保护**整型变量/单个 bit 位**。

C 语言一行代码编译后会拆成多条汇编指令，执行中途可能被抢占；原子操作由硬件保证这一组汇编**一次性执行完，不会被打断**。

> **原子操作局限**：仅能保护简单整型/单个 bit 位，**无法保护结构体、多段代码组成的临界区**。

#### 47.2.2 原子整形操作 API（`atomic_t`）

仅 32 位 SOC（如 I.MX6ULL）使用，64 位 SOC 用 `atomic64_t`（API 前缀为 `atomic64_`）。

```c
typedef struct {
    int counter;
} atomic_t;
```
- **atomic_t a;** 定义原子变量
- **atomic_t b = ATOMIC_INIT(0);** 初始化原子变量

| API | 作用 |
|-----|------|
| `ATOMIC_INIT(i)` | 定义原子变量并初始化为 `i` |
| `atomic_read(&v)` | 读取原子变量值 |
| `atomic_set(&v, i)` | 设置原子变量值为 `i` |
| `atomic_add(i, &v)` | 原子变量 + `i` |
| `atomic_sub(i, &v)` | 原子变量 - `i` |
| `atomic_inc(&v)` | 原子变量自增 +1 |
| `atomic_dec(&v)` | 原子变量自减 -1 |
| `atomic_inc_return(&v)` | +1 并返回**新值** |
| `atomic_dec_return(&v)` | -1 并返回**新值** |
| `atomic_dec_and_test(&v)` | -1 后判断结果是否为 0（常用作互斥标记） |

#### 47.2.3 原子位操作

无专用结构体，直接操作**内存地址的指定 bit 位**：

| API | 作用 |
|-----|------|
| `set_bit(nr, p)` | 将 `p` 地址的第 `nr` 位置 1 |
| `clear_bit(nr, p)` | 将 `p` 地址的第 `nr` 位清零 |
| `change_bit(nr, p)` | 翻转第 `nr` 位 |
| `test_bit(nr, p)` | 读取第 `nr` 位的值 |
| `test_and_set_bit(nr, p)` | 置 1，返回原来的值 |
| `test_and_clear_bit(nr, p)` | 清零，返回原来的值 |

---

### 47.3 自旋锁（`spinlock_t`）

#### 47.3.1 自旋锁简介

原子操作仅能保护简单变量；自旋锁用来保护**一段临界代码**（结构体、多变量操作）。

> **类比理解**：自旋锁 = 电话亭，一次只能进一个人；外面的人**原地转圈（自旋）等待**，不睡觉、不让出 CPU。

- **优点**：等待时不睡眠，上下文切换开销极小
- **核心局限**：锁持有期间持续占用 CPU，**临界区必须极短**，长临界区改用信号量/mutex
- **定义**：`spinlock_t lock;`

#### 47.3.2 自旋锁 API

##### 1）基础自旋锁（仅线程间竞争）

> 场景：仅内核线程之间竞争，无中断干扰。

| API | 作用 |
|-----|------|
| `DEFINE_SPINLOCK(lock)` | 定义 + 初始化自旋锁 |
| `spin_lock_init(&lock)` | 初始化已定义的自旋锁 |
| `spin_lock(&lock)` | 获取锁（拿不到则原地自旋） |
| `spin_unlock(&lock)` | 释放锁 |
| `spin_trylock(&lock)` | 尝试拿锁，拿不到则返回 0（不等待） |

> **致命禁忌**：**临界区内绝对不能调用任何会休眠的函数**！
> 
> 若持有者休眠 → 其他线程自旋等待锁，但内核抢占被关闭 → 无法调度唤醒持有者 → **死锁**！

##### 2）线程 + 中断竞争（最常用场景）

> 风险：线程拿到锁后被中断打断，中断也申请同一锁 → 中断自旋等待 → 线程无法运行释放锁 → 死锁。
> 
> **解决方案**：线程加锁前**关闭本地中断**。

| API | 作用 |
|-----|------|
| `spin_lock_irq(&lock)` | 关本地中断 + 获取锁 |
| `spin_unlock_irq(&lock)` | 开本地中断 + 释放锁 |
| `spin_lock_irqsave(&lock, flags)` | **保存当前中断状态 → 关本地中断 → 拿锁（强烈推荐！）** |
| `spin_unlock_irqrestore(&lock, flags)` | 恢复保存的中断状态 + 释放锁 |

> **推荐组合**：线程中用 `spin_lock_irqsave` / `spin_unlock_irqrestore`；中断里用 `spin_lock` / `spin_unlock`（中断本身已屏蔽其他中断，无需再关）。

##### 3）下半部（BH）竞争

> 下半部（Bottom Half，如软中断、tasklet）与线程竞争时使用：

| API | 作用 |
|-----|------|
| `spin_lock_bh(&lock)` | 关闭下半部 + 获取锁 |
| `spin_unlock_bh(&lock)` | 打开下半部 + 释放锁 |

#### 47.3.3 衍生锁

##### 1）读写自旋锁（`rwlock_t`）

> 场景：**读多写少**。多个读者可并发读；写锁独占，读写不能同时。

- 读锁：`read_lock(&rwlock)` / `read_unlock(&rwlock)`
- 写锁：`write_lock(&rwlock)` / `write_unlock(&rwlock)`
- 同样带 `_irqsave` 系列版本（如 `read_lock_irqsave`）。

##### 2）顺序锁（`seqlock_t`）

> 允许**读的时候可以写**，但写不能并发；读完后检测期间是否有写操作，有写则重读。

> **顺序锁限制**：无法保护指针类型数据——写操作可能修改指针值，读操作时访问旧指针会触发**野指针崩溃**。

#### 47.3.4 自旋锁使用四大注意事项

1. 锁持有时间必须**极短**，长时间占用浪费 CPU；长临界区改用信号量/mutex
2. 临界区**绝对禁止调用会休眠的函数**，否则死锁
3. 不能递归申请自旋锁（自己持有锁再次申请 → 原地自旋死锁）
4. 驱动编写统一按 **SMP 多核** 规范写，保证可移植性（即使是单核芯片）

---

### 47.4 信号量（`semaphore`）

#### 47.4.1 信号量简介

> **类比理解**：信号量 = 停车场，`count` 代表可用车位数量。

- 等待拿不到资源时，线程**进入休眠，让出 CPU**，不占用 CPU，利用率高
- **核心限制**：**绝对不能在中断上下文使用**——中断不允许休眠（仅进程上下文可）

##### 信号量分类

| 类型 | 初始 count | 作用 | 场景 |
|------|------------|------|------|
| 计数信号量 | >1 | 允许多线程同时访问 | 多资源同步（如多缓冲区） |
| 二值信号量 | 1 | 同一时间仅一个线程访问 | 互斥（**现在优先推荐 mutex**） |

```c
 struct semaphore {
    raw_spinlock_t lock;
    unsigned int count;
    struct list_head wait_list;
};
```

#### 47.4.2 信号量 API

| API | 作用 |
|-----|------|
| `DEFINE_SEMAPHORE(name)` | 定义二值信号量，`count=1` |
| `sema_init(&sem, val)` | 初始化信号量，设置 `count=val` |
| `down(&sem)` | 获取信号量，拿不到则**不可中断休眠** |
| `down_interruptible(&sem)` | 获取信号量，拿不到则**可被信号打断休眠（常用！）** |
| `down_trylock(&sem)` | 尝试获取，拿不到则返回非 0（不休眠） |
| `up(&sem)` | 释放信号量，`count+1`，唤醒等待线程 |

```c
//示例
struct semaphore sem;
sema_init(&sem, 1);
down(&sem);
/*临界区代码*/
up(&sem);
```

---

### 47.5 互斥体（`mutex`）

#### 47.5.1 mutex 简介

Linux 内核**专为互斥访问设计**，比二值信号量更严格、更专业——驱动互斥场景**优先用 mutex**。

- 一次仅能一个线程持有 mutex
- **使用限制**：
  1. mutex 可休眠 → **中断上下文绝对不能用**，中断只能用自旋锁
  2. 临界区内**可以调用阻塞/休眠函数**（和自旋锁相反）
  3. 必须由**持有者释放**，不支持递归上锁
  4. 释放前不能再对其他资源加锁（防死锁）

#### 47.5.2 mutex API

| API | 作用 |
|-----|------|
| `DEFINE_MUTEX(name)` | 定义并初始化 mutex |
| `mutex_init(&lock)` | 初始化已定义的 mutex |
| `mutex_lock(&lock)` | 获取锁，拿不到则休眠 |
| `mutex_unlock(&lock)` | 释放锁 |
| `mutex_trylock(&lock)` | 尝试获取，不成功则返回 0（不等待） |
| `mutex_lock_interruptible(&lock)` | 获取锁，休眠可被信号打断 |

```c
//示例
struct mutex lock;
mutex_init(&lock);
mutex_lock(&lock);
/*临界区*/
mutex_unlock(&lock);
```

---

## 第四十八章 并发与竞争实验

利用原子变量 `atomic_t` 实现设备互斥访问：同一时刻，只允许一个应用程序打开 LED 设备。

底层基础：45 章 gpioled 驱动，设备树无需修改。

---

### 48.1 驱动代码核心改动点（atomic.c）

#### 1）设备结构体增加原子变量 lock

```c
struct gpioled_dev{
    dev_t devid;
    struct cdev cdev;
    struct class *class;
    struct device *device;
    int major;
    int minor;
    struct device_node *nd;
    int led_gpio;
    atomic_t lock;  //新增原子变量，标记设备占用状态
};
```
- **atomic_t lock**：作为设备的 “占用标记”

#### 2）驱动入口 `led_init` 初始化原子变量

```c
atomic_set(&gpioled.lock, 1); /* 初始值=1：代表设备空闲，可以被打开 */
#### 3）open 函数：申请原子锁
```bash
static int led_open(struct inode *inode, struct file *filp)
{
    if (!atomic_dec_and_test(&gpioled.lock)) {
        atomic_inc(&gpioled.lock); //减1之后lock变成负数，恢复回去
        return -EBUSY;  //返回设备忙，第二个应用打开失败
    }
    filp->private_data = &gpioled;
    return 0;
}
```

`atomic_dec_and_test` 逻辑：先把原子变量 -1，然后判断结果是否等于 0。

1. 初始 `lock=1`，第一次调用：`1-1=0` → 返回 true，`!true` 为假，进入临界区，打开成功
2. 此时 `lock=0`；第二个应用调用：`0-1=-1` → 返回 false，`!false` 为真，执行里面代码
   - `lock` 变成 -1，必须执行 `atomic_inc` 恢复 `lock=0`
   - 返回 `-EBUSY`，应用层 open 会报错，提示设备忙

#### 4）release 函数：释放原子锁，归还设备

```c
static int led_release(struct inode *inode, struct file *filp)
{
    struct gpioled_dev *dev = filp->private_data;
    atomic_inc(&dev->lock); //原子变量+1，恢复lock=1，设备空闲
    return 0;
}
```

应用调用 `close`，触发 `release`，`lock` 恢复为 1，其他应用就可以再次打开 LED。

#### 5）write 函数：LED 控制逻辑

和 45 章 gpioled 一致，`copy_from_user` 从用户空间拿到 LED 状态，调用 `gpio_set_value` 控制 IO 电平。

注意原理图逻辑：`gpio_set_value(led_gpio, 0)` → LED 亮；写 1 → LED 灭。

---

### 48.2 测试 APP atomicApp.c

重点改动：打开 LED 成功之后，循环 sleep 模拟占用设备 25 秒。

```c
/* 模拟占用25S LED */
while(1) {
    sleep(5);
    cnt++;
    printf("App running times:%d\r\n", cnt);
    if(cnt >= 5) break;
}
```

- 逻辑：应用成功 open 设备后，程序不立刻 close，占用 25s
- 在这 25s 内，再开第二个终端运行 atomicApp 去打开 `/dev/gpioled` → open 直接失败，提示 `file open failed`

APP 使用命令：

```bash
# 点亮LED
./atomicApp /dev/gpioled 1
# 熄灭LED
./atomicApp /dev/gpioled 0
```

---

### 48.3 实验测试步骤

1. 编译驱动：Makefile，生成 `atomic.ko`
2. 编译测试 APP：`arm-linux-gnueabihf-gcc atomicApp.c -o atomicApp`
3. 开发板加载驱动：`insmod atomic.ko`，自动生成 `/dev/gpioled` 设备文件
4. 终端 A 执行：`./atomicApp /dev/gpioled 1`，APP 开始占用 LED，打印计数，持续 25s
5. 立刻新开终端 B，同样执行 `./atomicApp/dev/gpioled 1`
   - 现象：终端 B 直接打印 `file /dev/gpioled open failed!`，无法打开设备，验证互斥成功
6. 等待终端 A 程序运行结束，自动 close 释放原子变量；再次在终端 B 运行，可正常打开
