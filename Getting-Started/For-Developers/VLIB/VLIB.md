## VLIB(矢量处理库)
与vlib关联的文件位于./src/{vlib，vlibapi，vlibmemory}文件夹中。这些库提供矢量处理的支持，包括图节点调度（graph
-node-scheduling），可靠的多播支持（reliable multicast support）、超轻量级协作多任务线程（ultra-lightweight cooperative multi-tasking threads）、CLI、插件.DLL、物理内存和Linux epoll。该库的部分实现是美国专利7,961,636的体现。

### 初始化函数发现
vlib应用程序通过将结构和__attribute __（（constructor））函数放入镜像中来注册各种[initialization]事件。在适当的时候，vlib框架将遍历构造函数生成的单链接结构列表，根据指定的约束执行拓扑排序，并调用所指定的函数。Vlib应用程序使用此机制创建图节点，添加CLI功能，启动协作式多任务线程等。

vlib应用程序始终包含许多VLIB_INIT_FUNCTION（my_init_function）宏。

每个init/configure/等函数的返回类型均为clib_error_t *。如果一切正常，请确保函数返回0，否则框架将声明一个错误并退出。

vlib应用程序必须链接vppinfra，并且经常链接到其他库，例如VNET。在后一种情况下，可能有必要显式地引用符号（symbols），否则库的大部分在运行时可能是AWOL（擅离职守）。

#### 初始化函数构造和约束规范
添加一个初始化函数非常容易：

```
   static clib_error_t *my_init_function (vlib_main_t *vm)
   {
      /* ... initialize things ... */

      return 0; // or return clib_error_return (0, "BROKEN!");
   }
   VLIB_INIT_FUNCTION(my_init_function);
```

如上面给定的那样，my_init_function将“在某个时候”执行，但是没有顺序保证。

指定顺序限制也是很容易的：

```
   VLIB_INIT_FUNCTION(my_init_function) =
   {
      .runs_before = VLIB_INITS("we_run_before_function_1",
                                "we_run_before_function_2"),
      .runs_after = VLIB_INITS("we_run_after_function_1",
                               "we_run_after_function_2),
    };
```

也可以轻松的指定批量顺序限制，如“先a后b后c后d”：

```
   VLIB_INIT_FUNCTION(my_init_function) =
   {
      .init_order = VLIB_INITS("a", "b", "c", "d"),
   };
```

可以为单个init函数指定所有三种排序约束，尽管很难想象为什么会有必要。

### 节点图初始化
vlib数据包处理程序总是定义一组图节点来处理数据包。

通常通过VLIB_REGISTER_NODE宏来构造vlib_node_registration_t。在运行时，框架将此类注册集处理为有向图（directed graph）。在运行时将节点添加到图很容易，但该框架不支持删除节点。

vlib提供了几种类型的矢量处理图节点，主要用于控制框架的调度行为。vlib_node_registration_t的类型成员的功能如下：

* VLIB_NODE_TYPE_PRE_INPUT-在所有其他节点类型之前运行
* VLIB_NODE_TYPE_INPUT-在pre_input节点之后尽可能频繁地运行
* VLIB_NODE_TYPE_INTERNAL-仅在显式的使其可运行（runable）时，通过添加待处理帧。
* VLIB_NODE_TYPE_PROCESS-仅在显式的使其可运行时。“进程”节点实际上是协作的多任务线程。他们必须在相当短的时间内显式的挂起。

为了更好地理解图节点调度程序，请阅读./src/vlib/main.c:vlib_main_loop。

### 图节点调度器
Vlib_main_loop（）调度图节点。基本的矢量处理算法非常简单，但即使长时间盯着代码看也可能不太明显。它是这样工作的：某些输入节点或一组输入节点产生一个要处理的工作向量。图节点调度程序将工作向量推过有向图，并根据需要对其进行细分，直到原始工作向量已被完全处理为止。在这一点上，该过程重复进行。

