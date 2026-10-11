# 第六十八章 Linux 块设备驱动实验

前面我们都是在学习字符设备驱动，本章我们来学习一下块设备驱动框架，块设备驱动是 Linux 三大驱动类型之一。块设备驱动要远比字符设备驱动复杂得多，不同类型的存储设备又对应不同的驱动子系统，本章我们重点学习一下块设备相关驱动概念，不涉及到具体的存储设备。最后，我们使用 ALPHA 开发板板载 RAM 模拟一个块设备，学习块设备驱动框架的使用。

## 68.1 什么是块设备？

块设备是针对存储设备的，比如 SD 卡、EMMC、NAND Flash、Nor Flash、SPI Flash、机械硬盘、固态硬盘等。因此块设备驱动其实就是这些存储设备驱动，块设备驱动相比字符设备驱动的主要区别如下：

①、块设备只能以块为单位进行读写访问，块是 linux 虚拟文件系统(VFS)基本的数据传输单位。字符设备是以字节为单位进行数据传输的，不需要缓冲。
②、块设备在结构上是可以进行随机访问的，对于这些设备的读写都是按块进行的，块设备使用缓冲区来暂时存放数据，等到条件成熟以后再一次性将缓冲区中的数据写入块设备中。这么做的目的为了提高块设备寿命，大家如果仔细观察的话就会发现有些硬盘或者 NAND Flash 就会标明擦除次数(flash 的特性，写之前要先擦除)，比如擦除 100000 次等。因此，为了提高块设备寿命引入了缓冲区，数据先写入到缓冲区中，等满足一定条件后再一次性写入到真正的物理存储设备中，这样就减少了对块设备的擦除次数，提高了块设备寿命。

字符设备是顺序的数据流设备，字符设备是按照字节进行读写访问的。字符设备不需要缓冲区，对于字符设备的访问都是实时的，而且也不需要按照固定的块大小进行访问。

块设备结构的不同其 I/O 算法也会不同，比如对于 EMMC、SD 卡、NAND Flash 这类没有任何机械设备的存储设备就可以任意读写任何的扇区(块设备物理存储单元)。但是对于机械硬盘这样带有磁头的设备，读取不同的盘面或者磁道里面的数据，磁头都需要进行移动，因此对于机械硬盘而言，将那些杂乱的访问按照一定的顺序进行排列可以有效提高磁盘性能，linux 里面针对不同的存储设备实现了不同的 I/O 调度算法。

## 68.2 块设备驱动框架

### 68.2.1 block_device 结构体

linux 内核使用 `block_device` 表示块设备，`block_device` 为一个结构体，定义在 `include/linux/fs.h` 文件中，结构体内容如下：

**示例代码 68.2.1.1 block_device 结构体**

```c
1 struct block_device {
2     dev_t bd_dev; /* not a kdev_t - it's a search key */
3     int bd_openers;
4     struct inode *bd_inode; /* will die */
5     struct super_block *bd_super;
6     struct mutex bd_mutex; /* open/close mutex */
7     struct list_head bd_inodes;
8     void * bd_claiming;
9     void * bd_holder;
10    int bd_holders;
11    bool bd_write_holder;
12 #ifdef CONFIG_SYSFS
13    struct list_head bd_holder_disks;
14 #endif
15    struct block_device *bd_contains;
16    unsigned bd_block_size;
17    struct hd_struct *bd_part;
18    /*number of times partitions within this device have been opened.*/
19    unsigned bd_part_count;
20    int bd_invalidated;
21    struct gendisk *bd_disk;
22    struct request_queue *bd_queue;
23    struct list_head bd_list;
24    /*
25     * Private data. You must have bd_claim'ed the block_device
26     * to use this. NOTE: bd_claim allows an owner to claim
27     * the same device multiple times, the owner must take special
28     * care to not mess up bd_private for that case.
29     */
30    unsigned long bd_private;
31
32    /* The counter of freeze processes */
33    int bd_fsfreeze_count;
34    /* Mutex for freeze */
35    struct mutex bd_fsfreeze_mutex;
36 };
```

