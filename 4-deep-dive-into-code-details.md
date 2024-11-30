---
date: "2024-11-30T11:31:13.68560091+08:00"
lang: zh
slug: reading-large-code-challenges-and-practices-4-deep-dive-into-code-details
title: 阅读大规模代码：挑战与实践（4）深入代码细节
---
## 深入代码细节

在掌握了代码的宏观结构，尤其是模块关系和核心概念之后，我们终于开始面对最头疼和耗时的部分——深入细节。本章将介绍几种有效的方法，帮助你分析代码，掌握其内部运作机制。

**本章最关键，最有用的知识就是学会反向阅读代码，达到事倍功半的效果！**

**警告**：阅读代码细节，虽然字面上要“细”，但并不是真的要流水账般咀嚼每一行。我们基本上要遵循以下原则：

应该是什么样的：
- 理解功能和架构，以便优化、修复或扩展。
- 理解逻辑，从中学习编程思想、风格和最佳实践。
- 按照逻辑链路而非编译器解析链路来理解代码。
- 关注代码的美感和算法的巧妙实现，通过翻阅历史版本简化理解难度。

不应该是什么样的：
- 被动阅读代码，追求细节的理解，而不去理解其功能和架构。
- 总是接受作者的写法，不思考其实别人的写法（哪怕是知名程序员）其实也并不总是好的。
- 尝试逐函数、逐行理解，没有跳跃性。
- 死磕复杂、冗长和难懂的代码，而不去思考其核心逻辑（有可能算法本身简单，只是日积月累已经是屎山，而你却硬啃屎山）。

空话说完，我们必须决定从哪里入手了。

### 阅读测试代码

测试代码提供了对被测试代码预期行为的直接洞察。测试有很多种，一般可以分成集成测试和单元测试。集成测试关注模块之间的接口和交互的正确性。单元测试关注于单个模块的功能正确性。这恰好对应了代码的整体理解和局部理解。通过阅读单元测试，你可以了解函数或模块的使用方式、处理的输入输出以及边界条件。而集成测试则可以帮助你了解模块之间连接关系。二者结合可以得到比较全面的理解。所以从测试开始是最好的方法。

让我们以下面的 LLVM 某个单元测试代码为例。（你不需要事先对 LLVM 有所了解）

```cpp
TEST(UseTest, sort) {
  LLVMContext C;

  const char *ModuleString = "define void @f(i32 %x) {\n"
                             "entry:\n"
                             "  %v0 = add i32 %x, 0\n"
                             "  %v2 = add i32 %x, 2\n"
                             "  %v5 = add i32 %x, 5\n"
                             "  %v1 = add i32 %x, 1\n"
                             "  %v3 = add i32 %x, 3\n"
                             "  %v7 = add i32 %x, 7\n"
                             "  %v6 = add i32 %x, 6\n"
                             "  %v4 = add i32 %x, 4\n"
                             "  ret void\n"
                             "}\n";
  SMDiagnostic Err;
  char vnbuf[8];
  std::unique_ptr<Module> M = parseAssemblyString(ModuleString, Err, C);
  Function *F = M->getFunction("f");
  ASSERT_TRUE(F);
  ASSERT_TRUE(F->arg_begin() != F->arg_end());
  Argument &X = *F->arg_begin();
  ASSERT_EQ("x", X.getName());

  X.sortUseList([](const Use &L, const Use &R) {
    return L.getUser()->getName() < R.getUser()->getName();
  });
  unsigned I = 0;
  for (User *U : X.users()) {
    format("v%u", I++).snprint(vnbuf, sizeof(vnbuf));
    EXPECT_EQ(vnbuf, U->getName());
  }
  ASSERT_EQ(8u, I);

  X.sortUseList([](const Use &L, const Use &R) {
    return L.getUser()->getName() > R.getUser()->getName();
  });
  I = 0;
  for (User *U : X.users()) {
    format("v%u", (7 - I++)).snprint(vnbuf, sizeof(vnbuf));
    EXPECT_EQ(vnbuf, U->getName());
  }
  ASSERT_EQ(8u, I);
}
```

首先，我们看到这是一个测试函数 `UseTest.sort`。这个测试用例的目的是检验 LLVM 中 `Use` 列表的排序功能。