通过构造，该方案在框架尺寸上产生稳定的平衡。原因如下：随着帧尺寸的增加，每帧元素的处理时间会减少。有几种相关的力量在起作用；最简单的描述是向量处理对CPU L1 I-cache的影响。给定节点处理的第一个帧元素[packet]预热了L1 I-cache中的节点分发功能。所有后续的框架元素[packet]都会获利。随着我们增加框架元素[packet]的数量，每个元素的成本下降。

在轻负载下，运行图节点分派器完全是CPU周期的疯狂浪费。因此，如果流过的帧大小很低，则图节点调度程序将安排在定时的epoll等待中等待工作。该方案具有一定量的滞后，以避免在中断和轮询模式之间不断地来回切换。尽管图调度程序支持中断和轮询模式，但我们当前的默认设备驱动程序不支持。

图节点调度程序使用分层计时器轮（hierarchical timer wheel）在计时器到期时重新调度过程节点。

### 图调度器内部
可以安全地跳过此部分。无需了解图调度程序的内部知识即可创建图节点。

### 矢量数据结构
在vpp/lib中，我们将向量表示为vlib_frame_t类型的实例：

```
    typedef struct vlib_frame_t
    {
      /* Frame flags. */
      u16 flags;

      /* Number of scalar bytes in arguments. */
      u8 scalar_size;

      /* Number of bytes per vector argument. */
      u8 vector_size;

      /* Number of vector elements currently in frame. */
      u16 n_vectors;

      /* Scalar and vector arguments to next node. */
      u8 arguments[0];
    } vlib_frame_t;
```

请注意，可以使用此结构构造所有类型的向量-包括具有一些相关标量数据的向量。在vpp应用程序中，向量通常使用4字节的向量元素大小，并使用零字节表示关联的每帧标量数据。

帧总是在CLIB_CACHE_LINE_BYTES边界上分配。帧具有利用对齐属性的u32索引，因此，帧的最大可行主堆偏移量为CLIB_CACHE_LINE_BYTES * 0xFFFFFFFF：64 * 4 = 256 GB。

### 调度矢量
如您所见，向量并不直接与图节点关联。我们以两种方式表示该关联。最简单的是vlib_pending_frame_t：

```
    /* A frame pending dispatch by main loop. */
    typedef struct
    {
      /* Node and runtime for this frame. */
      u32 node_runtime_index;

      /* Frame index (in the heap). */
      u32 frame_index;

      /* Start of next frames for this node. */
      u32 next_frame_index;

      /* Special value for next_frame_index when there is no next frame. */
    #define VLIB_PENDING_FRAME_NO_NEXT_FRAME ((u32) ~0)
    } vlib_pending_frame_t;
```

处理帧的代码在../src/vlib/main.c:vlib_main_or_worker_loop()。
```
      /*
       * Input nodes may have added work to the pending vector.
       * Process pending vector until there is nothing left.
       * All pending vectors will be processed from input -> output.
       */
      for (i = 0; i < _vec_len (nm->pending_frames); i++)
	    cpu_time_now = dispatch_pending_node (vm, i, cpu_time_now);
      /* Reset pending vector for next iteration. */
```

待处理的帧node_runtime_index将帧与将处理该帧的节点相关联。

### 并发症

系好安全带。这里的故事和数据结构变得非常复杂...

在100,000英尺处：vpp使用有向图，而不是有向无环图。数据包多次访问ip[46]-lookup是非常正常的。最坏的情况：一个图节点将数据包入队其自身。

为了解决此问题，如果当前图节点的发生将数据包入队其自身的情况，则图分配器必须强制分配新帧。

无法保证立即处理待处理的帧，这意味着在将vlib_frame_t附加到vlib_pending_frame_t之后，可能会将更多数据包添加到基础vlib_frame_t中。如果填充了（pending_frame，frame）对，则必须小心分配新帧和待处理帧。