对于 `block_device` 结构体，我们重点关注一下第 21 行的 `bd_disk` 成员变量，此成员变量为 `gendisk` 结构体指针类型。内核使用 `block_device` 来表示一个具体的块设备对象，比如一个硬盘或者分区，如果是硬盘的话 `bd_disk` 就指向通用磁盘结构 `gendisk`。

**1、注册块设备**

和字符设备驱动一样，我们需要向内核注册新的块设备、申请设备号，块设备注册函数为 `register_blkdev`，函数原型如下：

```c
int register_blkdev(unsigned int major, const char *name)
```

函数参数和返回值含义如下：

- `major`：主设备号。
- `name`：块设备名字。

返回值：如果参数 `major` 在 1~255 之间的话表示自定义主设备号，那么返回 0 表示注册成功，如果返回负值的话表示注册失败。如果 `major` 为 0 的话表示由系统自动分配主设备号，那么返回值就是系统分配的主设备号(1~255)，如果返回负值那就表示注册失败。

**2、注销块设备**

和字符设备驱动一样，如果不使用某个块设备了，那么就需要注销掉，函数为 `unregister_blkdev`，函数原型如下：

```c
void unregister_blkdev(unsigned int major, const char *name)
```

函数参数和返回值含义如下：

- `major`：要注销的块设备主设备号。
- `name`：要注销的块设备名字。

返回值：无。

### 68.2.2 gendisk 结构体

linux 内核使用 `gendisk` 来描述一个磁盘设备，这是一个结构体，定义在 `include/linux/genhd.h` 中，内容如下：

**示例代码 68.2.2.1 gendisk 结构体**

```c
1 struct gendisk {
2     /* major, first_minor and minors are input parameters only,
3      * don't use directly. Use disk_devt() and disk_max_parts().
4      */
5     int major; /* major number of driver */
6     int first_minor;
7     int minors; /* maximum number of minors, =1 for
8                  * disks that can't be partitioned. */
9
10    char disk_name[DISK_NAME_LEN]; /* name of major driver */
11    char *(*devnode)(struct gendisk *gd, umode_t *mode);
12
13    unsigned int events; /* supported events */
14    unsigned int async_events; /* async events, subset of all */
15
16    /* Array of pointers to partitions indexed by partno.
17     * Protected with matching bdev lock but stat and other
18     * non-critical accesses use RCU. Always access through
19     * helpers.
20     */
21    struct disk_part_tbl __rcu *part_tbl;
22    struct hd_struct part0;
23
24    const struct block_device_operations *fops;
25    struct request_queue *queue;
26    void *private_data;
27
28    int flags;
29    struct device *driverfs_dev; // FIXME: remove
30    struct kobject *slave_dir;
31
32    struct timer_rand_state *random;
33    atomic_t sync_io; /* RAID */
34    struct disk_events *ev;
35 #ifdef CONFIG_BLK_DEV_INTEGRITY
36    struct blk_integrity *integrity;
37 #endif
38    int node_id;
39 };
```

我们简单看一下 `gendisk` 结构体中比较重要的几个成员变量：

- 第 5 行，`major` 为磁盘设备的主设备号。
- 第 6 行，`first_minor` 为磁盘的第一个次设备号。
- 第 7 行，`minors` 为磁盘的次设备号数量，也就是磁盘的分区数量，这些分区的主设备号一样，次设备号不同。
- 第 21 行，`part_tbl` 为磁盘对应的分区表，为结构体 `disk_part_tbl` 类型，`disk_part_tbl` 的核心是一个 `hd_struct` 结构体指针数组，此数组每一项都对应一个分区信息。
- 第 24 行，`fops` 为块设备操作集，为 `block_device_operations` 结构体类型。和字符设备操作集 `file_operations` 一样，是块设备驱动中的重点！
- 第 25 行，`queue` 为磁盘对应的请求队列，所以针对该磁盘设备的请求都放到此队列中，驱动程序需要处理此队列中的所有请求。

