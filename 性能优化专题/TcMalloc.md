# [1] Tcmalloc总体架构

https://3ms.huawei.com/km/groups/3959050/blogs/details/16645964

Tcmaloloc是 Google 开源的一款高效的内存分配器。它主要通过减少锁竞争和提高缓存效率来提高多线程程序的性能。TCMalloc 采用了线程本地缓存机制，使得每个线程在分配内存时尽可能避免与其他线程的竞争，从而减少了锁的使用，提升了性能。

### TCMalloc典型使用场景

-   **多线程应用**：TCMalloc 特别适合高并发、多线程的应用程序，如Web服务器、数据库系统、分布式计算等。
-   **频繁的小对象分配**：如高频率地进行 `malloc` 和 `free` 操作的程序，由于它能显著提升小块内存的分配效率。
-   **对性能要求苛刻的应用**：TCMalloc 的高效内存管理机制能够提升程序的整体性能，因此适合性能敏感的系统。

![img](/2025H2/Tcmalloc.assets/6e6aa8a720e785042667ea44958a2fac_1101x520.png@900-0-90-f.png)

![img](/2025H2/Tcmalloc.assets/b17493b40f0b0ed2dc87491caa030000_966x812.png@900-0-90-f.png)

# [2] ptmalloc、tcmalloc与jemalloc对比分析

https://3ms.huawei.com/km/blogs/details/17974920

https://cloud.tencent.com/developer/article/2390348?policyId=1004

https://3ms.huawei.com/km/blogs/details/14877368

# [3] 图解 TCMalloc

https://cloud.tencent.com/developer/article/2064631
