---
name: review
description: Review currently modified code. Analyzes implementation vulnerabilities, logic design, coupling degree, and code reusability, then outputs a prioritized optimization list from high to low priority. Use when the user asks to review changes, check code quality, or audit modifications.
argument-hint: "[optional: specific file or focus area]"
user-invocable: true
allowed-tools: Bash, Read, Grep, Glob
---

## 任务

对当前工作区中已修改的代码进行全面 Review，从以下 4 个维度分析问题，最终输出按优先级排列的优化列表。

## 第一步：获取变更范围

执行以下命令获取当前修改内容：

```
!`git diff HEAD 2>/dev/null || git diff 2>/dev/null`
```

如果有暂存区变更一并获取：

```
!`git diff --cached 2>/dev/null`
```

如果 $ARGUMENTS 指定了文件或目录，则只聚焦该范围。

## 第二步：理解上下文

对涉及修改的文件，使用 Read 工具阅读完整文件内容，理解修改前后的上下文，避免仅凭 diff 片段误判。

## 第三步：从 4 个维度分析

### 1. 实现漏洞 (Vulnerabilities)
关注点：
- 安全漏洞：注入、越权、敏感信息泄露、不安全的反序列化
- 边界条件未处理：空值、空集合、整数溢出、除零
- 并发问题：竞态条件、死锁、非线程安全操作
- 资源泄露：未关闭的连接、文件句柄、内存泄露
- 错误处理缺失或过于宽泛的 catch

### 2. 逻辑方案 (Logic & Design)
关注点：
- 业务逻辑正确性：是否符合预期行为
- 边缘 case 覆盖是否完整
- 算法或数据结构选择是否合理
- 条件分支是否有遗漏或冲突
- 异步/同步处理方式是否恰当

### 3. 耦合程度 (Coupling)
关注点：
- 模块间依赖是否过强（循环依赖、硬编码引用）
- 是否违反单一职责原则
- 是否过度依赖全局状态或单例
- 接口设计是否暴露了不必要的内部细节
- 修改此处是否会产生意想不到的连锁反应

### 4. 可复用程度 (Reusability)
关注点：
- 重复代码（DRY 原则违反）
- 魔法数字、硬编码字符串
- 函数/方法职责是否单一、是否可独立测试
- 是否有更通用的抽象机会
- 命名是否清晰表达意图

## 第四步：输出优化列表

**不要修改任何代码**，只列举问题。按优先级从高到低输出：

---

## Code Review 结果

### 变更范围
简要说明本次 Review 覆盖的文件和改动概述。

---

### 优化列表（按优先级排序）

🔴 **高优先级**（影响正确性、安全性或稳定性）
- `file.ts:42` [实现漏洞] 问题描述 → 建议

🟡 **中优先级**（影响可维护性、扩展性）
- `file.ts:10` [耦合程度] 问题描述 → 建议

🟢 **低优先级**（代码质量改进）
- `file.ts:5` [可复用程度] 问题描述 → 建议

---

## 注意事项
- 只列举问题，不修改任何代码
- 没有发现问题的维度直接跳过，不要强行填充
- 优先级：高=运行时正确性/安全，中=可维护性，低=风格/优化机会
- 建议要具体可操作，指向具体文件和行号
