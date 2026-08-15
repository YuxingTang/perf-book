\phantomsection
# Appendix B. Enable Huge Pages 启用大页 {.unnumbered}

\markboth{Appendix B}{Appendix B}

## Windows {.unnumbered}

To utilize huge pages on Windows, you need to enable `SeLockMemoryPrivilege` [security policy](https://docs.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/lock-pages-in-memory). This can be done programmatically via the Windows API, or alternatively via the security policy GUI.

要在Windows上使用大页，您需要启用“SeLockMemoryPrivilege”[安全策略](https://docs.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/lock-pages-in-memory)。这可以通过 Windows API以编程方式完成，也可以通过安全策略GUI完成。

1. Hit start &rarr; search "secpol.msc", and launch it.
2. On the left select "Local Policies" &rarr; "User Rights Assignment", then double-click on "Lock pages in memory".

1. 点击“开始”菜单 --> 搜索“secpol.msc”，然后启动它。
2. 在左侧选择“本地策略” --> “用户权限分配”，然后双击“锁定内存中的页（Lock pages in memory）”。

![Windows security: Lock pages in memory Windows 安全：锁定内存中的页](../../img/appendix-C/WinLockPages.png){width=100%}

3. Add your user and reboot the machine.
3. 添加您的用户并重启计算机。

4. Check that huge pages are used at runtime with [RAMMap](https://docs.microsoft.com/en-us/sysinternals/downloads/rammap) tool.
4. 使用 [RAMMap](https://docs.microsoft.com/en-us/sysinternals/downloads/rammap) 工具检查运行时是否使用了大页内存。

Use huge pages in the code with:
在代码中使用大页内存：

```cpp
void* p = VirtualAlloc(NULL, size, MEM_RESERVE | 
                                   MEM_COMMIT | 
                                   MEM_LARGE_PAGES,
                       PAGE_READWRITE);
...
VirtualFree(ptr, 0, MEM_RELEASE);
```

## Linux {.unnumbered}

On Linux OS, there are two ways of using huge pages in an application: Explicit and Transparent Huge Pages.

在Linux操作系统中，应用程序使用大页有两种方式：显式大页和透明大页。

### Explicit Huge Pages 显式大页 {.unnumbered .unlisted}

Explicit huge pages can be reserved at system boot time or before an application starts. To make a permanent change to force the Linux kernel to allocate 128 huge pages at the boot time, run the following command:

显式大页可以在系统启动时或应用程序启动之前预留。要永久强制Linux内核在启动时分配128个大页，请运行以下命令：

```bash
$ echo "vm.nr_hugepages = 128" >> /etc/sysctl.conf
```

To explicitly allocate 128 huge pages after the system has booted, you can use the following command:
要在系统启动后显式分配128个大页，可以使用以下命令：

```bash
$ echo 128 > /proc/sys/vm/nr_hugepages
```

You should be able to observe the effect in `/proc/meminfo`. Note that it is a system-wide view and not per-process:
您应该可以在 `/proc/meminfo` 中观察到效果。请注意，这是系统范围的视图，而不是每个进程的视图：

```bash
$ watch -n1 "cat /proc/meminfo  | grep huge -i"
AnonHugePages:      2048 kB
ShmemHugePages:        0 kB
FileHugePages:         0 kB
HugePages_Total:     128    <== 128 huge pages allocated
HugePages_Free:      128
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:          262144 kB <== 256MB of space occupied
```

You can utilize explicit huge pages in the code by calling `mmap` with the `MAP_HUGETLB` flag ([full example](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/map_hugetlb.c)[^25]):

你可以通过调用带有 `MAP_HUGETLB` 标志的 `mmap` 函数在代码中使用显式大页（[完整示例](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/mm/hugetlb_fault_after_madv.c)[^25]）：

```cpp
void ptr = mmap(nullptr, size, PROT_READ | PROT_WRITE,
                MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0);
...
munmap(ptr, size);
```

Other alternatives include:
其他替代方案包括：

* `mmap` using a file from a mounted `hugetlbfs` filesystem ([example code](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/hugepage-mmap.c)[^26]).
* `shmget` using the `SHM_HUGETLB` flag ([example code](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/hugepage-shm.c)[^27]).

* 使用已挂载的 `hugetlbfs` 文件系统中的文件执行 `mmap` 函数（[示例代码](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/mm/hugetlb-mmap.c)[^26]）。
* 使用 `SHM_HUGETLB` 标志执行 `shmget` 函数（[示例代码](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/mm/hugetlb-shm.c)[^27]）。

### Transparent Huge Pages {.unnumbered .unlisted}

To allow applications to use Transparent Huge Pages (THP) on Linux you should ensure that `/sys/kernel/mm/transparent_hugepage/enabled` is `always` or `madvise`. The former enables system-wide usage of THPs, while the latter gives control to the user code on which memory regions should use THPs, thus avoiding the risk of consuming more memory resources. Below is an example of using the `madvise` approach:

要允许应用程序在Linux上使用透明大页(THP: Transparent Huge Pages)，你应该确保 `/sys/kernel/mm/transparent_hugepage/enabled` 设置为 `always` 或 `madvise`。前者允许系统范围内使用THP，而后者则允许用户代码控制哪些内存区域应该使用THP，从而避免消耗更多内存资源的风险。以下是使用 `madvise` 方法的示例：

```cpp
void ptr = mmap(nullptr, size, PROT_READ | PROT_WRITE | PROT_EXEC,
                MAP_PRIVATE | MAP_ANONYMOUS, -1 , 0);
madvise(ptr, size, MADV_HUGEPAGE);
...
munmap(ptr, size);
```

You can observe the system-wide effect in `/proc/meminfo` under `AnonHugePages`:
你可以在 `/proc/meminfo` 的 `AnonHugePages` 下观察到系统范围的效果：

```bash
$ watch -n1 "cat /proc/meminfo  | grep huge -i" 
AnonHugePages:     61440 kB     <== 30 transparent huge pages are in use
HugePages_Total:     128
HugePages_Free:      128        <== explicit huge pages are not used
```

Also, you can observe how your application utilizes EHPs and/or THPs by looking at the `smaps` file specific to your process:

通，你还可以通过查看进程特定的 `smaps` 文件来观察应用程序如何使用EHP和/或THP：

```bash
$ watch -n1 "cat /proc/<PID_OF_PROCESS>/smaps"
```

[^25]: MAP_HUGETLB example - [https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/map_hugetlb.c](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/map_hugetlb.c).
[^26]: Mounted `hugetlbfs` filesystem - [https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/hugepage-mmap.c](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/hugepage-mmap.c). （译者注：在2026年的Linux内核代码上相关文件名和目录组织已经改变）挂载 `hugetlbfs` 文件系统 - [https://github.com/torvalds/linux/blob/master/tools/testing/selftests/mm/hugetlb-mmap.c](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/mm/hugetlb-mmap.c)。
[^27]: SHM_HUGETLB example - [https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/hugepage-shm.c](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/hugepage-shm.c). 译者注：在2026年的Linux内核代码上相关文件名和目录组织已经改变）SHM_HUGETLB示例 - [https://github.com/torvalds/linux/blob/master/tools/testing/selftests/mm/hugetlb-shm.c](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/mm/hugetlb-shm.c)