编写块设备驱动的时候需要分配并初始化一个 `gendisk`，linux 内核提供了一组 `gendisk` 操作函数，我们来看一下一些常用的 API 函数。

**1、申请 gendisk**

使用 `gendisk` 之前要先申请，`alloc_disk` 函数用于申请一个 `gendisk`，函数原型如下：

```c
struct gendisk *alloc_disk(int minors)
```

函数参数和返回值含义如下：

- `minors`：次设备号数量，也就是 `gendisk` 对应的分区数量。

返回值：成功：返回申请到的 `gendisk`，失败：`NULL`。

**2、删除 gendisk**

如果要删除 `gendisk` 的话可以使用函数 `del_gendisk`，函数原型如下：

```c
void del_gendisk(struct gendisk *gp)
```

函数参数和返回值含义如下：

- `gp`：要删除的 `gendisk`。

返回值：无。

**3、将 gendisk 添加到内核**

使用 `alloc_disk` 申请到 `gendisk` 以后系统还不能使用，必须使用 `add_disk` 函数将申请到的 `gendisk` 添加到内核中，`add_disk` 函数原型如下：

```c
void add_disk(struct gendisk *disk)
```

函数参数和返回值含义如下：

- `disk`：要添加到内核的 `gendisk`。

返回值：无。

**4、设置 gendisk 容量**

每一个磁盘都有容量，所以在初始化 `gendisk` 的时候也需要设置其容量，使用函数 `set_capacity`，函数原型如下：

```c
void set_capacity(struct gendisk *disk, sector_t size)
```

函数参数和返回值含义如下：

- `disk`：要设置容量的 `gendisk`。
- `size`：磁盘容量大小，注意这里是扇区数量。块设备中最小的可寻址单元是扇区，一个扇区一般是 512 字节，有些设备的物理扇区可能不是 512 字节。不管物理扇区是多少，内核和块设备驱动之间的扇区都是 512 字节。所以 `set_capacity` 函数设置的大小就是块设备实际容量除以 512 字节得到的扇区数量。比如一个 2MB 的磁盘，其扇区数量就是 `(2*1024*1024)/512=4096`。

返回值：无。

**5、调整 gendisk 引用计数**

内核会通过 `get_disk` 和 `put_disk` 这两个函数来调整 `gendisk` 的引用计数，根据名字就可以知道，`get_disk` 是增加 `gendisk` 的引用计数，`put_disk` 是减少 `gendisk` 的引用计数，这两个函数原型如下所示：

```c
struct kobject *get_disk(struct gendisk *disk)
void put_disk(struct gendisk *disk)
```

### 68.2.3 block_device_operations 结构体

和字符设备的 `file_operations` 一样，块设备也有操作集，为结构体 `block_device_operations`，此结构体定义在 `include/linux/blkdev.h` 中，结构体内容如下：

**示例代码 68.2.3.1 block_device_operations 结构体**

```c
1 struct block_device_operations {
2     int (*open) (struct block_device *, fmode_t);
3     void (*release) (struct gendisk *, fmode_t);
4     int (*rw_page)(struct block_device *, sector_t, struct page *, int rw);
5     int (*ioctl) (struct block_device *, fmode_t, unsigned, unsigned long);
6     int (*compat_ioctl) (struct block_device *, fmode_t, unsigned, unsigned long);
7     long (*direct_access)(struct block_device *, sector_t,
8             void **, unsigned long *pfn, long size);
9     unsigned int (*check_events) (struct gendisk *disk,
10            unsigned int clearing);
11    /* ->media_changed() is DEPRECATED, use ->check_events() instead */
12    int (*media_changed) (struct gendisk *);
13    void (*unlock_native_capacity) (struct gendisk *);
14    int (*revalidate_disk) (struct gendisk *);
15    int (*getgeo)(struct block_device *, struct hd_geometry *);
16    /* this callback is with swap_lock and sometimes page table lock held */
17    void (*swap_slot_free_notify) (struct block_device *, unsigned long);
18    struct module *owner;
19 };
```

