# Reading-Large-Scale-Code

我是 Pluveto，一个热爱学习的开发者。阅读代码是软件工程师的核心能力，但是我发现，关于如何写代码，以及各种系统的代码阅读笔记虽然非常多，但还是缺乏一本小册子能够由浅入深介绍如何阅读代码，恰好近几年接触了一些比较大的项目，包括开源项目和内部项目，我想把这些东西整理下来。

所有内容将会在这个仓库中更新。


**目录**

1. **引言** [链接](https://www.less-bug.com/posts/reading-large-scale-code-challenges-and-practices-1-introduction-and-contents/)
    - 1.1 阅读大型、复杂项目代码的挑战
    - 1.2 阅读代码的机会成本
    - 1.3 本系列文章的目的

2. **准备阅读代码** [链接](https://www.less-bug.com/posts/reading-code-at-scale-challenges-and-practices-2-dont-read-code-first/)
    - 2.1 阅读需求文档和设计文档
    - 2.2 以用户角度深度体验程序

3. **宏观理解代码结构** [链接](https://github.com/pluveto/Reading-Large-Scale-Code/blob/main/3-macro-understanding-of-code-structure.md)
    - 3.1 抓大放小：宏观视角的重要性
    - 3.2 建立概念手册：记录关键抽象和接口
    - 3.3 分析目录树
        - 3.3.1 命名推测用途
        - 3.3.2 文件结构猜测功能
        - 3.3.3 代码层面的功能推断

4. **深入代码细节** [链接](https://github.com/pluveto/Reading-Large-Scale-Code/blob/main/4-deep-dive-into-code-details.md) [番外篇](https://github.com/pluveto/Reading-Large-Scale-Code/blob/main/4-reading-large-code-challenges-and-practices-4-ex-extra-1000-line-read-in-action.md)
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

博客会同步更新：<https://www.less-bug.com>
