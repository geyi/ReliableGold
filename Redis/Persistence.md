# Redis RDB
Linux中的父子进程，父进程中的数据，子进程能不能看到？
- 常规思想，数据在进程之间是隔离的
- 进阶思想，父进程可以让子进程看到数据

在Linux中，export的环境变量，子进程的修改不会破坏父进程，父进程的修改也不会破坏子进程。（子进程中拥有环境变量的一个拷贝）

如果父进程是Redis，内存数据10GB
- 创建子进程的速度？
- 内存空间够不够？

fork系统调用，创建子线程速度快，需要的内存空间小。

copy on write，创建子进程时并不发生复制，使得创建进程变快了。根据经验，不可能父子进程把所有数据都改一遍。父进程修改数据时，就是父进程触发中断，结果就是父进程里面的地址空间被改变了。
```C
#include <stdio.h>
#include <unistd.h>
#include <stdint.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <stdlib.h>

#define SIZE 1024

// 将虚拟地址转换为物理地址
unsigned long virt_to_phys(void *addr) {
    int fd = open("/proc/self/pagemap", O_RDONLY);
    if (fd < 0) return 0;

    unsigned long vaddr = (unsigned long)addr;
    unsigned long offset = (vaddr / sysconf(_SC_PAGESIZE)) * sizeof(uint64_t);
    
    uint64_t entry;
    lseek(fd, offset, SEEK_SET);
    read(fd, &entry, sizeof(entry));
    close(fd);

    if (!(entry & (1ULL << 63))) return 0;  // 检查页面是否在内存中
    
    unsigned long frame = entry & ((1ULL << 55) - 1);  // 获取页帧号
    return (frame * sysconf(_SC_PAGESIZE)) + (vaddr % sysconf(_SC_PAGESIZE));
}

int main() {
    int *shared_array = malloc(SIZE * sizeof(int));
    printf("数组初始虚拟地址: %p\n", shared_array);
    printf("数组初始物理地址: 0x%lx\n", virt_to_phys(shared_array));

    pid_t pid = fork();

    if (pid == 0) {
        // 子进程修改数组中间元素
        printf("子进程修改前: [0]=%p, 物理=0x%lx\n", 
               &shared_array[512], virt_to_phys(&shared_array[512]));
        shared_array[512] = 42;
        printf("子进程修改后: [0]=%p, 物理=0x%lx\n", 
               &shared_array[512], virt_to_phys(&shared_array[512]));
    } else {
        sleep(1);
        // 父进程修改数组开头元素
        printf("父进程修改前: [0]=%p, 物理=0x%lx\n", 
               &shared_array[0], virt_to_phys(&shared_array[0]));
        shared_array[0] = 100;
        printf("父进程修改后: [0]=%p, 物理=0x%lx\n", 
               &shared_array[0], virt_to_phys(&shared_array[0]));
    }

    free(shared_array);
    return 0;
}
```

综上所述：Redis父进程通过fork系统调用创建了一个子进程，由子进程对内存中的数据进行落盘。（时点性，point-in-time）

> save：阻塞。应用场景：关机维护时。
> 
> bgsave：fork创建子进程。应用场景：仍然需要对外提供服务时。

缺点：
1. 可能丢失更多的数据

优点：
1. 单个的紧凑的文件，更适合传输和灾难恢复
2. 大数据集下相比AOF，Redis服务能更快的重启。即，恢复速度相对快（序列化反序列化）
3. 最大限度的提高了Redis的性能，因为RDB的持久化操作由fork出的子进程完成

# Redis AOF
Redis的写操作记录到文件中

如果开启了AOF，恢复时只会用AOF

优点：
1. 丢失数据少
2. AOF文件不容易损坏
3. 即使执行了FLUSHALL命令，只要还没有执行过重写操作，仍然可以通过停止服务器、删除最新命令并重新启动Redis来恢复数据

弊端：
1. 体量无限大，恢复慢

## 重写
bgrewriteaof：异步的重写AOF文件。重写时会用创建当前数据集所需的最小操作集生成一个全新的文件，一旦第二个文件准备好，Redis就会切换到新的文件并开始追加到新的记录

4.0之前
- 删除抵消的命令
- 合并重复的命令

最终也是一个纯指令的日志文件

4.0之后
- 将老的数据RDB到AOF文件中，将增量的以指令的的方式append到AOF中。(https://raw.githubusercontent.com/antirez/redis/4.0/00-RELEASENOTES)

记录指令时触发IO的三种策略
- NO
- ALWAYS
- everysec