可以看出，`block_device_operations` 结构体里面的操作集函数和字符设备的 `file_operations` 操作集基本类似，但是块设备的操作集函数比较少，我们来看一下其中比较重要的几个成员函数：

- 第 2 行，`open` 函数用于打开指定的块设备。
- 第 3 行，`release` 函数用于关闭(释放)指定的块设备。
- 第 4 行，`rw_page` 函数用于读写指定的页。
- 第 5 行，`ioctl` 函数用于块设备的 I/O 控制。
- 第 6 行，`compat_ioctl` 函数和 `ioctl` 函数一样，都是用于块设备的 I/O 控制。区别在于在 64 位系统上，32 位应用程序的 `ioctl` 会调用 `compat_ioctl` 函数。在 32 位系统上运行的 32 位应用程序调用的就是 `ioctl` 函数。
- 第 15 行，`getgeo` 函数用于获取磁盘信息，包括磁头、柱面和扇区等信息。
- 第 18 行，`owner` 表示此结构体属于哪个模块，一般直接设置为 `THIS_MODULE`。

### 68.2.4 块设备 I/O 请求过程

大家如果仔细观察的话会在 `block_device_operations` 结构体中并没有找到 `read` 和 `write` 这样的读写函数，那么块设备是怎么从物理块设备中读写数据？这里就引出了块设备驱动中非常重要的 `request_queue`、`request` 和 `bio`。

**1、请求队列 request_queue**

内核将对块设备的读写都发送到请求队列 `request_queue` 中，`request_queue` 中是大量的 `request`(请求结构体)，而 `request` 又包含了 `bio`，`bio` 保存了读写相关数据，比如从块设备的哪个地址开始读取、读取的数据长度，读取到哪里，如果是写的话还包括要写入的数据等。我们先来看一下 `request_queue`，这是一个结构体，定义在文件 `include/linux/blkdev.h` 中，由于 `request_queue` 结构体比较长，这里就不列出来了。大家回过头看一下示例代码 68.2.2.1 的 `gendisk` 结构体就会发现里面有一个 `request_queue` 结构体指针类型成员变量 `queue`，也就说在编写块设备驱动的时候，每个磁盘(`gendisk`)都要分配一个 `request_queue`。

**①、初始化请求队列**

我们首先需要申请并初始化一个 `request_queue`，然后在初始化 `gendisk` 的时候将这个 `request_queue` 地址赋值给 `gendisk` 的 `queue` 成员变量。使用 `blk_init_queue` 函数来完成 `request_queue` 的申请与初始化，函数原型如下：

```c
request_queue *blk_init_queue(request_fn_proc *rfn, spinlock_t *lock)
```

函数参数和返回值含义如下：

- `rfn`：请求处理函数指针，每个 `request_queue` 都要有一个请求处理函数，请求处理函数 `request_fn_proc` 原型如下：

```c
void (request_fn_proc) (struct request_queue *q)
```

请求处理函数需要驱动编写人员自行实现。

- `lock`：自旋锁指针，需要驱动编写人员定义一个自旋锁，然后传递进来，请求队列会使用这个自旋锁。

返回值：如果为 `NULL` 的话表示失败，成功的话就返回申请到的 `request_queue` 地址。

**②、删除请求队列**

当卸载块设备驱动的时候我们还需要删除掉前面申请到的 `request_queue`，删除请求队列使用函数 `blk_cleanup_queue`，函数原型如下：

```c
void blk_cleanup_queue(struct request_queue *q)
```

函数参数和返回值含义如下：

- `q`：需要删除的请求队列。

返回值：无。

**③、分配请求队列并绑定制造请求函数**

