> **Note to the reader**: I learned about the memory profiling tools mentioned in this document at the cost of my own memory. So, please let me know if you spot any errors and I will fix them. 

# 🧠 Memory Profiling Tools

This document compares various tools used to measure and analyze memory usage in Linux-based systems, especially for C/C++ applications. It focuses on how each tool handles different memory components and what kind of insights they provide.

---
## 📘 Definitions

- **Heap allocations**: Memory dynamically allocated during runtime using `malloc`, `calloc`, `new`, etc.

- **Stack memory**: Memory used for function calls, local variables, and control flow. Automatically managed.

- **Shared libraries**: Memory used by dynamically linked libraries (`.so` files) loaded into the process.
- **Resident Set Size (RSS)** : 
  - The resident set size (RSS) is the amount of space of physical memory (RAM) held by a process.
  - The peak resident set size (Peak RSS or Max RSS) refers to the peak amount of memory a process has had up to that point. 
  - This includes memory allocated from shared libraries, given they are still present in memory. Also, it includes all heap and stack memory.
  - RSS is not an accurate measure of the total memory processes are consuming, because it does not include memory consumed by libraries that were swapped out. On the other hand, the same shared libraries may be duplicated and counted in different processes.
  Shared libraries are nonetheless counted as part of RSS because they are useful for code execution. 
  - Regardless, RSS is a reliable estimate.

- **Virtual Memory (VM)** :
  - VM is the total address space allocated to a process.
  - When a program is run, it is not completely loaded on to the RAM. Instead, some part resides in swap files on the disk. 
  - Virtual memory is the sum total of allocated physical memory (i.e., on the RAM. This is also called RSS) and the unallocated memory on the disk.

