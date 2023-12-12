---
date: "2023-12-07T21:42:41.027500881+08:00"
draft: false
lang: zh
mathjax: true
slug: reading-code-at-scale-challenges-and-practices-3-macro-understanding-of-code-structure
title: 阅读大规模代码：挑战与实践（3）宏观理解代码结构
---

# 阅读大规模代码：挑战与实践（3）宏观理解代码结构

本文链接：<https://www.less-bug.com/posts/reading-code-at-scale-challenges-and-practices-3-macro-understanding-of-code-structure/>

**目录**

1. **引言** [链接](https://github.com/pluveto/Reading-Large-Scale-Code/blob/main/1-introduction-and-contents.md)
    - 1.1 阅读大型、复杂项目代码的挑战
    - 1.2 阅读代码的机会成本
    - 1.3 本系列文章的目的

2. **准备阅读代码** [链接](https://github.com/pluveto/Reading-Large-Scale-Code/blob/main/2-dont-read-code-first.md)
    - 2.1 阅读需求文档和设计文档
    - 2.2 以用户角度深度体验程序

3. **宏观理解代码结构** [链接](https://github.com/pluveto/Reading-Large-Scale-Code/blob/main/3-macro-understanding-of-code-structure.md)
    - 3.1 抓大放小：宏观视角的重要性
    - 3.2 建立概念手册：记录关键抽象和接口
    - 3.3 分析目录树
        - 3.3.1 命名推测用途
        - 3.3.2 文件结构猜测功能
        - 3.3.3 代码层面的功能推断

4. **深入代码细节**
    - 4.1 阅读测试代码：单元测试的洞察
    - 4.2 函数分析：追踪输入变量的来源
    - 4.3 过程块理解
        - 4.3.1 排除Guard语句，找到核心逻辑
        - 4.3.2 利用命名猜测功能
        - 4.3.3 倒序阅读：追溯参数构造

5. **分析交互流程**
    - 5.1 交互流程法：用户交互到输出的全流程

6. **调用关系和深层次逻辑**
    - 6.1 可视化调用关系
        - 6.1.1 画布法
        - 6.1.2 树形结构法

7. **辅助工具的使用**
    - 7.1 利用AI理解代码

8. **专项深入**
    - 8.1 阅读算法：理解概念和算法背景
    - 8.2 心态调节：将代码视为己出

9. **结语**
    - 阅读大型代码的心得与建议

---

## 抓大放小：宏观视角的重要性

在阅读代码时，往往面临着海量的代码文件和复杂的代码逻辑。如果我们只关注细节和具体实现，很容易陷入细枝末节的琐碎细节中，导致挫败感和低效率。

宏观视角意味着我们要从整体上抓住代码的结构和组织方式，将代码划分为不同的模块、组件或功能单元。通过宏观视角，我们可以快速了解代码的大致框架，理解各个模块之间的关系和交互方式。这种全局的认知有助于我们更好地理解代码的功能和设计，从而更高效地进行后续的代码阅读和分析工作。

## 建立概念手册：记录关键抽象和接口

这里我们提出了第一个实用的代码阅读方法：建立**概念手册**（Concept Manual）。

概念手册是一个记录关键抽象和接口的文档，用于帮助我们理清代码中的重要概念和它们之间的关系。

尽管许多系统表面上提供简单的接口，但是在实现细节中，它们必然会涉及到一些开发者自己发明的或是领域内通行的抽象。例如 jemalloc 内存分配器涉及到了 `size_class`、`bin`、`arena` 等抽象，并且反复在代码中出现。我们可以首先简单记录下来这些名字，然后在阅读代码时，逐渐理解并完善概念手册。

> 这里给出一篇写得非常好的文章，作者首先介绍了基础概念和结构，以及主要字段，然后分析了相关的算法实现，文章篇幅不长但是已经让人大致知道了整个系统的工作原理：
> [JeMalloc](https://zhuanlan.zhihu.com/p/48957114)（作者 UncP）

建立概念手册应该力求事半功倍，**一种最常见的误区是逐个研究各个字段的含义**，这样立刻你会陷入细节，递归式翻遍全网，然后因为一些小问题卡一整天。举个例子，`mm_struct` 是 Linux 内核中最重要的结构之一，它的定义如下：

```c
struct mm_struct {
	struct {
		struct {
			atomic_t mm_count;
		} ____cacheline_aligned_in_smp;

		struct maple_tree mm_mt;
#ifdef CONFIG_MMU
		unsigned long (*get_unmapped_area) (struct file *filp,
				unsigned long addr, unsigned long len,
				unsigned long pgoff, unsigned long flags);
#endif
		unsigned long mmap_base;	
		unsigned long mmap_legacy_base;	
#ifdef CONFIG_HAVE_ARCH_COMPAT_MMAP_BASES

		unsigned long mmap_compat_base;
		unsigned long mmap_compat_legacy_base;
#endif
		unsigned long task_size;	
		pgd_t * pgd;

#ifdef CONFIG_MEMBARRIER
		atomic_t membarrier_state;
#endif

		atomic_t mm_users;

#ifdef CONFIG_SCHED_MM_CID
		struct mm_cid __percpu *pcpu_cid;
		unsigned long mm_cid_next_scan;
#endif
#ifdef CONFIG_MMU
		atomic_long_t pgtables_bytes;	
#endif
		int map_count;			

		spinlock_t page_table_lock;
		struct rw_semaphore mmap_lock;

		struct list_head mmlist;
#ifdef CONFIG_PER_VMA_LOCK
		int mm_lock_seq;
#endif

		unsigned long hiwater_rss; 
		unsigned long hiwater_vm;  

		unsigned long total_vm;	   
		unsigned long locked_vm;   
		atomic64_t    pinned_vm;   
		unsigned long data_vm;	   
		unsigned long exec_vm;	   
		unsigned long stack_vm;	   
		unsigned long def_flags;

		seqcount_t write_protect_seq;

		spinlock_t arg_lock; 

		unsigned long start_code, end_code, start_data, end_data;
		unsigned long start_brk, brk, start_stack;
		unsigned long arg_start, arg_end, env_start, env_end;

		unsigned long saved_auxv[AT_VECTOR_SIZE]; 

		struct percpu_counter rss_stat[NR_MM_COUNTERS];

		struct linux_binfmt *binfmt;

		mm_context_t context;

		unsigned long flags; 

#ifdef CONFIG_AIO
		spinlock_t			ioctx_lock;
		struct kioctx_table __rcu	*ioctx_table;
#endif
#ifdef CONFIG_MEMCG
		struct task_struct __rcu *owner;
#endif
		struct user_namespace *user_ns;

		struct file __rcu *exe_file;
#ifdef CONFIG_MMU_NOTIFIER
		struct mmu_notifier_subscriptions *notifier_subscriptions;
#endif
#if defined(CONFIG_TRANSPARENT_HUGEPAGE) && !USE_SPLIT_PMD_PTLOCKS
		pgtable_t pmd_huge_pte; 
#endif
#ifdef CONFIG_NUMA_BALANCING
		unsigned long numa_next_scan;

		unsigned long numa_scan_offset;

		int numa_scan_seq;
#endif
		atomic_t tlb_flush_pending;
#ifdef CONFIG_ARCH_WANT_BATCHED_UNMAP_TLB_FLUSH

		atomic_t tlb_flush_batched;
#endif
		struct uprobes_state uprobes_state;
#ifdef CONFIG_PREEMPT_RT
		struct rcu_head delayed_drop;
#endif
#ifdef CONFIG_HUGETLB_PAGE
		atomic_long_t hugetlb_usage;
#endif
		struct work_struct async_put_work;

#ifdef CONFIG_IOMMU_SVA
		u32 pasid;
#endif
#ifdef CONFIG_KSM
		unsigned long ksm_merging_pages;
		unsigned long ksm_rmap_items;
		unsigned long ksm_zero_pages;
#endif 
#ifdef CONFIG_LRU_GEN
		struct {

			struct list_head list;

			unsigned long bitmap;
#ifdef CONFIG_MEMCG

			struct mem_cgroup *memcg;
#endif
		} lru_gen;
#endif 
	} __randomize_layout;

	unsigned long cpu_bitmap[];
};
```

这里有很多字段，虽然说它的注释非常丰富（这里篇幅所限有删减），但它涉及到很多概念、机制，妄图努努力花个一两天就能理解它的含义是不现实的，也是不理智和不值得的。没有人学习 Linux 的目的是搞懂所有内核源码，即便是 Linus 本人也不敢保证内核中每行代码每个字段他都了然于心。所以如果如果你的目的是研究某个特定的功能，那么你应该**只关注于你需要的部分，以及无论如何都绕不过的部分**，而不是试图把所有的细节都搞懂。

下面详细介绍如何建立概念手册。

总体而言，在建立概念手册时，我们可以首先识别代码中的关键抽象，例如类、接口、数据结构等。对于每个关键抽象，可以记录其名称、核心功能、核心字段以及与其他抽象的关联关系。这个过程不是直接在阅读代码之前完成，而是交替进行。

我们以 [spaskalev/buddy_alloc](https://github.com/spaskalev/buddy_alloc/blob/main/buddy_alloc.h) 这个项目为例，它是一个内存分配器，不要小看它只有两千行，这里两千行不是那种简单的业务代码，而是涉及到了很多算法和数据结构。而且一些比较大的项目，但凡设计比较合理的，拆解之后一个原子模块往往也不会超过一万行，大多也就是几千行。

阅读源码前你应该已经知道了什么是内存分配器，什么是 buddy 分配。

源码的前五百行内基本上是一些接口定义，你可以粗略地浏览一遍，不过应该还是一头雾水。现在看代码还太早了！

现在请你阅读下面的“分析目录树章节”。

回到 buddy_alloc 项目，我们现在知道，应该先阅读 README 可以了解到用法（Usage）。

> 如果 README 里没有，那么你可以去看看测试代码，测试代码可能有更丰富的用法。关注 tests 和 examples 目录！

```c
size_t arena_size = 65536;
/* You need space for the metadata and for the arena */
void *buddy_metadata = malloc(buddy_sizeof(arena_size));
void *buddy_arena = malloc(arena_size);
struct buddy *buddy = buddy_init(buddy_metadata, buddy_arena, arena_size);

/* Allocate using the buddy allocator */
void *data = buddy_malloc(buddy, 2048);
/* Free using the buddy allocator */
buddy_free(buddy, data);

free(buddy_metadata);
free(buddy_arena);
```

这里非常好地展现了如何使用这个库，以及库中最关键的用户接口、结构等。你应该已经能推测他们的大致用途，请跳转到“代码层面的功能推断”章节继续。

> 当一个库提供非常多的 API 时，你应该先找到最关键的接口，然后从这些接口开始阅读代码。

## 分析目录树

分析代码的目录结构也是宏观理解代码结构的重要步骤之一。代码的目录结构通常反映了代码的模块划分和组织方式，通过分析目录树，我们可以获取关于代码结构的一些线索和信息。

先看一个比较简单的，spaskalev/buddy_alloc 项目的目录树：

```text
.gitignore
CMakeLists.txt
CONTRIBUTING.md
LICENSE.md
Makefile
README.md
SECURITY.md
_config.yml
bench.c
buddy_alloc.h
test-fuzz.c
testcxx.cpp
tests.c
```

你心里应该对它进行归类：

```text
# 文档
README.md
CONTRIBUTING.md
LICENSE.md

# 测试代码
bench.c
test-fuzz.c
testcxx.cpp
tests.c

# 核心代码
buddy_alloc.h

# 构建相关
CMakeLists.txt
Makefile

# 其它
.gitignore
SECURITY.md
_config.yml
```

当然，不是所有项目都这么简单，但分类之后大致也是这些类别，例如 [LLVM](https://github.com/llvm/llvm-project)，一种分类如下：

```text
# 项目配置文件
.arcconfig
.arclint
.clang-format
.clang-tidy
.git-blame-ignore-revs
.gitignore
.mailmap

# 项目文档
CODE_OF_CONDUCT.md
CONTRIBUTING.md
LICENSE.TXT
README.md
SECURITY.md

# 持续集成/持续部署配置
.ci/
.github/

# LLVM项目主要组件
llvm/
clang/
clang-tools-extra/
lld/
lldb/
mlir/
polly/

# 运行时库
compiler-rt/
libc/
libclc/
libcxx/
libcxxabi/
libunwind/
openmp/

# 其他语言和工具支持
flang/
bolt/

# 跨项目测试
cross-project-tests/

# 第三方库
third-party/

# 通用工具
utils/

# 并行STL实现
pstl/

# LLVM运行时环境
runtimes/

# CMake支持
cmake/
```

对于第一次阅读这个项目的人，你可能感到无法分类，因为看不出来。这种情况下，我会立刻去看 CONTRIBUTING.md、README.md 文件。根据里面的指引，你可能会得到官方提供的目录树介绍，也可能直接重定向到一个专门的文档网站，例如 [LLVM](https://llvm.org/docs/)。一般比较大的项目都会有这样的文档，并且也会有比较丰富的非官方文档、教程和博客。

### 命名推测用途

首先，我们可以通过目录和文件的命名来推测其用途和功能。通常，良好的命名规范能够提供一些关于代码功能和模块划分的线索。

例如，如果一个目录名为 "utils"，那么很可能它包含了一些通用的工具函数；如果一个文件名以 "controller" 结尾，那么它可能是一个控制器模块的实现。

下面我整理了一些比较通用的常见的文件/目录命名惯例。

- `src`：常用于存放源代码。
- `utils`：通常用于存放通用的工具函数或类。
- `config`：常用于存放配置文件，例如应用程序的配置项、数据库连接配置等。
- `tests`：通常用于存放单元测试或集成测试的代码。
- `examples`：常用于存放示例代码，例如如何使用某个库或框架的示例。
- `docs`：通常用于存放文档，例如项目的说明文档、API 文档等。
- `scripts`：常用于存放脚本文件，例如构建脚本、部署脚本等。
- `dist` / `build`：通常用于存放构建后的发布版本，例如编译后的可执行文件或打包后的压缩包。通常会被添加到 `.gitignore` 中，因为它们可以通过源代码构建而来。
- `lib`：通常用于存放库文件。
- `include`：常用于存放头文件。
- `bin`：通常用于存放可执行文件，一般也会被添加到 `.gitignore` 中。

我们自己写代码时也最好遵循惯例，保持一致性和可读性，以便别人能够轻松理解代码结构和功能。

对于不同类型的项目，目录结构会有所变化。下面是一些特定类型的项目以及它们可能包含的目录：

Web 类项目：

- `controllers`：常用于存放控制器模块的实现，负责处理请求和控制应用逻辑。
- `models`：通常用于存放数据模型的定义和操作，例如数据库表的映射类或数据结构的定义。
- `views`：常用于存放视图文件，即用户界面的展示层。
- `services`：通常用于存放服务层的实现，负责处理业务逻辑和与数据访问层的交互。
- `public`：常用于存放公共资源文件，例如静态文件（CSS、JavaScript）或上传的文件。
- `routes`：通常用于存放路由配置文件或路由处理函数，负责处理不同 URL 路径的请求分发。
- `middlewares`：常用于存放中间件的实现，用于在请求和响应之间进行处理或拦截。

数据科学/机器学习项目：

- `data`：用于存放数据文件，如数据集、预处理后的数据等。
- `notebooks`：用于存放Jupyter笔记本，常用于数据分析和探索性数据分析(EDA)。
- `models`：用于存放训练好的模型文件，如`.h5`、`.pkl`等。
- `reports`：用于存放生成的分析报告，可以包括图表、表格等。
- `features`：用于存放特征工程相关的代码。
- `scripts`：用于存放数据处理或分析的独立脚本。

移动应用项目：

- `assets`：用于存放图像、字体和其他静态资源文件。
- `lib`：在像Flutter这样的框架中，用于存放Dart源文件。
- `res`：在Android开发中，用于存放资源文件，如布局XML、字符串定义等。
- `ios`/`android`：用于存放特定平台的原生代码和配置文件。

游戏开发项目：

- `assets`：用于存放游戏资源，如纹理、模型、音效、音乐等。
- `scripts`：用于存放游戏逻辑脚本，如Unity中的C#脚本。
- `scenes`：用于存放游戏场景文件。
- `prefabs`：在Unity中用于存放预设（可重用游戏对象模板）。

嵌入式系统/ IoT项目：

- `src`：用于存放源代码，可能会进一步细分为不同的功能模块。
- `include`：用于存放头文件，特别是在C/C++项目中。
- `drivers`：用于存放与硬件通信的驱动程序代码。
- `firmware`：用于存放固件代码。
- `tools`：用于存放与硬件通信或调试的工具。
- `sdk`：用于存放软件开发工具包。

### 文件结构猜测功能

如果一个目录的名字不足以让你推测出它的用途，那么你可以进一步分析目录中的文件结构。例如 LLVM 有一个目录叫做 BinaryFormat，光看名字会有点迷惑，但是如果看下面的文件：

```
BinaryFormat/
    ELFRelocs/
    AMDGPUMetadataVerifier.h
    COFF.h
    DXContainer.h
    DXContainerConstants.def
    Dwarf.def
    Dwarf.h
    DynamicTags.def
    ELF.h
    GOFF.h
    MachO.def
    MachO.h
    Magic.h
    Minidump.h
    MinidumpConstants.def
    MsgPack.def
    MsgPack.h
    MsgPackDocument.h
    MsgPackReader.h
    MsgPackWriter.h
    Swift.def
    Swift.h
    Wasm.h
    WasmRelocs.def
    WasmTraits.h
    XCOFF.h
```

就知道这个 Binary 原来是指的是 ELF 之类的二进制文件格式。再看到 WASM（一种浏览器里运行的二进制）COFF（Windows 系统的二进制）啥的，可以推测这个目录是用于存放二进制文件格式的定义和解析的，并且支持了很多种格式包括 ELF、MachO、WASM 等。你看，光从一个目录结构就能推测出这么多信息，某种意义上比你钻进去读代码更快。

> 有时候我们会猜错，但是没关系，我们可以在阅读代码的过程中不断完善概念手册。

### 代码层面的功能推断

除了目录和文件的分析，我们还可以通过观察代码的功能和调用关系来推断代码的整体结构。通过仔细观察代码中的函数、方法、类之间的调用关系，我们可以推断出它们之间的依赖关系和组织方式。

- 核心公开接口的特征：在示例代码中被频繁使用
- 核心、底层功能的特征：一个函数/方法/类被多个其他方法调用。
- 包装器的特征：短小的函数，并且基本上是对另一个/类模块的调用。
- 上层代码的特征：调用了多个其他模块的函数/方法/类，很长的 import 列表。
- ...

回到 buddy_alloc 项目，我们阅读示例代码，可以推测出一些比较重要的概念和函数：

- `buddy_metadata` - 用于存储 buddy 分配器的元数据
- `buddy_arena` - 一块一次性分配好的内存，后面的 buddy 分配器会从这里细分分配内存
- `buddy` - buddy 分配器的实例
- `buddy_init` - 初始化 buddy 分配器
- `buddy_malloc` - 分配内存
- `buddy_free` - 释放内存

我们还能推测出 buddy_metadata、buddy 应该是两个需要重点了解的核心结构，buddy_init、buddy_malloc、buddy_free 是三个重要的公开函数。

你可以把这些先记到概念手册里。然后进一步阅读代码的过程中，会涉及到他们的具体实现，以及更多隐含的概念。比如二叉树、tree_order、depth、index、目标深度、size_for_order、internal_position、local_offset、bitset 等。这是一个不断探索的过程。

---

本章介绍了宏观理解代码结构的重要性，并提供了一些实用的方法和技巧。通过采用宏观视角，建立概念手册，分析目录树以及观察代码的功能和调用关系，我们可以更好地理解代码的整体结构，为后续的代码阅读和分析工作打下坚实的基础。在下一章中，我们将介绍如何进行代码的细节分析，以更深入地理解代码的实现细节。