`blk_init_queue` 函数完成了请求队列的申请以及请求处理函数的绑定，这个一般用于像机械硬盘这样的存储设备，需要 I/O 调度器来优化数据读写过程。但是对于 EMMC、SD 卡这样的非机械设备，可以进行完全随机访问，所以就不需要复杂的 I/O 调度器了。对于非机械设备我们可以先申请 `request_queue`，然后将申请到的 `request_queue` 与"制造请求"函数绑定在一起。

先来看一下 `request_queue` 申请函数 `blk_alloc_queue`，函数原型如下：

```c
struct request_queue *blk_alloc_queue(gfp_t gfp_mask)
```

函数参数和返回值含义如下：

- `gfp_mask`：内存分配掩码，具体可选择的掩码值请参考 `include/linux/gfp.h` 中的相关宏定义，一般为 `GFP_KERNEL`。

返回值：申请到的无 I/O 调度的 `request_queue`。

我们需要为 `blk_alloc_queue` 函数申请到的请求队列绑定一个"制造请求"函数。这里我们需要用到函数 `blk_queue_make_request`，函数原型如下：

```c
void blk_queue_make_request(struct request_queue *q, make_request_fn *mfn)
```

函数参数和返回值含义如下：

- `q`：需要绑定的请求队列，也就是 `blk_alloc_queue` 申请到的请求队列。
- `mfn`：需要绑定的"制造"请求函数，函数原型如下：

```c
void (make_request_fn) (struct request_queue *q, struct bio *bio)
```

"制造请求"函数需要驱动编写人员实现。

返回值：无。

一般 `blk_alloc_queue` 和 `blk_queue_make_request` 是搭配在一起使用的，用于那些非机械的存储设备、无需 I/O 调度器，比如 EMMC、SD 卡等。`blk_init_queue` 函数会给请求队列分配一个 I/O 调度器，用于机械存储设备，比如机械硬盘等。

**2、请求 request**

请求队列(`request_queue`)里面包含的就是一系列的请求(`request`)，`request` 是一个结构体，定义在 `include/linux/blkdev.h` 里面，这里就不展开 `request` 结构体了，太长了。`request` 里面有一个名为"bio"的成员变量，类型为 `bio` 结构体指针。前面说了，真正的数据就保存在 `bio` 里面，所以我们需要从 `request_queue` 中取出一个一个的 `request`，然后再从每个 `request` 里面取出 `bio`，最后根据 `bio` 的描述将数据写入到块设备，或者从块设备中读取数据。

**①、获取请求**

我们需要从 `request_queue` 中依次获取每个 `request`，使用 `blk_peek_request` 函数完成此操作，函数原型如下：

```c
request *blk_peek_request(struct request_queue *q)
```

函数参数和返回值含义如下：

- `q`：指定 `request_queue`。

返回值：`request_queue` 中下一个要处理的请求(`request`)，如果没有要处理的请求就返回 `NULL`。

**②、开启请求**

使用 `blk_peek_request` 函数获取到下一个要处理的请求以后就要开始处理这个请求，这里要用到 `blk_start_request` 函数，函数原型如下：

```c
void blk_start_request(struct request *req)
```

函数参数和返回值含义如下：

- `req`：要开始处理的请求。

返回值：无。

**③、一步到位处理请求**

我们也可以使用 `blk_fetch_request` 函数来一次性完成请求的获取和开启，`blk_fetch_request` 函数很简单，内容如下：

**示例代码 68.2.4.1 blk_fetch_request 函数源码**

```c
1 struct request *blk_fetch_request(struct request_queue *q)
2 {
3     struct request *rq;
4
5     rq = blk_peek_request(q);
6     if (rq)
7         blk_start_request(rq);
8     return rq;
9 }
```

可以看出，`blk_fetch_request` 就是直接调用了 `blk_peek_request` 和 `blk_start_request` 这两个函数。

**④、其他和请求有关的函数**

关于请求的 API 还有很多，常见的见表 68.2.4.1：

**表 68.2.4.1 请求相关 API 函数**

