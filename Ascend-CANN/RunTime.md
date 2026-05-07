
CUDA：Compute Unified Device Architecture 统一计算设备架构
CANN：Compute Architecture for Neural Networks 神经网络的计算架构

# 1、RunTime
RunTime 运行时管理器， 引用《深入理解CANN技术原理与应用》中对runtime的定义：作为神经网络软件任务流向系统硬件资源的大坝系统闸门，运行管理器专门为神经网络的任务分配提供了资源管理通道。
昇腾AI 芯片通过运行管理器而执行在应用程序的进程空间中，为应用程序提供了存储（Memory）管理、设备（Device）管理、执行流（Stream）管理、事件（Event）管理、核函数（Kernel）执行等功能。其系统模型如下图所示。

Runtime 实际上是一个.so 文件，即 libruntime.so，在线模式下，host侧加载 libruntime.so，离线模式下 device侧来加载libruntime.so

- OM API ： ops manage，维测模块，主要提供Log、Profiling、Data Dump等
- Device API：设备管理，主要对上层APP提供调用接口
- Context API: Context管理，指定后续业务运行的上下文，起到资源隔离的作用，方便控制影响范围
- Stream API：流管理，Stream在《CUDA Programming Guid》中的定义是一些列命令的集合，此处，则是一系列task的集合。Stream由上层模块GE拆分而来且Stream内的task是串行保序执行，Stream间是并发执行。少量存在依赖关系的Stream间需要同步。单个Device支持最大Stream的数量为1024个
- Event API：事件管理，event是不同stream间进行同步时触发的。适用于Stream间的等待
- Memory API：内存管理，用于分配和释放Device侧的内存，支持两种内存模型：
    -- 类CUDA4：Host与Device地址空间分离，需要显式拷贝共享数据；
    -- 类CUDA6：Host与Device是统一的地址空间。可直接传递指针共享数据
- Model API： 模型管理，下沉task时调用的，在model与stream绑定后，stream内的task即为下沉task，通过将task下沉到Task Scheduler侧，在需要时再触发task调度，提升模型执行效率
- Kernel API：Kernel管理，对Kernel算子的计算处理等