### 下一帧和下一帧的所有权
vlib_next_frame_t是最后一个关键图调度器数据结构：

```
    typedef struct
    {
      /* Frame index. */
      u32 frame_index;

      /* Node runtime for this next. */
      u32 node_runtime_index;

      /* Next frame flags. */
      u32 flags;

      /* Reflects node frame-used flag for this next. */
    #define VLIB_FRAME_NO_FREE_AFTER_DISPATCH \
      VLIB_NODE_FLAG_FRAME_NO_FREE_AFTER_DISPATCH

      /* This next frame owns enqueue to node
         corresponding to node_runtime_index. */
    #define VLIB_FRAME_OWNER (1 << 15)

      /* Set when frame has been allocated for this next. */
    #define VLIB_FRAME_IS_ALLOCATED	VLIB_NODE_FLAG_IS_OUTPUT

      /* Set when frame has been added to pending vector. */
    #define VLIB_FRAME_PENDING VLIB_NODE_FLAG_IS_DROP

      /* Set when frame is to be freed after dispatch. */
    #define VLIB_FRAME_FREE_AFTER_DISPATCH VLIB_NODE_FLAG_IS_PUNT

      /* Set when frame has traced packets. */
    #define VLIB_FRAME_TRACE VLIB_NODE_FLAG_TRACE

      /* Number of vectors enqueue to this next since last overflow. */
      u32 vectors_since_last_overflow;
    } vlib_next_frame_t;
```

图节点调度函数调用vlib_get_next_frame（…），将“（u32 *）to_next”设置到vlib_frame_t中与从当前节点到指定的下一个节点的第i个弧（aka next0）相对应的正确位置。

经过一番摸索-宏的两个级别-处理到达vlib_get_next_frame_internal（…）。 Get-next-frame-internal提取与所需图弧相对应的vlib_next_frame_t。

下一个帧数据结构相当于一个以图弧为中心的帧缓存。节点完成向框架中添加元素后，它将获取vlib_pending_frame_t并最终到达图调度程序的运行队列。但是，不能保证不会将更多矢量元素从相同（source_node，next_index）弧或从不同（source_node，next_index）弧添加到基础框架。

保持弧到帧缓存的一致性是必要的。保持一致性的第一步是确保一次只有一个图节点认为它“拥有”目标vlib_frame_t。

返回图节点调度功能。在通常情况下，一定数量的数据包将添加到通过调用vlib_get_next_frame（…）获得的vlib_frame_t中。

在调度函数返回之前，需要为其实际使用的所有图弧调用vlib_put_next_frame（…）。该操作将vlib_pending_frame_t添加到图调度程序的待处理帧向量中。

Vlib_put_next_frame在待处理帧（pending frame）索引和vlib_next_frame_t索引中记录。

### dispatch_pending_node动作
如上所示，主图调度循环调用带处理节点。

Dispatch_pending_node恢复挂起的帧，以及图节点的运行时/调度功能。此外，它恢复当前与vlib_frame_t关联的next_frame，并将vlib_frame_t与next_frame分离。

代码在…/src/vlib/main.cdispatch_pending_node（…）中，请注意以下节：

```
  /* Force allocation of new frame while current frame is being
     dispatched. */
  restore_frame_index = ~0;
  if (nf->frame_index == p->frame_index)
    {
      nf->frame_index = ~0;
      nf->flags &= ~VLIB_FRAME_IS_ALLOCATED;
      if (!(n->flags & VLIB_NODE_FLAG_FRAME_NO_FREE_AFTER_DISPATCH))
	    restore_frame_index = p->frame_index;
    }
```

由于它实现了一些二阶优化，所以值得一试。几乎在事后，它调用了dispatch_node，而后者实际上调用了图节点的调度功能。

### 进程/线程模型

### 过程事件

### 缓冲区

### 共享内存消息API

### 调试CLI

### 在线程之间传递缓冲区