+++
title = 'Is it safe to run postgres on kubernetes?'
date = 2023-10-26T00:09:29+05:30
draft = false
+++


Recently came across a issue while restoring a backup of a postgres database running on a kubernetes cluster that the restoration failed due to a bus error/SIGBUS.
```
LOG:  server process (PID 497) was terminated by signal 7: Bus error
```
Bus error is a signal raised by hardware, when a process tries to access a part of memory that the CPU cannot address physically. This was the first time I came across a bus error in kubernetes and was curious to see what was causing it. So the postgres pod/container is trying to access memory which doesn't exist(or it's not accessible by the container 😉). First thing to look was for resource limits/requests in pod spec and check if postgres was trying to access/allocate more than the limit. 

```
resources:
  limits:
    cpu: "2"
    hugepages-2Mi: 128Mi
    memory: 4096M
  requests:
    cpu: "2"
    hugepages-2Mi: 128Mi
    memory: 4096M
```
That seems fine and postgres was not trying to access more than 4GB of memory. After a quick search on google found that it could be [related to hugepages](https://github.com/kubernetes/kubernetes/issues/71233). The solution mentined in the issue was to turn off hugepages, which is not ideal because enabling hugepages can provide better performance for databases because you need to load less no of pages into memory.

### Quick detour
What is hugepage and why is it used in postgres?

In Linux, Virtual memory and paging work together to manage memory efficiently and provide an abstraction layer between actual physical memory and the user space. Virtual memory address space is divided into fixed-size units called pages and the physical memory is divided into fixed-size units called frames. usually both are of the same size(4KB is default page size on x86 systems). Page table stores the mapping between virtual and physical memory. When a CPU access memory using virtual address it needs to be translated to physical address, which involves looking up the page table. Accessing page tablescan be slow as page table resides in the main memory(RAM). To speed up the lookup/address translation, CPUs uses a TLB (Translation Lookaside Buffer) which is a small cache of recently used page table entries. TLBs are faster than page tables because they are smaller and specialized cache built within CPUs. When a CPU needs to translate a virtual address to a physical address, it first checks if the virtual address is in the TLB. If the translation is found in the TLB("TLB hit"), it just returns the physical address. If the translation is not found in the TLB("TLB miss"), it needs to access the page table to perform a translation, which is called a "page table walk" which is costly and time consuming.


As the amount of physical memory increases(because of the clouds), number of TLB entries required to manage the memory increases to provide better performance. So, to solve this either you can increase the the size of TLB(not ideal) or increase the size of the pages from 4KB(hugepages).

Hugepages address this by using larger contagious memory blocks(eg 2MB or 1GB on x86 systems), so now a single TLB entry can map large memory space.By using hugepages, with the same TLB size more memory can managed with fewer TLB misses.


When memory is not tuned properly, databases try to use up all the memory on the node and crash due to OOM(Out Of Memory) killer. Infact this is one of the top reason for most of the postgreSQL database crashes reported to percona, and they have explained the reason and need of hugepages in this [article](https://www.percona.com/blog/why-linux-hugepages-are-super-important-for-database-servers-a-case-with-postgresql/).

> TLDR: 
    Page table gets bloated because it's trying to cache too many translations of smaller pages and starts consuming all the memory including swap. Solution to this is to enable hugepages and tune no of hugepages based on heuristics. This reduces the memory consumption by a big margin and the background processes won't be targeted by OOM Killer.

## How to confirm if it was due to hugepages

I was still not sure if the SIGBUS was because of hugepages in the postgres pod/container. To confim this I had a simple plan

- Allocate 128Mi/2Mi = 64 hugepages inside the pod and see if it works fine without error.
- Allocate 100 hugepages(which is more than the configured limit set on pod spec) - it should throw error right? It didn't 🧐
- Allocate 100 hugepages and try to write to the allocated memory

Wrote simple C programs to do this:


``` go
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/mman.h>

#define PAGE_SIZE 2097152 // 2 MiB (size of a huge page)

int main() {
    void *addr = NULL;

    size_t num_pages = 100; // Number of hugepages

    addr = mmap(NULL, num_pages * PAGE_SIZE, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0);

    if (addr == MAP_FAILED) {
        perror("Memory allocation in hugepages failed");
        return 1;
    }

    printf("Memory allocated in hugepages successfully!\n");


    // Unmap the memory when you're done with it
    if (munmap(addr, num_pages * PAGE_SIZE) == -1) {
        perror("munmap failed");
        return 1;
    }

    return 0;
}
```
alloc.c


This program just allocates the no of hugepages specified and prints whether the allocation was successful or not. When I ran this to allocate 64 pages and 100 pages it successfully allocated for both the runs. I was surprised how it could allocate more than 64 pages(that's the resource limit we have set). Upon checking the no of available hugepages on the pod I found out it has the same no of pages as the host node and not the limit configured on pod spec.

```
$ grep -i hugepage /proc/meminfo
HugePages_Total:      500
HugePages_Free:       460
HugePages_Rsvd:        40
HugePages_Surp:        0
Hugepagesize:       2048 kB
```

That's not good, so a pod can use up all the hugepages on the host node? Not actually. I wrote another C program which allocates memory and tries to write something to the allocated space.

``` go
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/mman.h>
#include <fcntl.h>
#include <errno.h>

#define NUM_PAGES 65
#define PAGE_SIZE 2097152 // 2 MiB (size of a huge page)
#define FILE_NAME "/dev/hugepages/hugepagefile"


static void write_bytes(char *addr)
{
    unsigned long i;
    unsigned long length = NUM_PAGES*PAGE_SIZE;
    for (i = 0; i < length; i++)
        *(addr + i) = (char)i;
}


int main() {
    void *addr = NULL;
    char buf[32];
    int fd;

    sprintf(buf, "%s", FILE_NAME);
    fd = open(buf, O_CREAT | O_RDWR, 0755);
    size_t num_pages = NUM_PAGES;

    addr = mmap(NULL, num_pages * PAGE_SIZE, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0);

    if (addr == MAP_FAILED) {
        perror("Memory allocation in hugepages failed");
        return 1;
    }

    else{
        // when memory is allocated write to that space
        write_bytes(addr);
    }

    printf("Memory allocated in hugepages successfully!\n");

    // Unmap the memory when you're done with it
    if (munmap(addr, num_pages * PAGE_SIZE) == -1) {
        perror("munmap failed");
        return 1;
    }

    return 0;
}
```
writemem.c



![SIGBUS](/img/sigbus.png)

Now running this on the pod triggered a SIGBUS/Bus error 🎉. So a pod can allocate as many hugepages as the host node has but can't use all the allocated memory because of the limits configured in the pod spec. That explains why we got a Bus error while restoring the backup, pod tried to allocate more than 64 hugepages and it was successful. Postgres process tried to use that allocated memory but the pod physically can't address the memory outside the configured limits and threw a Bus error. That sounds really bad, when there's a SIGBUS the process is terminated(as we saw in the original log), so all postgres server processes are terminated and this triggers a auto-recovery which tries to start the postgres server and there are high chances that it might fail again because of bus error. This can potentially corrupt the database 💀.


## How to overcome this and run postgres on k8s then?

Let me remind you, yu can configure no of hugepages via sysctl (`vm.nr_hugepages`). So can't we just set no of hugepages on a pod and call it a day? You can't configure or change all sysctl values in a pod, you should enable the sysctl on kubelet via a kubelet arg `--allowed-unsafe-sysctls=<sysctl_name>`. But even with this you can't configure `vm.nr_hugepages` sysctl on pod level because it's not namespaced. What this means is it can't be set independently for each pod on a node and being namespaced is a requirement for sysctl to be accessible in a pod context. You might see something similar to this if it's not namespaced.

```
FATA[0003] Error: failed to run Kubelet: failed to create kubelet: the sysctl "vm.nr_hugepages" are not known to be namespaced
```

## Conclusion

So, is it safe to run postgres databases on kubernetes clusters?

If you are just running 1 postgres database on a cluster, you can schedule the postgres pods on particluar nodes using taints and tolerations where you have enabled hugepages and it can use all the hugepages of the underlying node. But if you want to run multiple databases on a cluster then postgres won't play nice with k8s, it's not very reliable and you might face these issues. And obviously you can always go back and disable hugepages and run multiple databases on a cluster, you'll loose all the performance advantages provided by hugepages and the OOM Killer can crash the postgres processes as explained in the article by Percona.


### References

1. [More about page tables in linux](https://lwn.net/Articles/717293/)
2. [LWN article for hugepages](https://lwn.net/Articles/374424/)
3. [Configuring hugepages with crunchydata PGO](https://access.crunchydata.com/documentation/postgres-operator/latest/guides/huge-pages)