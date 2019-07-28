---
layout: post
title:  "热迁移:multifd"
date:   2019-07-28 20:00:41 +0800
categories: [虚拟化]
tag: [虚拟化,热迁移]
---

<!-- vim-markdown-toc GFM -->

* [multifd](#multifd)
    * [QEMU](#qemu)
    * [初始化](#初始化)
        * [源端初始化](#源端初始化)
        * [目的端](#目的端)
    * [数据传输](#数据传输)
        * [传输路径](#传输路径)
        * [整体框架图](#整体框架图)
* [Q&A](#qa)
    * [1.每次交给每个multifd_send_thread的请求多少？](#1每次交给每个multifd_send_thread的请求多少)
    * [2. multifd的sync机制是什么？用来做什么的？](#2-multifd的sync机制是什么用来做什么的)
    * [3. multifd相对于非multifd的迁移还有什么优势么?](#3-multifd相对于非multifd的迁移还有什么优势么)

<!-- vim-markdown-toc -->

# multifd
> QEMU 4.0.0源码

## QEMU

迁移时的进程：源端
```
 89911 root      20   0  0.126t 0.120t  12856 R 99.9 32.8  59:28.60 CPU 0/KVM
 89912 root      20   0  0.126t 0.120t  12856 R 99.9 32.8  59:20.67 CPU 1/KVM
 89914 root      20   0  0.126t 0.120t  12856 R 99.9 32.8  59:20.71 CPU 3/KVM
 89913 root      20   0  0.126t 0.120t  12856 R 99.7 32.8  59:20.39 CPU 2/KVM
112511 root      20   0  0.126t 0.120t  12856 R 97.7 32.8   0:09.99 live_migration
112540 root      20   0  0.126t 0.120t  12856 S 26.7 32.8   0:02.48 multifdsend_0
112541 root      20   0  0.126t 0.120t  12856 S 26.3 32.8   0:02.49 multifdsend_3
112543 root      20   0  0.126t 0.120t  12856 R 23.7 32.8   0:02.39 multifdsend_1
112542 root      20   0  0.126t 0.120t  12856 S 23.0 32.8   0:02.42 multifdsend_2
 89905 root      20   0  0.126t 0.120t  12856 S  0.0 32.8   0:01.60 qemu-system-x86
 89906 root      20   0  0.126t 0.120t  12856 S  0.0 32.8   0:00.00 qemu-system-x86
 89907 root      20   0  0.126t 0.120t  12856 S  0.0 32.8   0:00.03 worker
 89910 root      20   0  0.126t 0.120t  12856 S  0.0 32.8   0:00.00 IO mon_iothread
 89916 root      20   0  0.126t 0.120t  12856 S  0.0 32.8   0:00.00 vnc_worker
```

迁移时的进程：目的端
```
 34576 root      20   0  0.126t 0.120t  12604 S 44.7 32.7   0:35.46 multifdrecv_1
 34574 root      20   0  0.126t 0.120t  12604 R 43.3 32.7   0:34.05 multifdrecv_3
 34573 root      20   0  0.126t 0.120t  12604 S 41.7 32.7   0:35.94 multifdrecv_0
 34575 root      20   0  0.126t 0.120t  12604 S 36.0 32.7   0:32.31 multifdrecv_2
 34560 root      20   0  0.126t 0.120t  12604 S  0.0 32.7   0:00.82 qemu-system-x86
 34561 root      20   0  0.126t 0.120t  12604 S  0.0 32.7   0:00.00 qemu-system-x86
 34565 root      20   0  0.126t 0.120t  12604 S  0.0 32.7   0:00.00 IO mon_iothread
 34566 root      20   0  0.126t 0.120t  12604 S  0.0 32.7   0:00.00 CPU 0/KVM
 34567 root      20   0  0.126t 0.120t  12604 S  0.0 32.7   0:00.00 CPU 1/KVM
 34568 root      20   0  0.126t 0.120t  12604 S  0.0 32.7   0:00.00 CPU 2/KVM
 34569 root      20   0  0.126t 0.120t  12604 S  0.0 32.7   0:00.00 CPU 3/KVM
 34571 root      20   0  0.126t 0.120t  12604 S  0.0 32.7   0:00.00 vnc_worker
```

## 初始化
### 源端初始化
```
#define MULTIFD_PACKET_SIZE (512 * 1024)

qmp_migrate
    -->fd_start_outgoing_migration
        -->migration_channel_connect
            -->migrate_fd_connect
                -->multifd_save_setup
                    -->[for each thread]socket_send_channel_create[multifd_new_send_channel_async];
                        -->[new thread]multifd_new_send_channel_async
                            -->[new thread]qemu_thread_create["multifdsend_%i",multifd_send_thread]
                -->qemu_thread_create["live_migration", migration_thread]
```


```
multifd_send_thread
    -->multifd_send_initial_packet
    -->while 搬运工
```


### 目的端

```
migration_ioc_process_incoming
```

```
socket_accept_incoming_migration[每个连接都会调用一次]
    -->migration_ioc_process_incoming
        -->[if (!mis->from_src_file)]migration_incoming_setup[f]
            -->multifd_load_setup: 第一个连接建立，不包括multifd的连接
            -->mis->from_src_file = f
        -->[初始化multifd recv链接]multifd_recv_new_channel
            --> qemu_thread_create("multifdrecv_%i", multifd_recv_thread]
        -->[multifd所有fd初始化成功后]migration_incoming_process
```

```
multifd_recv_thread:接收端搬运工
```

## 数据传输

### 传输路径

```
ram_save_iterate
    -->ram_find_and_save_block
        -->ram_save_host_page
            -->ram_save_target_page
                -->ram_save_multifd_page
                    -->multifd_queue_page
                        -->multifd_send_pages
    -->multifd_send_sync_main
        -->multifd_send_pages
```

### 整体框架图

```
        multifd send thread              Migration thread                       multifd send thread

                                                +-------------------+
                                                |                   |
                                                v                   |
                                      +-------------------+         |
                                      |Get Dirty Page Info|         |
    +---------+                       +---------+---------+         |                  +-------------+
    |         |                                 |                   |                  |             |
    |         v                                 v                   |                  v             |
    |  +----------------+   [pages] +-------------------------+     |  [pages] +----------------+    |
    |  | wait to start  | <---------+Notify compression thread+-----+--------> | wait to start  |    |
    |  +-------+--------+           |  to start compress      |     |          +-------+--------+    |
    |          |                    +-----------+-------------+     |                  |             |
    |          v                                |                   |                  v             |
    |  +----------------+                       v                   |          +----------------+    |
    |  | send out data  |           +-----------+-------------+     |          | send out data  |    |
    |  +-------+--------+     +---> |Get idle send thread, and| <---+-----+    +-------+--------+    |
    |          |              |     |wait if all busy         |     |     |            |             |
    |          v              |     +-----------+-------------+     |     |            v             |
    |  +-----------------+    |                 |                   |     |    +-----------------+   |
    |  | Notify migration|    |                 v                   |     |    | Notify migration|   |
    |  | thread          +----+     +-------------------------+     |     +----+ thread          |   |
    |  +-------+---------+          |     Sync with dest      |     |          +-------+---------+   |
    |          |                    |   (RAM_SAVE_FLAG_EOS)   |     |                  |             |
    +----------+                    +-----------+-------------+     |                  +-------------+
                                                |                   |
                                                |                   |
                                                +-------------------+
```

# Q&A

## 1.每次交给每个multifd_send_thread的请求多少？

每次最多可以缓存512*1024/4K = 128个连续的dirty页面

```
#define MULTIFD_PACKET_SIZE (512 * 1024)
uint32_t page_count = MULTIFD_PACKET_SIZE / qemu_target_page_size();
```

(1) ram_save_target_page会不断缓存内存页，缓存条件：
- 同一个RAMBlock
- 连续脏页

```
static void multifd_queue_page(RAMBlock *block, ram_addr_t offset)
{
    MultiFDPages_t *pages = multifd_send_state->pages;

    if (!pages->block) {
        pages->block = block;
    }

    if (pages->block == block) {
        pages->offset[pages->used] = offset;
        pages->iov[pages->used].iov_base = block->host + offset;
        pages->iov[pages->used].iov_len = TARGET_PAGE_SIZE;
        pages->used++;

        if (pages->used < pages->allocated) {          //=====>会在这里缓存，达到就交给multifd_send_thread
            return;
        }
    }

    multifd_send_pages();

    if (pages->block != block) {
        multifd_queue_page(block, offset);
    }
}
```

(2)最后会在multifd_send_sync_main中刷一次

```
ram_save_iterate
    -->multifd_send_sync_main
        -->multifd_send_pages
```

## 2. multifd的sync机制是什么？用来做什么的？

multifd机制中，由主线程分发脏页交由multifd_send_thread发送数据，为避免后面传输的内存与
之前的内存传输混乱，每次迭代都需要与目的端同步

sync原理图如下：

```

     src migration thread                                                                          dst main thread

           |                                                                                             |
           |                                                                                             |
           |                                                                                             |
      +----+------+                                                                                      |
      | send sync |---+                                                                                  |
      +----+------+   |                                                       +--------------------------+-------------+
           |          | 1. begin sync                                         |                          |             |
           |          |                                                       v                          |             |
           |          |         +-------------+  MULTIFD_FLAG_SYNC    +-------------+                    |             |
           |          +-------->| send thread |---------------------->| recv thread |-------+            |             |
           |          |         +-----+-------+                       +-------------+       |            |             |
           |          |               |                                       +-------------+------------+-------------+
           |          |    +----------+      2. send MULTIFD_FLAG_SYNC to dst |             |            |             |
           |          |    |                                                  v             |            |             |
           |          |    |    +-------------+  MULTIFD_FLAG_SYNC    +-------------+       |            |             |
           |          +----+--->| send thread |---------------------->| recv thread |-------+            |             |
           |               |    +-----+-------+                       +-------------+       |            |             |
           |               |          |                                                     |            |             |
           |               |          |                                                     |            |             |
      +----+------+        |          |                                                     |            |             |
      |   wait    |<-------+----------+                                                     |            |             |
      +----+------+     3. notify migration thread                           5. recv and notify          |             |
           |                                                                                |            |             |
           |                                                                                |            |             |
      +-----------+                          4.RAM_SAVE_FLAG_EOS                            |       +-----------+      |
      | send EOS  |  -----------------------------------------------------------------------+---->  | recv EOS  |      |
      +----+------+                                                                         |       +----+------+      |
           |                                                                                |            |             |
           |                                                                                |            |             |
           |                                                                                |       +-----------+      |
           |                                                                                +------>|   wait    |      |
           |                                                                                        +----+------+      |
           |                                                                                             |             |
           |                                                                                             +-------------+
           |                                                                                             |  6. notify
           |                                                                                             |
           |                                                                                             |
```

## 3. multifd相对于非multifd的迁移还有什么优势么?

**普通迁移：**
1.普通迁移，每页都需要带上一个header信息，且只有一条流：如下

```
[page header 1][4k page 1][page header 2][4k page 2]
```
2.普通迁移的每页内存都需要一次拷贝

**mltifd迁移：**
1.多路并发批量迁移
```
[page header 1][4k page 1-1][4k page 1-2]...[4k page 1-x]
[page header 2][4k page 2-1][4k page 2-2]...[4k page 1-y]
...
[page header n][4k page n-1][4k page n-2]...[4k page n-z]

```
2.内存不需要拷贝一次，通过sendmsg + iovcs直接传输