| 函数 | 描述 |
|---|---|
| `blk_end_request()` | 请求中指定字节数据被处理完成。 |
| `blk_end_request_all()` | 请求中所有数据全部处理完成。 |
| `blk_end_request_cur()` | 当前请求中的 chunk。 |
| `blk_end_request_err()` | 处理完请求，直到下一个错误产生。 |
| `__blk_end_request()` | 和 `blk_end_request` 函数一样，但是需要持有队列锁。 |
| `__blk_end_request_all()` | 和 `blk_end_request_all` 函数一样，但是需要持有队列锁。 |
| `__blk_end_request_cur()` | 和 `blk_end_request_cur` 函数一样，但是需要持有队列锁。 |
| `__blk_end_request_err()` | 和 `blk_end_request_err` 函数一样，但是需要持有队列锁。 |

**3、bio 结构**

每个 `request` 里面会有多个 `bio`，`bio` 保存着最终要读写的数据、地址等信息。上层应用程序对于块设备的读写会被构造成一个或多个 `bio` 结构，`bio` 结构描述了要读写的起始扇区、要读写的扇区数量、是读取还是写入、页偏移、数据长度等等信息。上层会将 `bio` 提交给 I/O 调度器，I/O 调度器会将这些 `bio` 构造成 `request` 结构，而一个物理存储设备对应一个 `request_queue`，`request_queue` 里面顺序存放着一系列的 `request`。新产生的 `bio` 可能被合并到 `request_queue` 里现有的 `request` 中，也可能产生新的 `request`，然后插入到 `request_queue` 中合适的位置，这一切都是由 I/O 调度器来完成的。`request_queue`、`request` 和 `bio` 之间的关系如图 1-1 所示。

![图1-1 request_queue、request 和 bio 之间的关系](photo/Linux%20块设备驱动实验/1-1%20request_queue、request%20和%20bio%20之间的关系.png)

`bio` 是个结构体，定义在 `include/linux/blk_types.h` 中，结构体内容如下：

**示例代码 68.2.4.2 bio 结构体**

```c
1 struct bio {
2     struct bio          *bi_next;    /* 请求队列的下一个 bio */
3     struct block_device *bi_bdev;    /* 指向块设备 */
4     unsigned long        bi_flags;   /* bio 状态等信息 */
5     unsigned long        bi_rw;      /* I/O 操作,读或写 */
6     struct bvec_iter     bi_iter;    /* I/O 迭代器：扇区、长度、索引 */
7     unsigned int         bi_phys_segments;
8     unsigned int         bi_seg_front_size;
9     unsigned int         bi_seg_back_size;
10    atomic_t             bi_remaining;
11    bio_end_io_t         *bi_end_io;
12    void                 *bi_private;
13 #ifdef CONFIG_BLK_CGROUP
14 /*
15  * Optional ioc and css associated with this bio. Put on bio
16  * release. Read comment on top of bio_associate_current().
17  */
18    struct io_context    *bi_ioc;
19    struct cgroup_subsys_state *bi_css;
20 #endif
21    union {
22 #if defined(CONFIG_BLK_DEV_INTEGRITY)
23        struct bio_integrity_payload *bi_integrity;
24 #endif
25    };
26
27    unsigned short       bi_vcnt;    /* bio_vec 列表中元素数量 */
28    unsigned short       bi_max_vecs;/* bio_vec 列表长度 */
29    atomic_t             bi_cnt;     /* pin count */
30    struct bio_vec       *bi_io_vec; /* bio_vec 列表 */
31    struct bio_set       *bi_pool;
32    struct bio_vec       bi_inline_vecs[0];
33 };
```

重点来看一下第 6 行和第 30 行，第 6 行为 `bvec_iter` 结构体类型的成员变量，第 30 行为 `bio_vec` 结构体指针类型的成员变量。

`bvec_iter` 结构体描述了要操作的设备扇区等信息，结构体内容如下：

**示例代码 68.2.4.3 bvec_iter 结构体**