### 📚 Good References:
- [Virtual Memory vs. Resident Set Size](https://docs.hpc.qmul.ac.uk/using/memory/)
- [Virtual Memory - Wikipedia](https://en.wikipedia.org/wiki/Virtual_memory)
- [Virtual Memory in Operating System](https://www.geeksforgeeks.org/operating-systems/virtual-memory-in-operating-system/)
- [Stack Overflow: VmRSS vs RSS](https://stackoverflow.com/questions/10400751/how-do-vmrss-and-resident-set-size-match)
- [Tudor Brindus: Kernel lies about memory](https://tbrindus.ca/sometimes-the-kernel-lies-about-process-memory-usage/)
- [Red Hat: Understanding memory usage](https://access.redhat.com/articles/1618)

---

## Memory Usage Tracking Techniques

### 1. Using Memory Profilers
> Memory profilers are useful for an in-depth memory usage analysys for the entire program.
- [Valgrind Massif](https://valgrind.org/docs/manual/ms-manual.html),
- [Heaptrack](https://github.com/KDE/heaptrack) 
- [gperftools](https://github.com/gperftools/gperftools)

### 2. Using `getrusage()`
> 1. Retrieves the maximum resident set size (RSS) of the calling process at the time of invocation.  
> 1. Can also report the maximum user CPU time and system CPU time consumed.  
> 1. Provides additional resource usage metrics — see the [manual page](https://man7.org/linux/man-pages/man2/getrusage.2.html) for full details.

#### 📊 Key Metrics in `getrusage()`
| Metric       | Description                          | Type     |
|--------------|--------------------------------------|----------|
| `ru_maxrss`  | Peak physical memory usage           | Physical Memory |
| `ru_utime`   | Time spent executing user code       | CPU Time |

#### Application
- To get max RSS at a particular instant of the code, you can use do the following:
  ``` C
  #include <sys/resources/h> // You need to include this.
  long get_rss_max()
  {
    struct rusage usage;
    int err = getrusage(RUSAGE_SELF,&usage);  
    long rss_max = usage.ru_maxrss; //rss_max of the calling process

    #ifdef MPI
      // If you want total rss_max used by all the processors, then use this 
      long local_rss_max = rss_max;
      long global_rss_max;
      err = MPI_Allreduce(&(local_rss_max), &(global_rss_max), 1, MPI_LONG, MPI_SUM, MPI_COMM_WORLD);
      if (err == MPI_SUCCESS)
        rss_max = global_rss_max;
    #endif
    return rss_max;
  }
  
  int main()
  {
    <----- your code -------->

    long rss_max = get_rss_max();
    std::cout << " Maximum RSS usage by the program:" << rss_max <<"KB" <<std::endl;
    return 0;
  }
  ```
- You can also output elapsed user CPU time by using `usage.timeval ru_stime` instead.  

- Another amazing use for `getrusage()` is that you can check the amount of memory used by a certain code block. 
  - This is useful for when you want to track the memory added by a particular code block. 
  - If the code block contains `malloc` calls or any "Create/Allocate" functions used by libraries, this can be used to track the amount of memory added by the calls.  
  ```C
  #include <sys/resources/h> 

  int main()
  {
    long before = get_rss_max();
    <----- your code -------->
    long after = get_rss_max();
    
    std::cout << " RSS spike due to the code block is:" << after-before <<"KB" <<std::endl;
    return 0;
  }

  ```

### 3. Using `/proc/self/status`
> 1. Provides access to various memory usage statistics for the calling process.  
> 1. Includes details on virtual memory segments and resident set size (RSS).  
> 1. Useful for lightweight, file-based introspection without requiring external tools or libraries.
> 1. It is possible that the information is inaccurate.
> 1. For more information, please consult the [linux manual page](https://man7.org/linux/man-pages/man5/proc_pid_status.5.html).

#### 📊 Key Memory Metrics in `/proc/self/status`



| Field      | Description                                                                 | What It Includes                                                                 | Type  of Memory    |
|------------|-----------------------------------------------------------------------------|----------------------------------------------------------------------------------|-----------|
| **VmSize** | Total virtual memory size                                                   | All mapped memory: heap, stack, shared libs, mmap, etc.                         | Virtual   |
| **VmPeak** | Peak virtual memory size                                                    | Highest `VmSize` reached during process lifetime                                | Virtual   |
| **VmData** | Size of the data segment                                                    | Heap, global/static variables, some mmap regions                                | Virtual   |
| **VmHWM**  | High Water Mark (peak resident set size)                                    | Peak RAM usage during process lifetime                                          | Physical  |
| **VmRSS**  | Current resident set size                                                   | Memory currently resident in RAM                                                | Physical  |
| **VmPTE**  | Page table entries size                                                     | Memory used by page tables                                                      | Virtual   |
| **VmLib**  | Shared library code size                                                    | Memory mapped for shared libraries                                              | Virtual   |
| **VmExe**  | Executable code segment size                                                | Memory used by the main executable’s code                                       | Virtual   |
| **VmStk**  | Stack segment size                                                          | Size of the process stack                                                       | Virtual   |



#### Application
- `/proc/self/status` can be used as follows:
  ``` C
  long get_VmData()
  {

    FILE *fp = fopen("/proc/self/status", "r");
    char line[256];
    long vmdat;
    while (fgets(line, sizeof(line), fp))
    {
        if (strncmp(line, "VmData:", 7) == 0) 
        // Number of Characters from "V" to ":" (inclusive) are 7
        // Hence we use strncmp(line, "VmDat:", 7) == 0
        {
            sscanf(line + 7, "%ld", &vmdat);
        }
    }
    fclose(fp);

    #ifdef MPI // You can disable this if you want info per processor
      // If you want total rss_max used by all the processors, then use this 
      long local_vmdat = vmdat;
      long global_vmdat;
      int err = MPI_Allreduce(&(local_vmdat), &(global_vmdat), 1, MPI_LONG, MPI_SUM, MPI_COMM_WORLD);
      if (err == MPI_SUCCESS)
        vmdat = global_vmdat;
    #endif
    return vmdat;
  }

  int main()
  {

    <----- your code block ------->
    long vmdat = get_VmData();
     std::cout << " Total virtual memory usage by the program/process at this instant is:" << vmdate <<"KB" <<std::endl;

    return 0;
  }
  ```
- One can write similar code to output `VmRSS`, `VmHWM`, etc.
- Similar to `getrusage()`, we can use `/proc/self/status` to get the spike in `VmData` caused by a code block as follows:
  ```C

  int main()
  {
    long before = get_VmData();
    <----- your code -------->
    long after = get_VmData();
    
    std::cout << " Increase in virtual memory usage due to the code block is :" << after-before <<"KB" <<std::endl;
    return 0;

  }
  ```
### 4. Using `/proc/self/smaps_rollup`
> 1. Similar to `/proc/self/status` but more accurate and time-consuming. 
> 1. `smaps_rollup` as its name suggests is [`/proc/self/smaps`](https://man7.org/linux/man-pages/man5/proc_pid_smaps.5.html) rolled up into an aggregate file for the entire process.
> 1. Please consult its corresponding [manual](https://www.kernel.org/doc/Documentation/ABI/testing/procfs-smaps_rollup) for more information.
> 1. Comprehensive information could be found [here](https://www.kernel.org/doc/Documentation/filesystems/proc.rst). It is best to go over the entire page as it containts pretty much everything in `/proc/` filesystem.


#### 🧠 Key memory metrics in `smaps_rollup`

| Metric              | Definition |
|---------------------|------------|
| Size                 | The total virtual memory size of all memory mappings for a process, summed together and reported in kilobytes (kB) |
| Rss                 | Amount of the mapping currently resident in RAM (resident set size). |
| Shared_Clean        | Total amount of memory that is shared between processes and unmodified (clean). These pages are typically backed by files and can be easily reclaimed. |
| Shared_Dirty        | Total amount of shared memory that has been modified (dirty). These pages may need to be written back to disk before being reclaimed. |
| Private_Clean       | Total amount of private memory (used only by this process) that is unmodified. These are clean pages that can be reclaimed without data loss. |
| Private_Dirty       | Total amount of private memory that has been modified. These pages contain changes specific to this process and must be preserved or written back before being reclaimed. |

#### Application
- `/proc/self/smaps_rollup` can be used in a way similar to `/proc/self/status`.
  ``` C
  long get_SmapsSize() 
  {
    // Get the Total "Size" used by the process (see above table)
    FILE *fp = fopen("/proc/self/smaps_rollup", "r");
    if (!fp) {
        perror("Failed to open /proc/self/smaps_rollup");
        return -1;
    }

    char line[256];
    long size_kb = 0;

    while (fgets(line, sizeof(line), fp)) {
        if (strncmp(line, "Size:", 5) == 0) 
        //If you want another field, say Rss, use it as:
        //strncmp(line, "Rss:", 4) == 0 AND,
        //Change 5 to 4 in sscanf() line below
        {
            sscanf(line + 5, "%ld", &size_kb);
            break;
        }
    }

    fclose(fp);

    #ifdef MPI
      long local_size = size_kb;
      long global_size;
      int err = MPI_Allreduce(&local_size, &global_size, 1, MPI_LONG, MPI_SUM, MPI_COMM_WORLD);
      if (err == MPI_SUCCESS) 
          size_kb = global_size;
      
    #endif

    return size_kb;
  }

  ```
- Similar to [`/proc/self/status`](#3-using-procselfstatus), the above function can be used to either get instantaneous virtual memory size or called before and after a code block to get the virtual memory spike due to the code block. 

### 4. Using `memusage()`
- Coming Soon
### 5. Using `mtrace()`
- Coming Soon
---

