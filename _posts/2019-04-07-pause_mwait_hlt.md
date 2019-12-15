---
layout: post
title:  "pause,hlt,mwait/monitor"
date:   2019-04-07 17:16:41 +0800
categories: [内核]
tag: [intel手册,内核,虚拟化]
---

<!-- vim-markdown-toc GFM -->

* [pause, hlt, monitor/mwait](#pause-hlt-monitormwait)
    * [pause && PAUSE-loop exiting(PLE)](#pause--pause-loop-exitingple)
        * [PAUSE (v2-c4.3)](#pause-v2-c43)
        * [PLE(PAUSE-loop exiting)](#plepause-loop-exiting)
    * [hlt](#hlt)
        * [虚拟化环境下](#虚拟化环境下)
    * [mwait && monitor](#mwait--monitor)
        * [hlt and mwait/monitor](#hlt-and-mwaitmonitor)
        * [虚拟化环境下](#虚拟化环境下-1)
    * [参考](#参考)

<!-- vim-markdown-toc -->

categories: [内核,虚拟化]
tags: [intel手册,内核]

# pause, hlt, monitor/mwait

---

## pause && PAUSE-loop exiting(PLE)

### PAUSE (v2-c4.3)
Operation: Execute_Next_Instruction(DELAY);

The PAUSE instruction is provided to improve the performance of “spin-wait loops”

- Improves the performance of spin-wait loops.

> When executing a “spin-wait loop,” processors will suffer a severe performance penalty when exiting the loop because it detects a possible memory order violation. The PAUSE instruction provides a hint to the processor that the code sequence is a spin-wait loop. The processor uses this hint to avoid the memory order violation in most situations, which greatly improves processor performance. For this reason, it is recommended that a PAUSE instruction be placed in all spin-wait loops.

- An additional function of the PAUSE instruction is to reduce the power consumed by a processor while executing a spin loop

> A processor can execute a spin-wait loop extremely quickly, causing the processor to consume a lot of power while it waits for the resource it is spinning on to become available. Inserting a pause instruction in a spin- wait loop greatly reduces the processor’s power consumption.

- v3-c8.10.6.1 Use the PAUSE Instruction in Spin-Wait Loops

![use pasue instruction in spin-wait loops](https://media.githubusercontent.com/media/renyi-github/resources/master/pictures/2019/use_pause_in_spin_wait_loops.png)

### PLE(PAUSE-loop exiting)

虚拟化环境下，PAUSE命令的表现受到VM-execution两个控制项影响：
- PAUSE exiting:内核见```CPU_BASED_PAUSE_EXITING```
- PAUSE-loop exiting:内核见```SECONDARY_EXEC_PAUSE_LOOP_EXITING,PLE_GAP,PLE_WINDOW ```

详见intel手册
![pause ple 1](https://media.githubusercontent.com/media/renyi-github/resources/master/pictures/2019/pause_ple_1.png)
![pause ple 2](https://media.githubusercontent.com/media/renyi-github/resources/master/pictures/2019/pause_ple_2.png)

---

## hlt

![use pasue instruction in spin-wait loops](https://media.githubusercontent.com/media/renyi-github/resources/master/pictures/2019/hlt_instruction.png)

- If all logical processors within a physical package are halted, the processor will enter a power-saving state

### 虚拟化环境下

- HLT. The HLT instruction causes a VM exit if the “HLT exiting” VM-execution control is 1.
- 内核相关参数：```CPU_BASED_HLT_EXITING```


---

## mwait && monitor

**简介:**
- monitor:Sets up an address range used to monitor write-back stores.
- mwait:Enables a logical processor to enter into an optimized state while waiting for a write-back store to the address range set up by the MONITOR instruction

mwait与monitor命令必须搭配使用，其中monitor命令指定要监控的地址范围，mwait开始监控
使用方法如下：
![monitor and mwait use](https://media.githubusercontent.com/media/renyi-github/resources/master/pictures/2019/monitor_mwait_use.png)

mwait主要有两个用途
- MWAIT for Address Range Monitoring: 等待监控的内存区域发生写操作
- MWAIT for Power Management: 执行mwait指令时，可以指定进入不同的C-state


### hlt and mwait/monitor
hlt指令通常用于idle的处理流程中，当OS中任务空闲时，idle处理中主动调用hlt让cpu进入C-state，节约共享资源（如在hyper thread时）降低能耗，但是要让其退出这样的状态就需要其它cpu发送一个IPI中断过来，处理IPI中断会导致延迟增加等问题，因此建议使用mwait/monitor命令
使用对比：
- 常规使用hlt指令
![hlt in idle](https://media.githubusercontent.com/media/renyi-github/resources/master/pictures/2019/hlt_in_idle1.png)
![hlt in idle](https://media.githubusercontent.com/media/renyi-github/resources/master/pictures/2019/hlt_in_idle2.png)

- mwait/monitor改造
![monitor and mwait in idle](https://media.githubusercontent.com/media/renyi-github/resources/master/pictures/2019/monitor_mwait_in_idle.png)
![monitor and mwait in idle](https://media.githubusercontent.com/media/renyi-github/resources/master/pictures/2019/monitor_mwait_in_idle_c1.png)

### 虚拟化环境下
![monitor and mwait in idle](https://media.githubusercontent.com/media/renyi-github/resources/master/pictures/2019/mwait_in_virtual.png)

内核相关控制域：
```
CPU_BASED_MWAIT_EXITING
CPU_BASED_MONITOR_EXITING
```

---

## 参考
- [intel 手册](https://software.intel.com/en-us/download/intel-64-and-ia-32-architectures-sdm-combined-volumes-1-2a-2b-2c-2d-3a-3b-3c-3d-and-4)
	- v3-c8.10 MANAGEMENT OF IDLE AND BLOCKED CONDITIONS
	- v3-c14.6 MWAIT EXTENSIONS FOR ADVANCED POWER MANAGEMENT
- cpu_idle_loop
