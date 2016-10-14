# Application Level

##### **Initialization tax & I/O Size **- Transferring I/O incurs overhead such as: initializing buffers, making a system call, context switching, allocating kernel metadata, checking process privileges & limits, mapping addresses to devices, executing kernel and drive code to delivery I/O and freeing metadata and buffers. This tax occurs for both small and large I/O alike. The more I/O transferred by each I/O, the better. Increasing I/O size can increase performance by minimizing overhead. 1 x 128kb I/O is more efficient than 128 x 1kb I/O. However, there is a down size when the application does not need to read a large I/O size. For example a DB performing 8kb random read may run more slowly with 128kb I/O size, as 120kb will be wasted (introducing I/O latency). 

**Summary: **I/O size should match the average request size of the application. 

##### **Buffering - **To improve write performance, data may be coalesced in a buffer before being sent to the next level, increasing the I/O size and efficiency of the operation. This may increase write latency also, as the first write to a buffer waits for additional ones before being sent. 

##### Concurrency and Parallelism -

Concurrency is the ability to load and begin executing multiple runnable programs. They do not necessarily execute on-CPU at the same instant. Each of these programs may be an application process. Different functions within an application can be made concurrent (multiprocessing, multithreading). Event-based concurrency uses an event loop in order to service different functions typically with a single thread (Node.js). 

Parallelism takes advantage of a multiprocessor system. Applications engaging in parallelism executes over multiple CPUs. Apart from increased throughput of CPU work, multiple threads allow I/O to be performed concurrently, as other threads can execute while a thread blocked on I/O waits. 

**TODO: Synchronization primitives and hash table of locks **

##### **Non-Blocking I/O - **Normal process operation would have a process block and enter a sleep state during I/O. This negatively affects concurrent I/O (application must make many threads\processes to handle I/O concurrently) along with being inefficient  for frequent short lived I/O (overhead costs). Non-Blocking I/O issues I/O without blocking.

##### **Process Binding -** Binding a process or thread to a specific CPU. This can improve memory locality of the application, reducing the cycles for memory I/O and improving overall application performance. The OS usually handles this for us. Be careful of this concept in shared cloud server environments (AWS), you may not be the only one binding to a an underlying physical CPU.

##### **Thread State Analysis** **- **At a minimum two thread states exist: On-CPU (executing) and Off-CPU (waiting for a turn on-CPU, or for I/O, locks, paging, work, and so on…). If most of the time is spent on-CPU, cpu profiling can be applied. If off-CPU other methodologies must be used. Digging deeper Unix like systems provide six states (these are loosely defined)

1. Executing: on-CPU

2. Runnable: waiting for a turn on-CPU

3. Anonymous paging: runnable, but blocked waiting for anonymous page-ins

4. Sleeping: waiting for I/O, including network, block, and data/text page-in

5. Lock: waiting to acquire a synchronization lock (waiting on someone else)

6. Idle: waiting for work

Performance is increased by reducing the time in the first 5 states. You can investigate these states as follows:

1. Executing: Check whether this is user or kernel mode time and the reason for CPU consumption by using profiling. This should give you a hint of which code paths are taking the most time on CPU and show lock spins.

2. Runnable: Spending time in this state means the application needs more CPU resources. Examine CPU load for the entire system, and any CPU limits present for the application (e.g. resource controls)

3. Anonymous paging: A lack of available main memory for the application can cause anonymous paging and delays. Examine memory usage for the entire system and any memory limits presented for the application. 

4. Lock: Identify the lock, the thread holding it, and the reason why the holder held it for so long. 

Analyzing these symptoms:

1. Time spent executing: *top *reports this as %CPU

