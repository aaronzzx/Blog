# 前言

从这篇文章开始将会进入红黑树的范畴，而在学习红黑树之前必须先学会**B-树**，只要掌握了B-树那么红黑树也是手到擒来。

# 什么是B-树

> **B-树**是一种平衡的多路搜索树，这种数据结构常被应用在数据库和文件系统的实现上。

如下面两棵树都是B-树，分别是三阶B树（也叫2-3树）和四阶B树（也叫2-3-4树）：

![three_level_b_tree](https://raw.githubusercontent.com/aaronzzx/blog/main/images/three_level_b_tree.png)![four_level_b_tree](https://raw.githubusercontent.com/aaronzzx/blog/main/images/four_level_b_tree.png)

观察这两棵树可以发现一些特点：

1. 一个节点可以存储超过 2 个元素，可以拥有超过 2 个子节点
2. 拥有二叉搜索树的一些性质
3. 平衡，每个节点所有子树高度一致
4. 比较矮，即使存储了十几个元素，树的高度也只有 3

在大概了解了B-树后，接下来学习一下B-树的性质。

# m阶B-树的性质（m ≥ 2）

假设一个节点存储的元素个数为 **x**

根节点：**1 ≤ x ≤ m - 1**

非根节点：**⌈ m / 2 ⌉ - 1 ≤ x ≤ m - 1**（⌈ m / 2 ⌉ 表示结果向上取整，⌊ m / 2 ⌋ 表示结果向下取整）

如果有子节点，子节点个数：**y = x + 1**