```c
1 struct bvec_iter {
2     sector_t        bi_sector;     /* I/O 请求的设备起始扇区(512 字节) */
3     unsigned int    bi_size;       /* 剩余的 I/O 数量 */
4     unsigned int    bi_idx;        /* bio_vec 中当前索引 */
5     unsigned int    bi_bvec_done;  /* 当前 bvec 中已经处理完成的字节数 */
6 };
```

`bio_vec` 结构体描述内存页信息，内容如下：

**示例代码 68.2.4.4 bio_vec 结构体**

```c
1 struct bio_vec {
2     struct page     *bv_page;      /* 页 */
3     unsigned int     bv_len;       /* 长度 */
4     unsigned int     bv_offset;   /* 偏移 */
5 };
```

可以看出 `bio_vec` 就是"page, offset, len"组合，`page` 指定了所在的物理页，`offset` 表示所处页的偏移地址，`len` 就是数据长度。

我们对于物理存储设备的操作不外乎就是将 RAM 中的数据写入到物理存储设备中，或者将物理设备中的数据读取到 RAM 中去处理。数据传输三个要求：数据源、数据长度以及数据目的地，也就是你要从物理存储设备的哪个地址开始读取、读取到 RAM 中的哪个地址处、读取的数据长度是多少。既然 `bio` 是块设备最小的数据传输单元，那么 `bio` 就有必要描述清楚这些信息，其中 `bi_iter` 这个结构体成员变量就用于描述物理存储设备地址信息，比如要操作的扇区地址。`bi_io_vec` 指向 `bio_vec` 数组首地址，`bio_vec` 数组就是 RAM 信息，比如页地址、页偏移以及长度，"页地址"是 linux 内核里面内存管理相关的概念，这里我们不深究 linux 内存管理，我们只需要知道对于 RAM 的操作最终会转换为页相关操作。

`bio`、`bvec_iter` 以及 `bio_vec` 这三个结构体之间的关系如图 1-2 所示。

![图1-2 bio、bio_iter 与 bio_vec 之间的关系](photo/Linux%20块设备驱动实验/1-2%20bio、bio_iter%20与%20bio_vec%20之间的关系.png)

**①、遍历请求中的 bio**

前面说了，请求中包含有大量的 `bio`，因此就涉及到遍历请求中所有 `bio` 并进行处理。遍历请求中的 `bio` 使用宏 `__rq_for_each_bio`，内容如下：

**示例代码 68.2.4.5 __rq_for_each_bio 宏**

```c
#define __rq_for_each_bio(_bio, rq) \
if ((rq->bio)) \
    for (_bio = (rq)->bio; _bio; _bio = _bio->bi_next)
```

`_bio` 就是遍历出来的每个 `bio`，`rq` 是要进行遍历操作的请求；`_bio` 参数为 `bio` 结构体指针类型，`rq` 参数为 `request` 结构体指针类型。

**②、遍历 bio 中的所有段**

`bio` 包含了最终要操作的数据，因此还需要遍历 `bio` 中的所有段，这里要用到 `bio_for_each_segment`，此也是一个宏，内容如下：

**示例代码 68.2.4.6 bio_for_each_segment 宏**

```c
#define bio_for_each_segment(bvl, bio, iter) \
    __bio_for_each_segment(bvl, bio, iter, (bio)->bi_iter)
```

第一个 `bvl` 参数就是遍历出来的每个 `bio_vec`，第二个 `bio` 参数就是要遍历的 `bio`，类型为 `bio` 结构体指针，第三个 `iter` 参数保存要遍历的 `bio` 中 `bi_iter` 成员变量。

**③、通知 bio 处理结束**

如果使用"制造请求"，也就是抛开 I/O 调度器直接处理 `bio` 的话，在 `bio` 处理完成以后要通知内核 `bio` 处理完成，使用 `bio_endio` 函数，函数原型如下：

```c
void bio_endio(struct bio *bio, int error);
```

函数参数和返回值含义如下：

- `bio`：要结束的 `bio`。
- `error`：如果 `bio` 处理成功填 0，如果失败填负值，例如 `-EIO`。

返回值：无。