2. Runnable: Can be viewed via the kernel’s *schedstats* feature exposed via /proc/*/schedstat. *Perf sched* (perf is a deep profiling tool) can be used also

3. Anonymous paging: Also known as *swapping* can be found if the kernel feature *delay accounting *is enabled (not typically) Dtrace and SystemTap can be used also (profiling tools) 

4. Sleeping: Can be determined indirectly with tools such as *pidstat -d * and seeing if the process is performing I/O. If the application is spending a considerable amount of time sleeping due to I/O the tool *pstack *which takes a snapshot of the process's user space stack. 

##### Syscall Analysis - 

Studying process execution based on syscall time. We now want to analyze **Executing: **on-CPU (user mode)** ** and **Syscalls: **time during a system call (kernel mode running or waiting). Tools to use for this include *strace, Dtrace, and SystemTap*

##### **I/O Profiling -** Using a tool such as *Dtrace* to profile a process’s I/O system calls. These profiles can give you an idea of which functions an application is running take the most time to complete. 

# CPU Level

##### Terminology: 

<table>
  <tr>
    <td>Processor</td>
    <td>Physical chip that plugs into a socket on teh system or processor board and containers one or more CPUs implemented as cores or hardware threads</td>
  </tr>
  <tr>
    <td>Core</td>
    <td>An independent CPU instance on a multicore processor. The use of cores is a way to scale processors, called chip-level multiprocessing </td>
  </tr>
  <tr>
    <td>Hardware Thread</td>
    <td>A CPU architecture that supports executing multiple threads in parellel on a single core (Hyper-Threading), which each thread is an independent CPU instance. </td>
  </tr>
  <tr>
    <td>CPU instruction</td>
    <td>A single CPU operation, from its instruction set. There are instructions for arithmetic operations, memory I/O, and control logic. </td>
  </tr>
  <tr>
    <td>Logical CPU</td>
    <td>Also called a virtual processor, an operating system CPU instance (a schedulable CPU entity). This may be implemented by the processor as a hardware thread (in which case it may be called a virtual core), a core, or a single-core processor</td>
  </tr>
  <tr>
    <td>Scheduler</td>
    <td>The kernel subsystem that assigns threads to run on CPUs</td>
  </tr>
  <tr>
    <td>Run queue</td>
    <td>A queue of runnable threads that are waiting to be serviced by CPUs. </td>
  </tr>
</table>


##### CPU Architecture:

**Physical**

<table>
  <tr>
    <td>Processor

Core

HThread
HThread

Core

HThread
HThread

Core

HThread
HThread


Core

HThread
HThread


</td>
  </tr>
</table>


**Seen by OS**

<table>
  <tr>
    <td>
CPU 
0
CPU
2
CPU
4
CPU
6
CPU
1
CPU
3
CPU
5
CPU
7
</td>
  </tr>
</table>


##### **Utilization **- CPU utilization is measured by the time a CPU instance is busy performing work during an interval, expressed as a percentage. CPU performance does not degrade under high usage alone. The measure of utilization spans all clock cycles including memory access (High CPU can actually be indication of frequent memory stalls waiting for memory I/O.) 

##### User-Time/Kernel-Time - The CPU time spent executing user-level application code is called user-time, and kernel-level code is kernel-time. Kernel-time includes time during system calls, kernel threads, and interrupts. When measured across the system the user/kernel time ratio indicates the type of workload being performed. Applications that are CPU bound may spend almost all their time executing user-level code and have a user/kernel ratio approaching 99/1. Applications that are I/O-intensive have a high rate of system calls, which execute code to perform the I/O. 

**Summary****: **CPU-Bound workloads = high user time. I/O bound workloads = high kernel time

##### **Saturation ****-** A CPU at 100% utilization is saturated and threads will encounter scheduler latency (waiting to run on-CPU). 

##### **Scheduling classes (threads)**** - **Scheduling classes manage the behavior of runnable threads, specifically their priorities (**nice values**), whether their on-CPU time is time-sliced, and the duration of those time-slices (time quantum). Additional controls are found via **scheduling policies **which may be selected within a scheduling class and can control scheduling between threads of the same priority. In Linux the nice value sets the **static priority** of the thread, which is separate from the dynamic priority that the scheduler calculates. 

Linux provides the following Thread scheduler classes:

* RT: provides fixed and high priorities for real-time workloads. The kernel supports both user- and kernel-level preemption, allowing RT tasks to be dispatched with low latency. The priority range is 0-99. 

    * Policies: 

        * RR: SCHED_RR is round-robin scheduling. Once a thread has used its time quantum, it is moved to the end of the run queue for that priority level, allow others of the same priority to run.

        * FIFO: SCHED_FIFO is first-in first-out scheduling, which continues running the thread at the head of the run queue until it voluntarily leaves, or until a higher-priority thread arrives. The thread continues to run, even if other threads of the same priority are on the run queue

* O(1): Introduced in Linux 2.6 as the default time-sharing scheduler for user processes. This scheduler dynamically improves the priority of I/O-bound over CPU-bound workloads, to reduce latency of interactive and I/O workloads. Policies: Normal/Batch

* CFS: Completely fair scheduling was added in linux 2.6.23 kernel as the default time-sharing scheduler for user processes. The scheduler manages tasks on a red-black tree instead of traditional run queues, which is keyed from the task CPU time. This allows low CPU consumers to be easily found and executed in preference to CPU-bound workloads, improving the performance of interactive and I/O-bound workloads

    * Policies (Applying to both O(1) and CFS)

        * NORMAL: SCHED_NORMAL is time-sharing scheduling and is the default for user processes. The scheduler dynamically adjusts priority based on the scheduling based on the scheduling class. For O(1), the time slice duration is set based on the static priority: longer durations for higher priority work. For CFS, the time slice is dynamic. 

        * BATCH: SCHED_BATCH is similar to SCHED_NORMAL, but with the expectation that the thread will be CPU-bound and should not be scheduled to interrupt other I/O bound interactive work

RT workloads are of highest priority followed by O(1)/CFS and with Idle trailing last. 

##### **NUMA Grouping**** - **Performance on NUMA systems can be significantly improved by making the kernel NUMA-aware, so that it can make better scheduling and memory placement decisions. This can automatically detect and create groups of localized CPU and memory resources and organize them in a topology to reflect the NUMA architecture. This topology allows the cost of any memory access to be estimated. On Linux these groupings are called **scheduling domains** which are in a topology beginning with the* root domain.* 

##### **Tools Methodology ****- **This analysis methodology covers the tools that can be used to analyze CPU performance

1. uptime: check load averages to see if CPU load is increasing or decreasing.

2. vmstat: Run vmstat per second, and check the idle columns to see how much headroom there is. Less than 10% can be a problem

3. mpstat: Check for individual hot (busy) CPUs, identifying a possible thread scalability problem

4. top/prstat: See which processes and users are the top CPU consumers

5. pidstat/prstat: Break down the top CPU consumers into user- and system-time

6. perf/dtrace/stap/oprofile: Profile CPU usage stack traces for either user- or kernel-time, to identify why the CPUs are in use

7. perf/cpustat: Measure CPI

##### Tools Rundown for CPU -

vmstat (ran with no flags): 

<table>
  <tr>
    <td>vmstat's first run will report metrics from system start

Sample output: 
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 1625548  15580 364464    0    0    14     7    8   10  0  0 100  0  0

r: number of processes waiting for run time
b: number of processes in uninterruptible sleep

us: spent running non-kernel code (user-time, including nice time)
sy: spent running kernel code. (system time)
id: spend idle 
wa: time spent waiting for IO
st: time stolen from a virtual machine (virtualization specific)
</td>
  </tr>
</table>


mpstat (ran with -P ALL flag, will not repeat metrics explanation when listed above also):

Use mpstat when you want the same information vmstat gives you, but broken down by cpu

<table>
  <tr>
    <td>Sample output:
11:23:32 AM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
11:23:32 AM  all    0.03    0.01    0.04    0.01    0.00    0.00    0.00    0.00    0.00   99.92
11:23:32 AM    0    0.02    0.01    0.03    0.01    0.00    0.00    0.00    0.00    0.00   99.94
11:23:32 AM    1    0.05    0.02    0.04    0.01    0.00    0.00    0.00    0.00    0.00   99.89

cpu: processor number
%nice: percentage of cpu utilization that occurred while executing at the user level with nice priority
%irq: percentage of time spent by the CPU or CPUs to service interrupts
%soft: percentage of time servicing softirqs (software interrupts) 
%steal: same as st from above

</td>
  </tr>
</table>


pidstat (ran 1 second interval pidstat 1 ):

Use pidstat when you want the same information vmstat gives you, but broken down by PID

<table>
  <tr>
    <td>11:43:41 AM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
11:43:42 AM     0       147    0.00    0.99    0.00    0.99     1  kworker/1:2

11:43:42 AM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
11:43:43 AM     0         7    0.00    1.00    0.00    1.00     1  rcu_sched


</td>
  </tr>
</table>


**Summary - **When looking for **system wide** CPU information use **_vmstat_***. *When looking for **per virtual core** CPU information use **_mpstat_**. When looking for **per process** CPU information use **_pidstat_***. *

##### Tuning - 

* Scheduling Priority and Class

    * *nice *command can be used to adjust process priority. Positive nice values decrease priority, and negative nice values increase priority. 

    * *chrt *command can show and set the scheduling priority directly, and the scheduling policy. 

* Process Binding

    * *taskset *command uses a CPU mask or ranges to set CPU affinity

* Exclusive CPU sets

    * Linux provides *cpusets*, which allow CPUs to be grouped and processes assigned to them. This can improve performance similarly to process binding. But performance can be improved further by making cpuset exclusive -- preventing other processes from using it. 

    * Reference documentation for full details (it’s a pseudo file system that needs to be mounted similar to Cgroups.) 

* Resource Controls

    * Linux has container groups (Cgroups), which control resource usage by processes or groups of processes. CPU usage can be controlled using shares, and the CFS scheduler allows fixed limits to be imposed (CPU bandwidth), in terms of allocating microseconds of CPU cycles per interval. 

# Memory Level

##### Performance factor overview -

* Main memory stores application and kernel instructions, their working data, and file system caches. Exhausting this resource leads to the disk subsystem being used in it’s place, which is orders of magnitude slower

* CPU expense of allocating and freeing memory

* Copying memory

* Managing memory address space mapping

* Memory locality in multi socket architectures (NUMA) (memory attached to local sockets have lower access latency than remote sockets)

##### Terminology 

<table>
  <tr>
    <td>Main memory</td>
    <td>Alos referred to as physical memory, this describes the fast data storage area of a computer, commonly provided as DRAM</td>
  </tr>
  <tr>
    <td>Virtual memory</td>
    <td>An abstraction of main memory that is (almost) infinite and non contended. Virtual memory is not real memory. </td>
  </tr>
  <tr>
    <td>Resident memory</td>
    <td>Memory that currently resides in main memory</td>
  </tr>
  <tr>
    <td>Anonymous memory</td>
    <td>Memory with no file system location or path name. It includes the working data of a process address space, called the heap</td>
  </tr>
  <tr>
    <td>Address space </td>
    <td>A memory context, there are virtual address spaces for each process, and for the kernel</td>
  </tr>
  <tr>
    <td>Segment</td>
    <td>An area of memory flagged for a particular purpose, such as for storing executable or writable pages</td>
  </tr>
  <tr>
    <td>OOM</td>
    <td>Out of memory, when the kernel detects low available memory.</td>
  </tr>
  <tr>
    <td>Page</td>
    <td>A unit of memory, as used by the OS and CPUs. Historically it is either 4 or 8 kb. Modern processes have multiple page size support for larger sizes.</td>
  </tr>
  <tr>
    <td>Page fault</td>
    <td>An invalid memory access. These are normal occurrences when using on-demand virtual memory.</td>
  </tr>
  <tr>
    <td>Paging</td>
    <td>The transfer of pages between main memory and the storage devices</td>
  </tr>
  <tr>
    <td>Swapping</td>
    <td>From Unix, this is the transfer of entire processes between main memory and the swap devices. Linux often uses swapping to refer to paging to the swap device. </td>
  </tr>
  <tr>
    <td>Swap</td>
    <td>An on-disk area for paged anonymous data and swapped processes. It may be an area on a storage device, also called a physical swap device, or a file system file, called a swap file. Some tools use the term swap to refer to virtual memory (which is confusing and incorrect)</td>
  </tr>
</table>


##### **File system paging**** - **File system paging is caused by the reading and writing of pages in memory-mapped files. This is attributed to *mmap()* system calls. When needed, the kernel can free memory by paging some out. This is where the terminology gets a bit tricky: if a file system page has been modified in main memory ("dirty"), the page-out will require it to be written to disk. If, instead, the file system page has not been modified (“clean”), the page-out merely frees the memory for immediate reuse, since a copy already exists on disk. Because of this, the term page-out means that a page was moved out of memory—this may or may not have included a write to a storage device (you may see this defined differently).

##### **Anonymous Paging**** -** Anonymous paging involves data that is private to processes: the process heap and stacks. Anonymous term is used because it has no named location in the operating system (no file system path name). Anonymous paging requires moving the data to the physical swap devices or swap files (swapping). This is considered "bad" paging and hurts performance. Anonymous page-in’s block, anonymous page-outs can be performed asynchronously by the kernel. 

##### **Demand Paging and Faults**** -** maps pages of virtual memory to physical memory on demand. This defers the CPU overhead of creating the mappings until they are actually needed and accessed, instead of at the time a range of memory is first allocated. A page fault occurs as a page is accessed when there is initially no page mapping for virtual to physical memory. If the mapping can be satisfied from another page in memory, it is called a **minor fault**. Page faults that require storage device access such as accessing an uncached memory-mapped file, are called **major faults. **Due to demand allocation any page of virtual memory can be in the following states:

1. Unallocated

2. Allocated, but unmapped (unpopulated and not yet faulted)

3. Allocated, and mapped to main memory (RAM)

4. Allocated, and mapped to the physical swap device (disk) (if page is paged out to swap device)

Transition from B to C is a page fault. If it requires disk I/O it’s a major fault; otherwise it’s a minor page fault. From these states, two memory usage terms can be also defined:

* Resident set size (RSS): the size of allocated main memory pages (c) 

* Virtual memory size: the size of all allocated areas (B + C + D)

**Summary: **File system paging = paging of known memory-mapped files (files which have a known place on the file system and are identifiable to the OS). Anonymous Paging = Paging of a process’s private address space, this is a performance issue! Demand paging = the deferring of virtual to physical memory until necessary. Page faults are the method in which demand paging is implemented. Minor fault = When the mapping can happen directly to memory. Major fault = when the mapping needs to happen to disk (performance issue!). RSS = The size of allocated main memory pages. Virtual memory size = ALL allocated area 

##### Overcommit - 

Allows more memory to be allocated than the system can possibly store -- more than physical memory and swap devices combined. Relies on demand paging and the tendancy of applications to not use much of the memory they have allocated. This is a tunable feature (see tuning section)

##### **Swapping -** 

The movement of an entire process between main memory and the physical swap device or swap file. This is the original Unix technique for managing main memory. This swap includes all private data unless it’s data originating from the file system, in which case the file system is referred to when swapping back into memory. A small amount of process metadata always resides in kernel memory. This is a performance intensive operation. 

##### **File system Cache Usage** **- **The OS uses available memory to cache the file system, improving performance. Memory used for the file system cache can be thought of us "unused" from a systems resource perspective. 

##### **Allocators**** - **User-land libraries or kernel-based routines, which provide the software programmer with an easy interface for memory usage (malloc(), free()). These can affect performance significantly. There are a variety of user- and kernel-level allocators for memory allocation. 

	Features:

* Simple API: for example, malloc(), free()

* Efficient memory usage: When servicing memory allocations of a variety of sizes, memory usage can become fragmented, where there are many regions that waste memory. Allocators can strive to coalesce the unused regions, so that larger allocations can make use of them, improving efficiency. 

* Performance: Memory allocations can be frequent, and on multithreaded environments they can perform poorly due to contention for synchronization primitives. Allocators can be designed to use locks sparingly and also make useof per-thread or per-CPU caches to improve memory locality. 

* Observability: An allocator may provide statistics and debug modes to show how it is being used, and which code paths are responsible for allocations. 

**Slab allocator: **Manages caches of objects of a specific size, allowing them to be recycled quickly without the overhead of page allocation. Used heavily in kernel allocations, which are frequently for fixed-size structs.

**Slub allocator: **Addresses shortcomings of Slab allocator (time complexity) and is now default allocator in Linux. 

**glibc **

##### Freeing Memory -

Linux has several methods to add pages back to the free list (freeing memory)

![image alt text](image_0.png)

*Linux OS pictures on left, Solaris on right

The methods for freeing memory consist of:

* **Free List**: a list of pages that are unused (also called idle memory) and available for immediate allocation. This is usually implemented as multiple free page lists, one for each locality group (NUMA)

    * Linux uses the buddy allocator for managing pages providing mutiple free lists for different size memory allocations, following a power of two scheme. Buddy refers to finding neighboring pages of free memory so they can be allocated together. 

* **Reaping**: when a low memory threshold is crossed, kernel modules and kernel slab allocator can be instructed to immediately free any memory that can easily be freed. Also known as shrinking

    * Mostly involves freeing memory from the kernel slab allocator caches. These caches contain unused memory in slab-size chunks, ready for reuse. Reaping returns this memory to the system for page allocations. 

On Linux, specifically the methods are:

* **Page cache**: A cache for file system reads and writes. Parameter **swappiness **sets preference of freeing memory from the page cache verse swapping.

* **Swapping\kswapd**: controlled by the page-out daemon **kswapd**. This daemon finds not-recently-used pages to add to the free list, including application memory. Once located they are paged-out which involves writing these pages to a swap file or swap device. 

    * **Page Scanning: **When available memory in the free list drops below a threshold, the page-out daemon begins scanning pages for removal. Page scanning is an indication of memory pressure. 

    * Once free memory has reached the lowest threshold, kswapd operates in synchronous mode, freeing pages of memory as they are requested. Tunable lower threshold for this action: vm.min_free_kbytes.

    * The page cache (file system cache) keeps lists for inactive pages and active pages. These operate in LRU fashion, allow kswapd to find free pages quickly:

        * kswapd scans the inactive list first, and then the active if needed. Scanning means walking the lists. If pages are locked or dirty it may be ineligible for being freed. 

* **OOM Killer**: finds and kills sacrificial processes, found using select_bad_process() and then killed by calling oom_kill_process().  

##### **Process Address Space** - range of virtual pages that are mapped to physical pages as needed. Split into areas called **segments** for storing thread stacks, process executable, libraries, and heap. 

![image alt text](image_1.png)

The program executable & Libraries segment contains separate text and data segments (Not pictured above). 

Segments:

* **Executable text**: contains the executable CPU instructions for the process. This is mapped from the text segment of the binary program on the file system. It is read-only with the execute permission

* **Executable data: **contains initialized variables mapped from the data segment of the binary program. This is read/write permissions, so that the variables can be modified while the program is running. It also has a private flag, so that modifications are not flushed to disk.

* **Heap**: This is the working memory for the program and is anonymous memory (no file system location). It grows as needed.

* **Stack: **stacks of the running threads, mapped read/write

##### Tools Methodology:

* **Page scanning: **Look for continous page scanning (more than 10s) as a sign of memory pressure. *sar -B *and checking the *pgscan *columns. 

* **Paging**: Paging of memory is  further indication that the system is low on memory. *vmstat *and check *si *and *so* columns. 

* **vmstat: **Run vmstat per second and check *free *column for available memory

* **OOM Killer: **Check syslog and dmesg for Out of memory messages

* **top: **Which processes and users are the top physical memory consumers (resident) and virtual memory consumers 

* **dtrace/stap/perf: **Trace memory allocation with stack traces, to identify the cause of memory usage. 

##### Tools Rundown for CPU -

<table>
  <tr>
    <td>vmstat (ran with 1 second intervals, no flags)

procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 2666948 628620 22845380    0    0     1    52    1    1  3  1 97  0  0
 0  0      0 2666948 628620 22845380    0    0     0     0  926 1661  4  1 95  0  0
 0  0      0 2666948 628620 22845380    0    0     0     0  872 1619  2  0 97  0  0

swpd: amount of swapped out memory
free: free available memory
buff: memory in the buffer cache
cache: memory in the page cache
si: memory swapped in (paging)
so: memory swapped out (paging) 

Shown with -a options:

inact: inactive memory in the page cache
active: memory in the page cache </td>
  </tr>
</table>


<table>
  <tr>
    <td>sar 

-B: paging statistics
-H: huge pages statistics
-r: memory statistics
-R: memory statistics
-W: swapping statistics

</td>
  </tr>
</table>


<table>
  <tr>
    <td>slabtop (ran with -sc flags)

 Active / Total Objects (% used)    : 6753116 / 6774957 (99.7%)
 Active / Total Slabs (% used)      : 185021 / 185021 (100.0%)
 Active / Total Caches (% used)     : 80 / 113 (70.8%)
 Active / Total Size (% used)       : 1014591.66K / 1020285.58K (99.4%)
 Minimum / Average / Maximum Object : 0.01K / 0.15K / 15.44K

  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME                   
5346549 5341452  99%    0.10K 137091       39    548364K buffer_head
171461 171461 100%    1.01K   5531       31    176992K ext4_inode_cache
479220 478849  99%    0.19K  22820       21     91280K dentry
113360 111131  98%    0.61K   4360       26     69760K proc_inode_cache
120904 119866  99%    0.57K   4318       28     69088K radix_tree_node
 18928  18928 100%    0.55K    676       28     10816K inode_cache

Statistics at top.
OBJS: object count within slab cache
ACTIVE: active objects in slab cache
OBJ SIZE: the size of an individual object
CACHE SIZE: total size of the cache</td>
  </tr>
</table>


<table>
  <tr>
    <td>ps (ran with aux flags)

USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0 185448  6040 ?        Ss   Oct09   0:10 /sbin/init noprompt persistent text
root         2  0.0  0.0      0     0 ?        S    Oct09   0:00 [kthreadd]
root         3  0.0  0.0      0     0 ?        S    Oct09   0:01 [ksoftirqd/0]
root         5  0.0  0.0      0     0 ?        S<   Oct09   0:00 [kworker/0:0H]
root         7  0.0  0.0      0     0 ?        S    Oct09   3:02 [rcu_sched]
root         8  0.0  0.0      0     0 ?        S    Oct09   0:00 [rcu_bh]

%MEM: main memory usage (phsyical memory, RSS) as a percentage of the total in the system
RSS: resident set size (Kbytes)
VSZ: virtual memory size (Kbytes)</td>
  </tr>
</table>


<table>
  <tr>
    <td>pmap (ran with flag targeting a pid and -X)
Lists the memory mappings of a process, showing their sizes, permissions, and mapped objects

         Address Perm   Offset Device    Inode   Size   Rss   Pss Referenced Anonymous Swap Locked Mapping
        00400000 r-xp 00000000  08:22 12219668    304   304   304        304         0    0      0 remmina
        0064c000 r--p 0004c000  08:22 12219668      4     4     4          4         4    0      0 remmina
        0064d000 rw-p 0004d000  08:22 12219668     12    12    12         12        12    0      0 remmina
        023ae000 rw-p 00000000  00:00        0   8748  8484  8484       8484      8484    0      0 [heap]
    7f5724000000 rw-p 00000000  00:00        0    136    16    16         16        16    0      0 
    7f5724022000 ---p 00000000  00:00        0  65400     0     0          0         0    0      0 
    7f572b000000 rw-s 00000000  00:05  7897100  16384  3384  1692       3384         0    0      0 SYSV00000000 (deleted)
    7f572c000000 rw-p 00000000  00:00        0    136    12    12         12        12    0      0 
    7f572c022000 ---p 00000000  00:00        0  65400     0     0          0         0    0      0 
    7f5730000000 rw-p 00000000  00:00        0    136    12    12         12        12    0      0 
    7f5730022000 ---p 00000000  00:00        0  65400     0     0          0         0    0      0 
    7f5734000000 rw-p 00000000  00:00        0    696   140   140        140       140    0      0 
    7f57340ae000 ---p 00000000  00:00        0  64840     0     0          0         0    0      0 
    7f5738f28000 r-xp 00000000  08:22  6049956     20    20     5         20         0    0      0 libnss_dns-2.21.so
    7f5738f2d000 ---p 00005000  08:22  6049956   2044     0     0          0         0    0      0 libnss_dns-2.21.so
    7f573912c000 r--p 00004000  08:22  6049956      4     4     4          4         4    0      0 libnss_dns-2.21.so
    7f573912d000 rw-p 00005000  08:22  6049956      4     4     4          4         4    0      0 libnss_dns-2.21.so
    7f5739130000 r-xp 00000000  08:22  6045442      8     8     0          8         0    0      0 libnss_mdns4_minimal.so.2
    7f5739132000 ---p 00002000  08:22  6045442   2044     0     0          0         0    0      0 libnss_mdns4_minimal.so.2
    7f5739331000 r--p 00001000  08:22  6045442      4     4     4          4         4    0      0 libnss_mdns4_minimal.so.2
    7f5739332000 rw-p 00002000  08:22  6045442      4     4     4          4         4    0      0 libnss_mdns4_minimal.so.2
    7f5739338000 r-xp 00000000  08:22  6041455     40    40     3         40         0    0      0 libnss_files-2.21.so
    7f5739342000 ---p 0000a000  08:22  6041455   2048     0     0          0         0    0      0 libnss_files-2.21.so
    7f5739542000 r--p 0000a000  08:22  6041455      4     4     4          4         4    0      0 libnss_files-2.21.so
    7f5739543000 rw-p 0000b000  08:22  6041455      4     4     4          4         4    0      0 libnss_files-2.21.so



</td>
  </tr>
</table>


##### Tuning -

Kernel Params:

![image alt text](image_2.png)

Multiple Page Sizes:

Large page sizes can improve memory I/O performance by improving the hit ratio of the TLB cache (increasing its reach). Enable **Huge Pages** and look into** Transparent Huge Pages (THP)**

Resource Controls: 

Basic resource controls, including setting a main memory limit and a virtual memory limit using *ulimit*. Linux has Cgroups also:

* memory.memsw.limit_in_bytes: the maximum allowed memory and swap space, in bytes 

* memory.limit_in_bytes: the maximum allowed user memory, including file cache usage,in 

* bytes memory.swappiness: similar to vm.swappiness described earlier but can be set for a cgroup 

* memory.oom_control: can be set to 0, to allow the OOM killer for this cgroup, or 1, to disable it

# File System Level

##### Terminology

<table>
  <tr>
    <td>File system</td>
    <td>An organization of data as files and directories, with a file-based interface for accessing them, and file permissions to control access. Additional content may include special file types for devices, sockets, and pipes and metadata including file access timestamps</td>
  </tr>
  <tr>
    <td>File system cache</td>
    <td>An area of main memory (usually DRAM) used to cache file system contents, which may include different caches for various data and metadata</td>
  </tr>
  <tr>
    <td>Operations</td>
    <td>The requests of the file system, including read(), write(). open(), close(), stat(). mkdir() and other operations</td>
  </tr>
  <tr>
    <td>I/O</td>
    <td>input/output. File system I/O can be defined in several ways; her it is used to mean only operations that directly read and write (performing I/O) including read(), write(), stat()(read statistics), and mkdir() (write a new directory). I/O does not include open() and close()</td>
  </tr>
  <tr>
    <td>Logical I/O</td>
    <td>I/O issues by the application to the file system</td>
  </tr>
  <tr>
    <td>Physical I/O</td>
    <td>I/O issued directly to the disks by the file system (or via raw I/O)</td>
  </tr>
  <tr>
    <td>Throughput</td>
    <td>The current data transfer rate between applications and the file system, measured in bytes per second</td>
  </tr>
  <tr>
    <td>inode</td>
    <td>Data structure containing metadata for a file system object, including permissions, timestamps, and data pointers</td>
  </tr>
  <tr>
    <td>VFS</td>
    <td>Virtual file system, a kernel interface to abstract and support different file system types. </td>
  </tr>
  <tr>
    <td>Volume manager</td>
    <td>Software for managing physical storage devices in a flexible way, creating virtual volumes from them for use by the OS.</td>
  </tr>
</table>


**File System Interfaces -**

A basic model of a file system in terms of interfaces

![image alt text](image_3.png)

**File System Cache -**

A generic file system cache stored in main memory, servicing a read operation

![image alt text](image_4.png)

**Concepts -**

File system latency -

The primary metric of file system performance. The time from a logical file system request to it’s completion. Inclusive of time spent in the file system, kernel disk I/O subsystem, and waiting on disk devices (Physical I/O). Processes often block during I/O requests, making I/O directly proportional to application performance (Unless Non-Blocking I/O techniques are used or when I/O is initiated from an asynchronous thread). Monitoring I/O latency has been historically difficult and focuses on disks performance. Threads doing backround flushes of I/O to disk will look like high bursts of disk I/O latency, however no application is blocked by this. Tracing and profiling are necessary to identify these occurences. 

Caching -

The file system will use main memory (RAM) as a cache to improve performance. Applications logical I/O latency improves (no latency occurs from disk access). Cache memory grows while free memory shrinks as the system remains operation, this is normal and page cache memory can be thought of as "free" however removing pages from the page cache can affect I/O operation latency. 

Multiple types of cache are used by the file system and block device subsystem. 

![image alt text](image_5.png)

Random vs Sequential I/O -

A series of logical file system I/O can be described as *random * or *sequential*, based on the file offset of each I/O. Sequential I/O means the next I/O begins at the end of the previous I/O. Random I/O have no apparent relationship between them, and the offset changes randomly. 

![image alt text](image_6.png)

Due to performance characteristics of spinning disks, file systems have historically attempted to reduce random I/O by placing file data on disk sequentially and contiguously. This reduces *fragmentation*. 

Prefetch (AKA read-ahead) -

Some I/O workloads are too large for memory or will most likely be read into cache once and not accessed again (removing it from cache quickly if it's an LRU cache). Prefetch combats this by predicting sequential read workloads based on current and previous file I/O offsets. Prefetch caches additional blocks that is *assumes *the application will be requesting, if the application does request these blocks they will be in cache when it attempts the request. Here’s an example

1. An application issues a file read(), passing execution to the kernel

2. The file system issues the read from disk

3. The previous file offset pointer is compared to the current location, and if they are sequential, the file system issues additional reads

4. The first read completes, and the kernel passes the data and execution back to the application

5. Any additional reads complete, populating the cache for future application reads.

![image alt text](image_7.png)

     *Application reads to offset 1 and then 2 tigger prefetch for the next 3 offsets*

Prefetch is a tunable feature

Write-Back Caching -

Treats writes as completed after the transfer to main memory, and writing them to disk happens sometime later, asynchronously. The file system process for writing this "dirty" data to disk is called *flushing. *An example:

1. An application issues a file write(), passing execution to the kernel.

2. Data from the application address space is copied into the kernel

3. The kernel treats the write() syscall as completed, passing execution back to the application

4. Sometime later, an asynchronous kernel task finds the written data and issues disk writes. 

The trade-off is reliability. When writes go to memory, if system failure occurs they can be lost. The flush could also happen incompletely leaving behind an on-disk state that is corrupted. 

Synchronous Writes -

Kernel does not return execution to application until write has made it all the way to disk. Two forms:

* Individual Synchronous Writes - Write I/O is synchronous when a file is opened using the flag O_SYNC or one of the variants O_DSYNC and O_RSYNC. Some filesystems have mount options to force all write I/O to all files to be synchronous. 

* Synchronously Committing Previous Writes - Using fsycn() sys call the application flushes writes to disk synchronously as "check-points" (not a technical term) in their code. This can improve performance by grouping synchronous writes at once. 

Raw and Direct I/O -

Raw I/O - issued directly to disk offsets, bypassing the file system all together. 

Direct I/O - Uses the filesystem but does not utilize the page cache. Mapping of file offsets to disk offsets are still performed by filesystem code and I/O may also be resized to match the size used by the file system for on-disk layout (record size). A good time to use Direct I/O is for large write operations you don’t want to populate your page cache with. 

Non-Blocking I/O -

Can avoid the performance or resource overhead of thread creation. Using the O_NONBLOCK or O_NDLEY flags to open() syscall opens a file descriptor that will issue non-blocking I/O reads and writes. When reading and writing the appropriate functions will return ‘EAGAIN’ error instead of blocking, allowing your application to try later. 

Memory-Mapped Files -

Mapping files to the process address space and accessing memory offsets directly. Avoids syscall execution and context switch overheads when calling read() and write(). Can also avoid double copying of data, if the kernel supports direct copying of the file data buffer to the process address space. 

Created with mmap() syscall and removed using munmap(). Mappings can be tuned using madvice(). 

Disadvantage of using mapping on multiprocessor systems can be the overhead to keep each CPU MMU in sync, specifically the CPU cross calls to remove mappings (TLB shootdowns). Can be minimized by delaying TLB updates (lazy shootdowns)

Metadata - 

While data describes the contents of files and directories, metadata describes information about them. May refer to information that can be read from the file system interface (POSIX) or information needed to implement the file system on-disk layout. 

* Logical Metadata - Information that is read and written to the filesystem by consumers (applications), either

    * Explicitly: reading file statistics (stat()), creating and deleting files (creat(), unlink()) and directories (mkdir(), rmdir())

    * Implicitly: file system access timestamp updates, directory modification timestamp updates

* Physical Metadata

    * The on-disk layout metadata necessary to record all file system information. Depends on file system type and can include superblocks, inodes, blocks of data pointers (primary, secondary, …) and free lists.

A workload can be "metadata-heavy" typically refers to logical metadata, for example, web servers that stat() files to ensure they haven’t changed since caching, at a much greater rate than actually reading file data contents. 

Logical versus Physical I/O

I/O requested by applications to the file system (logical I/O) may not match disk I/O (physical I/O) for several resources File systems cache reads, buffer writes, and create additional I/O to maintain the on-disk physical layout metadata needed to record where everything is. This causes unrelated disk I/O (both inflated and deflated) in relation to the original application I/O call. These can be characterized as follows: 

* Unrelated:

    * Other applications: The disk I/O is from another application

    * Other tenants: The disk I/O is from another tenant (Virtualization)

    * Other kernel tasks: kernel rebuilding a software RAID coume or performing async file system checksum verification (for example) 

* Indirect: 

    * File system prefetch: adding additional I/O that may not be used by the application

    * File system buffering: write-back caching defers and coalesce write for later flushing to disk. May appear as large, infrequent bursts

* Deflated - Where disk I/O is smaller than application I/O or even nonexistent:

    * File system caching: satisfying reads from main memory instead of disk

    * File system write cancellation: The same byte offsets are modified multiple times before flushed once to disk. 

    * Compression: reducing the data volume from logical to physical I/O 

    * Coalescing: merging sequential I/O before issuing them to disk. 

    * In-memory file system: Content may never be written to disk (tmpfs)

* Inflated - Where disk I/O is larger than application I/O:

    * File system metadata: adding additional I/O

    * File system record size: rounding up I/O size (inflating bytes), or fragmenting I/O (inflating count)

    * Volume manager parity: read-modify-write cycles, adding additional I/O

Example of 1-Byte application write -

1. An application performs a 1-byte write to an existing file.

2. The file system identifies the location as part of a 128 Kbyte file system record, which is not cached (but the metadata to reference it is)

3. The file system requests that the record be loaded from disk

4. The disk device layer breaks the 128 Kbyte read into smaller reads suitable for the device

5. The disks perform multiple smaller reads, totalling 128 Kbytes.

6. The file system now replaces the 1 byte in the record with the new byte.

7. Sometime later, the file system requests that the 128 Kbyte dirty record be written back to disk

8. The disks write the 128 Kbyte record (broken up if needed)

9. The file system writes new metadata, for example, references (for copy-on-write) or access time. 

10. The disk performs more writes

So while the application performed only a single 1-byte write, the disk performed multiple reads (128 Kbytes in total) and more writes (over 128 Kbytes). 

File system operation performance -

File system operations can exhibit different performance based on their type. 

Capacity -

When the file system fills, performance may degrade. Writing new data may take more time to locate the free blocks on the disk for computation, and any disk I/O needed. Areas of free space on disk are likely to be smaller and more sparsely located, degrading performance due to smaller I/O and random I/O. 

##### Architecture:

Generic depiction of file system I/O stack

![image alt text](image_8.png)

VFS:

The virtual file system provide a common interface for different file system types. 

![image alt text](image_9.png)

Linux re uses the terms *inodes *and *superblock* in the VFS domain (which is also present when talking about data’s (and metadata’s) on-disk structures making things slightly confusing. In documentation you’ll see the on disk structures prefixed with the file system name i.e. *ext4_inode *and *ext4_super_block. *

File system caches:

![image alt text](image_10.png)

*Overview of file system caches in linux*

Unix originally had only the buffer cache to improve the performance of block device access. Today Linux have multiple different cache types. 

* Page cache: caches virtual memory pages including mapped file system pages. The size is dynamic and will grow to use available memory, freeing it again when applications need it. 

    * *flush - *flusher threads which are created per device to better balance the per-device workload and improve throughput. Pages are flushed to disk for the following reasons:

        * After an interval (30 s)

        * The sync(), fsync(), or msync() system calls

        * Too many dirty pages (dirty_ratio)

        * No available pages in the page cache

* Dentry Cache: Remebers mappings from directory entry (struct dentry) to VFS inode. Improves path name lookup performance (open()). When path name is traversed, each name lookup can check theDcache for a direct inode mapping, instead of stepping through the directory contents. Size can be seen via /proc. Will shrink via LRU when system needs more memory. 

    * Negative caching: remembering which lookups lead to non-existent entries. This improves performance of failed lookups, which commonly occur for library path lookup.

* Inode Cache: Contains VFS inodes (struct inode), each describing properties of a file system object, many of which are returned via the stat() system call. Inode cache grows dynamically holding at least all inodes mapped by the Dcache. Will shrink with application memory pressure. Size can be seen via /proc

##### File System Features

Block versus Extent:

*  Block-based file systems: store data in fixed-size blocks, referenced by pointers stored in metadata blocks. For large files this can require many block pointers and metadata blocks, and the placement of blocks may become scattered, leading to random I/O. Some block-based file-systems attempt to place blocks contiguously to avoid this. Another approach is to use variable block sizes, so that larger sizes can be used as the file grows, which also reduces metadata overhead. 

* Extent-based filesystems: preallocate contiguous space for files (extents). Growing them as needed. For the cost of space overhead, this improves streaming performance and can improve random I/O performance as file data is localized. 

Journaling:

A *log* recording changes to the file system so that in the event of a system crash, changes can be replayed atomically. Allows file systems to recovery to a consistent state quickly. The journal is written to disk synchronously, and for some file systems it can be configured to use a separate device. Some journals record both data and metadata (I/O written twice, can consume I/O resources) others write only metadata and maintain data integrity by employing copy-on-write.

*Log-structured file system -* consists of only a journal where all data and metadata updates are written to a continuous and circular log. Optimizes write performance, as write are always sequential and can be merged to use larger I/O sizes. 

Copy-on-write:

A filesystem that does not overwrite existing blocks but instead follows these steps:

1. Write blocks to a new location (a new copy)

2. Updates reference to new blocks

3. Add old blocks to the free list

This helps file system integrity in the event of a system failure and also improves performance by turning random writes into sequential ones.

Scrubbing:

Asynchronous reads of all data blocks and verifies checksums, to detect failed drivers as early as possible, ideally while the failure is still recoverable due to RAID. Scrubbing negatively affects performance. 

ToDo: Filesystem types

##### Volumes and Pools -

Allow file systems to be built upon multiple disks and can be configured using different RAID strategies. 

Volumes - present multiple disks as one virtual disk or disk partition (LVM). When built upon whole disks (and not slices or partitions), volumes isolate workloads, reducing performance issues of contention 

Pooled storage - multiple disks in a storage pool, from which multiple file systems can be created. Pooled storage is more flexible than volume storage, as file systems can grow and shrink regardless of backing devices. This approach is used by *ZFS* and *btrfs. *Pooled storage can use all disk devices for all file systems, improving performance. Workloads are not isolated; in some cases, multiple pools may be used to separate workloads, given the trade-off of some flexibility, as disk devices must be initially placed in one pool or another. 

![image alt text](image_11.png)

*       Illustration of Volume vs pooled storage*

 Additional performance considerations:

* Stripe width: matching this to the workload

* Observability: Virtual device utilization can be confusing, check the separate physical devices. 

* CPU overhead: especially when performing RAID parity computation. This has become less of an issue on modern, faster CPUs

* Rebuilding: also called resilvering, this is when an empty disk is added to a RAID group (e.g. replacing a failed disk). Can significantly hurt performance. 