#### 1. 创建 LLVM 上下文和模块

```cpp
LLVMContext C;
const char *ModuleString = "define void @f(i32 %x) {...}";
SMDiagnostic Err;
std::unique_ptr<Module> M = parseAssemblyString(ModuleString, Err, C);
```

- **LLVMContext**: 这是 LLVM 的一个核心类，用于管理编译过程中的全局数据，比如类型和常量。通过上下文，我们能确保多线程环境下的数据安全。
- **Module**: 代表一个 LLVM 模块，是 LLVM IR 中的顶级容器，包含函数和全局变量等。
- **parseAssemblyString**: 这个函数将 LLVM IR 字符串解析为一个模块对象。我们在这里了解到 LLVM 提供了丰富的 API 来处理和操作 IR。

这几行代码让我们意识到，对一个模块源码进行解析，得到一个关联到上下文的模块对象。这让我们对 LLVM 的模块组织有了一个印象。

#### 2. 获取函数和参数

```cpp
Function *F = M->getFunction("f");
ASSERT_TRUE(F);
ASSERT_TRUE(F->arg_begin() != F->arg_end());
Argument &X = *F->arg_begin();
ASSERT_EQ("x", X.getName());
```

这些代码意味着 LLVM IR 中 `define void @f(i32 %x)` 表示一个函数 `f`，其参数名为 `x`。并且我们学到了可以通过 `*F->arg_begin()` 获取函数的第一个参数。这看起来像个迭代器，推测可以遍历函数的所有参数。我们还学到了可以通过 `getName()` 方法获取参数的名称。

> 而这事先不需要你对 LLVM IR 有所了解。

#### 3. 使用列表排序

```cpp
X.sortUseList([](const Use &L, const Use &R) {
  return L.getUser()->getName() < R.getUser()->getName();
});
```

这里显然定义排序规则，根据 User 的 Name 进行排序。

#### 4. 验证排序结果

```cpp
unsigned I = 0;
for (User *U : X.users()) {
  format("v%u", I++).snprint(vnbuf, sizeof(vnbuf));
  EXPECT_EQ(vnbuf, U->getName());
}
ASSERT_EQ(8u, I);
```
这里我们学到的是可以通过 `users()` 方法获取 `Use` 列表，并遍历它。根据断言，我们得到的顺序是 `v0` 到 `v7`。

结合 X 的定义，我们推测 Use 表示哪些“users”用到了 x。

> 实际上这是编译原理的 Use-Def 分析的一部分，虽然我们不懂编译原理，但我们通过阅读测试代码开始有点了解了！

#### 5. 逆序排序

```cpp
X.sortUseList([](const Use &L, const Use &R) {
  return L.getUser()->getName() > R.getUser()->getName();
});
```

- 这部分代码用相反的排序规则对 `Use` 列表进行逆序排列。略。

小结一下，这里我们仅仅通过测试，就开始有点理解一门新语言 LLVM IR，还对编译中的 Use 关系有了一定的了解。这可比去读 LLVM IR Parser 或者编译原理的书来的快多了！（当然，如果你真的要从事这方面的工作，还是需要去啃的）

### 函数分析：追踪输入变量的来源

现在祭出一个能大大提高阅读效率的方法——变量来源分析。

```rust
impl SelfTime {
    ...
    pub fn self_duration(&mut self, self_range: Range<Duration>) -> Duration {
        if self.child_ranges.is_empty() {
            return self_range.end - self_range.start;
        }

        // by sorting child ranges by their start time,
        // we make sure that no child will start before the last one we visited.
        self.child_ranges
            .sort_by(|left, right| left.start.cmp(&right.start).then(left.end.cmp(&right.end)));
        // self duration computed by adding all the segments where the span is not executing a child
        let mut self_duration = Duration::from_nanos(0);

        // last point in time where we are certain that this span was not executing a child.
        let mut committed_point = self_range.start;

        for child_range in &self.child_ranges {
            if child_range.start > committed_point {
                // we add to the self duration the point between the end of the latest span and the beginning of the next span
                self_duration += child_range.start - committed_point;
            }
            if committed_point < child_range.end {
                // then we set ourselves to the end of the latest span
                committed_point = child_range.end;
            }
        }

        self_duration
    }
}
```

