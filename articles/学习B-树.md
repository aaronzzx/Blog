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

# B-树对比二叉搜索树

观察以下的**B-树**和**二叉搜索树**：

![four_level_b_tree_compare_bst](https://raw.githubusercontent.com/aaronzzx/blog/main/images/four_level_b_tree_compare_bst.png)![bst_compare_four_level_b_tree](https://raw.githubusercontent.com/aaronzzx/blog/main/images/bst_compare_four_level_b_tree.png)

可以发现它们还是可以有共同点的，例如将二叉搜索树的 **8** 与父节点 **4** 合并，**12** 与父节点 **10** 合并，**13** 、**15** 与父节点 **14** 合并，那么可以得到这样的一棵树：

![bst_to_b_tree](https://raw.githubusercontent.com/aaronzzx/blog/main/images/images\bst_to_b_tree.jpg)

可以发现，合并后的二叉搜索树完美对应上面的B-树，那么肯定存在某种规则能够使二叉搜索树转换为B-树。

# 添加导致上溢

**向B-树添加一个元素，必然是添加到叶子节点中。**

在二叉搜索树中我们是进行大小比对的二分查找，除去元素相等的情况，一定是找到度为零或者度为一的节点才停下来，那么这个节点就是新节点的父节点。

而对于B-树，它的叶子节点一定都是在同一层，且每个节点存储的元素个数最少 ≥ 2（m阶B-树，m ≥ 3），例如有这样一棵4阶B-树：

![four_level_b_tree_compare_bst](https://raw.githubusercontent.com/aaronzzx/blog/main/images/four_level_b_tree_compare_bst.png)

往里添加一个 16 ，过程是这样的：首先与【4，8】对比，发现更大，往右子节点找，与【10，12】对比，发现还是大，接着往右子节点找，与【13，14，15】对比，发现还是大，但这时已经没有子节点了，说明当前这个节点就是叶子节点，于是将 16 插入这个叶子节点中。

插入后的结果有两个，第一种是元素数量没达到B-树单个节点最多可存储元素的上限，这种情况无需任何操作；第二种是达到了上限，以上面这棵4阶B-树为例，它允许单个节点最多存储 3 个元素，但眼下的情况明显已经超了，于是需要进行上溢操作。

什么是上溢？

当单个节点的元素个数超出限制后（≥ m阶B-树的 m），取出当前节点中间的一个元素向上与父节点合并，同时中间元素的左右分裂为 2 个子节点，这 2 个子节点的元素个数必然不会低于最低限制（⌈ m / 2 ⌉ - 1）。

一次分裂完成后可能导致父节点上溢，按照此方法解决。

文字不好理解，还是看图：

![b_tree_overflow](https://raw.githubusercontent.com/aaronzzx/blog/main/images/b_tree_overflow.jpg)