> 摘自：https://github.com/meilisearch/meilisearch/blob/main/crates/tracing-trace/src/processor/span_stats.rs

看见这一大坨代码不要慌，先看函数签名 `fn self_duration(&mut self, self_range: Range<Duration>) -> Duration`，这说明输入一个 Duration，输出也是 Duration，而且函数可能会修改当前对象的状态。

然后我们直接跳到返回值 `self_duration` 双击选中，看到其实修改它的就一处 `self_duration += child_range.start - committed_point;`

![gh](https://raw.githubusercontent.com/pluveto/0images/master/obsidian/1732934445000ld1ivk.png)

此时就知道它其实就是把一系列的区间长度求和，区间的开始为 `child_range.start`，区间的结束为 `committed_point`。

然后我们对 committed_point 还是不了解，再双击选中 committed_point。

![gh](https://raw.githubusercontent.com/pluveto/0images/master/obsidian/1732934566000f8kplv.png)

发现对它的修改也只有一处，就是当 committed_point < child_range.end 时，committed_point 被设置为 child_range.end。

两相结合就是说 self_duration 其实就是本次的 child_range.start 剪掉上次 committed_point 之间的长度。也就是说反映了 child_range 与 committed_point 之间没有执行的区间（空隙）长度。

所以整体代码的意思就是：

1. 先对 child_ranges 进行排序，保证 child_range.start 都小于等于前一个 child_range.end。
2. 然后遍历 child_ranges，将其间的空隙累加到 self_duration，并更新 committed_point。committed_point 表示上次 child_range 的末尾。
3. 最后返回 self_duration。

再看开头的条件，就是说如果 child_ranges 为空，那么 self_duration 就是 self_range.end - self_range.start。即没有子区间，则空隙就是整个区间。

<svg width="500" height="150" xmlns="http://www.w3.org/2000/svg">
  <!-- Self range -->
  <line x1="50" y1="50" x2="450" y2="50" stroke="black" stroke-width="4" />
  <text x="50" y="40" fill="black">Start</text>
  <text x="440" y="40" fill="black">End</text>

  <!-- Child ranges -->
  <line x1="100" y1="50" x2="150" y2="50" stroke="red" stroke-width="4" />
  <line x1="200" y1="50" x2="250" y2="50" stroke="red" stroke-width="4" />
  <line x1="300" y1="50" x2="350" y2="50" stroke="red" stroke-width="4" />

  <!-- Gaps (Self duration) -->
  <line x1="150" y1="50" x2="200" y2="50" stroke="blue" stroke-width="4" stroke-dasharray="4" />
  <line x1="250" y1="50" x2="300" y2="50" stroke="blue" stroke-width="4" stroke-dasharray="4" />
  <line x1="350" y1="50" x2="450" y2="50" stroke="blue" stroke-width="4" stroke-dasharray="4" />

  <!-- Legend -->
  <rect x="50" y="100" width="20" height="10" fill="black" />
  <text x="80" y="110" fill="black">Self Range</text>

  <rect x="200" y="100" width="20" height="10" fill="red" />
  <text x="230" y="110" fill="red">Child Ranges</text>

  <rect x="350" y="100" width="20" height="10" fill="blue" />
  <text x="380" y="110" fill="blue">Self Duration</text>
</svg>

总结一下：来源分析的方法，就是从后往前分析变量的生成和传递路径，从而理解变量的来源。这比你顺着读要好读很多，因为越靠前的语句，越有可能只是用来凑参数的！实际上我们平时写代码也是这样，代码的核心语句就一两条，此上很多条都是为了给这几个核心语句计算参数。

### 过程块理解

理解代码中的过程块（如条件块、函数块等）关键在于删繁就简。

#### 排除Guard语句，找到核心逻辑

Guard 语句通常用于提前退出函数或处理异常情况。排除这些语句，可以让你更专注于函数的主要逻辑。这种代码在业务代码中尤其常见。

> 下面摘自：https://github.com/kubernetes/kubernetes/blob/master/pkg/registry/core/pod/strategy.go

```go
// ResourceLocation returns a URL to which one can send traffic for the specified pod.
func ResourceLocation(ctx context.Context, getter ResourceGetter, rt http.RoundTripper, id string) (*url.URL, http.RoundTripper, error) {
        // Allow ID as "podname" or "podname:port" or "scheme:podname:port".
        // If port is not specified, try to use the first defined port on the pod.
        scheme, name, port, valid := utilnet.SplitSchemeNamePort(id)
        if !valid {
                return nil, nil, errors.NewBadRequest(fmt.Sprintf("invalid pod request %q", id))
        }

        pod, err := getPod(ctx, getter, name)
        if err != nil {
                return nil, nil, err
        }

        // Try to figure out a port.
        if port == "" {
                for i := range pod.Spec.Containers {
                        if len(pod.Spec.Containers[i].Ports) > 0 {
                                port = fmt.Sprintf("%d", pod.Spec.Containers[i].Ports[0].ContainerPort)
                                break
                        }
                }
        }
        podIP := getPodIP(pod)
        if ip := netutils.ParseIPSloppy(podIP); ip == nil || !ip.IsGlobalUnicast() {
                return nil, nil, errors.NewBadRequest("address not allowed")
        }

        loc := &url.URL{
                Scheme: scheme,
        }
        if port == "" {
                // when using an ipv6 IP as a hostname in a URL, it must be wrapped in [...]
                // net.JoinHostPort does this for you.
                if strings.Contains(podIP, ":") {
                        loc.Host = "[" + podIP + "]"
                } else {
                        loc.Host = podIP
                }
        } else {
                loc.Host = net.JoinHostPort(podIP, port)
        }
        return loc, rt, nil
}
```

这里很多语句都是验证参数之类的，删掉之后变成这样（port 为空的逻辑，一定程度上也可以看做 edge case，虽然对程序正确性至关重要，但是阅读的时候瞟一眼就行）：

```go
// ResourceLocation returns a URL to which one can send traffic for the specified pod.
func ResourceLocation(ctx context.Context, getter ResourceGetter, rt http.RoundTripper, id string) (*url.URL, http.RoundTripper, error) {
        // Allow ID as "podname" or "podname:port" or "scheme:podname:port".
        // If port is not specified, try to use the first defined port on the pod.
        scheme, name, port, _ := utilnet.SplitSchemeNamePort(id)
        pod, _ := getPod(ctx, getter, name)
        podIP := getPodIP(pod)
        loc := &url.URL{
                Scheme: scheme,
        }
        loc.Host = net.JoinHostPort(podIP, port)
        return loc, rt, nil
}
```

是不是感觉一下子代码变得慈眉善目了？严格来说这个代码的核心逻辑就两句话：

1. 通过 `utilnet.SplitSchemeNamePort` 解析不同部分。
2. 构造一个 URL 对象，并设置 Host 字段。

#### 利用命名猜测功能

良好的命名可以提供大量信息，结合上下文，能帮助你快速理解代码的功能和目的。通过分析变量、函数和类的命名，可以推测其作用和关联。

不良好的命名，咱也可以先大胆猜测。下面的代码摘自 https://github.com/LuaJIT/LuaJIT/blob/v2.1/src/lj_mcode.c

```cpp
struct text_region FindNodeTextRegion() {
  struct text_region nregion;
#if defined(__linux__) || defined(__FreeBSD__)
  dl_iterate_params dl_params;
  uintptr_t lpstub_start = reinterpret_cast<uintptr_t>(&__start_lpstub);

#if defined(__FreeBSD__)
  // On FreeBSD we need the name of the binary, because `dl_iterate_phdr` does
  // not pass in an empty string as the `dlpi_name` of the binary but rather its
  // absolute path.
  {
    char selfexe[PATH_MAX];
    size_t count = sizeof(selfexe);
    if (uv_exepath(selfexe, &count))
      return nregion;
    dl_params.exename = std::string(selfexe, count);
  }
#endif  // defined(__FreeBSD__)

  if (dl_iterate_phdr(FindMapping, &dl_params) == 1) {
    Debug("start: %p - sym: %p - end: %p\n",
          reinterpret_cast<void*>(dl_params.start),
          reinterpret_cast<void*>(dl_params.reference_sym),
          reinterpret_cast<void*>(dl_params.end));

    dl_params.start = dl_params.reference_sym;
    if (lpstub_start > dl_params.start && lpstub_start <= dl_params.end) {
      Debug("Trimming end for lpstub: %p\n",
            reinterpret_cast<void*>(lpstub_start));
      dl_params.end = lpstub_start;
    }

    if (dl_params.start < dl_params.end) {
      char* from = reinterpret_cast<char*>(hugepage_align_up(dl_params.start));
      char* to = reinterpret_cast<char*>(hugepage_align_down(dl_params.end));
      Debug("Aligned range is %p - %p\n", from, to);
      if (from < to) {
        size_t pagecount = (to - from) / hps;
        if (pagecount > 0) {
          nregion.found_text_region = true;
          nregion.from = from;
          nregion.to = to;
        }
      }
    }
  }
#elif defined(__APPLE__)
  struct vm_region_submap_info_64 map;
  mach_msg_type_number_t count = VM_REGION_SUBMAP_INFO_COUNT_64;
  vm_address_t addr = 0UL;
  vm_size_t size = 0;
  natural_t depth = 1;

  while (true) {
    if (vm_region_recurse_64(mach_task_self(), &addr, &size, &depth,
                             reinterpret_cast<vm_region_info_64_t>(&map),
                             &count) != KERN_SUCCESS) {
      break;
    }

    if (map.is_submap) {
      depth++;
    } else {
      char* start = reinterpret_cast<char*>(hugepage_align_up(addr));
      char* end = reinterpret_cast<char*>(hugepage_align_down(addr+size));

      if (end > start && (map.protection & VM_PROT_READ) != 0 &&
          (map.protection & VM_PROT_EXECUTE) != 0) {
        nregion.found_text_region = true;
        nregion.from = start;
        nregion.to = end;
        break;
      }

      addr += size;
      size = 0;
    }
  }
#endif
  Debug("Found %d huge pages\n", (nregion.to - nregion.from) / hps);
  return nregion;
}

```

我们先把非核心代码删掉，把某些逻辑块替换成伪代码：

```cpp
struct text_region FindNodeTextRegion() {
  struct text_region nregion;
  dl_iterate_params dl_params;
  uintptr_t lpstub_start = reinterpret_cast<uintptr_t>(&__start_lpstub);

  if (dl_iterate_phdr(FindMapping, &dl_params) == 1) {
    dl_params.start = dl_params.reference_sym;
    if (lpstub_start > dl_params.start && lpstub_start <= dl_params.end) {
      dl_params.end = lpstub_start;
    }

    if (dl_params.start < dl_params.end) {
      char* from = hugepage_align_up(dl_params.start);
      char* to = hugepage_align_down(dl_params.end);
      Debug("Aligned range is %p - %p\n", from, to);
      if (from < to) {
        size_t pagecount = (to - from) / hps;
        if (pagecount > 0) {
          Set nregion data;
        }
      }
    }
  }
  return nregion;
}
```

删繁就简之后，我们直接定位 nregion 的赋值点，找到了核心逻辑：寻找一个 hugepage 区域。

`dl_iterate_phdr` 这个函数虽然事先不知道什么意思，但是从名字来看有迭代，意味着其内部会循环地执行，从使用来看，返回 == 1 应该是表示找到了东西。而参数 FindMapping 应该是表示查找的回调函数。因此我们推测 `dl_iterate_phdr` 的功能应该是遍历进程的一些操作系统管理的数据，找到其中满足 FindMapping 条件的，把结果放到 dl_params。

实际上 dl 是动态链接（dynamic linking）的缩写。phdr: 程序头（program header）的缩写。所以整个函数就是遍历当前进程加载的所有程序头（包括程序和库）的函数。不同领域会有不同的缩写，

比如：

- irq: Interrupt Request
- pcb: Process Control Block
- mmu: Memory Management Unit
- dma: Direct Memory Access
- vfs: Virtual File System
- ipc: Inter-Process Communication
- pid: Process Identifier
- tss: Task State Segment
- smp: Symmetric Multiprocessing
- numa: Non-Uniform Memory Access

熟悉之后可以帮助我们快速理解代码，避免频繁查文档，递归式查各种文档耽误时间。
