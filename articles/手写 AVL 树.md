# 前言

在学习**AVL树**之前，需要已经掌握二叉搜索树的相关知识，之前的文章[手写二叉树]( [https://github.com/aaronzzx/blog/blob/main/articles/%E6%89%8B%E5%86%99%E4%BA%8C%E5%8F%89%E6%A0%91.md](https://github.com/aaronzzx/blog/blob/main/articles/手写二叉树.md) ) , [二叉树之实现集合接口]( [https://github.com/aaronzzx/blog/blob/main/articles/%E4%BA%8C%E5%8F%89%E6%A0%91%E4%B9%8B%E5%AE%9E%E7%8E%B0%E9%9B%86%E5%90%88%E6%8E%A5%E5%8F%A3.md](https://github.com/aaronzzx/blog/blob/main/articles/二叉树之实现集合接口.md) )已经详细叙述过了，那么现在就开始 AVL 树的学习。

# 回顾二叉搜索树

在二叉搜索树的学习中可以发现，它的添加、删除、查询时间复杂度都是在 **O(logn)** 级别，效率与树的高度有关，但在前面的学习中我们添加元素的时候是按照自己指定的顺序来添加的，例如添加这样一个数组，那这棵树就长这样：

```
[7, 4, 9, 2, 5, 8, 11]

     7
   /   \
  4     9
 / \   / \
2   5 8  11
```

很明显这是一颗完美二叉树，但是如果由小到大进行添加呢？这棵树又是长什么样？

```
[2, 4, 5, 7, 8, 9, 11]

2
 \
  4
   \
    5
     \
      7
       \
        8
         \
          9
           \
           11
```

啥呀这是？这可不就是链表吗？这种情况称之为树退化成了链表，相应的时间复杂度也从 **O(logn)** 退化成 **O(n)** ，如果是这样那二叉搜索树还有什么用呢？不慌，这时候就需要我们的**平衡二叉树**登场了。

# 平衡二叉树

> **平衡树**是[计算机科学](https://zh.wikipedia.org/wiki/计算机科学)中的一类数据结构，为改进的[二叉查找树](https://zh.wikipedia.org/wiki/二叉查找树)。一般的二叉查找树的查询复杂度取决于目标结点到树根的距离（即深度），因此当结点的深度普遍较大时，查询的均摊复杂度会上升[[1\]](https://zh.wikipedia.org/wiki/平衡树#cite_note-knuth-1)。为了实现更高效的查询，产生了**平衡树**。
>
> [![img](https://upload.wikimedia.org/wikipedia/commons/thumb/a/a9/Unbalanced_binary_tree.svg/240px-Unbalanced_binary_tree.svg.png)](https://zh.wikipedia.org/wiki/File:Unbalanced_binary_tree.svg)
>
> 不平衡的[树结构](https://zh.wikipedia.org/wiki/树_(数据结构))
>
> [![img](https://upload.wikimedia.org/wikipedia/commons/thumb/0/06/AVLtreef.svg/240px-AVLtreef.svg.png)](https://zh.wikipedia.org/wiki/File:AVLtreef.svg)
>
> 平衡的[树结构](https://zh.wikipedia.org/wiki/树_(数据结构))
>
> 在这里，平衡指所有叶子的深度趋于平衡，更广义的是指在树上所有可能查找的均摊复杂度偏低。
>
> ——来自维基百科

如果用平衡二叉树来添加上面的有序数组，则树应该是这样：

```
[2, 4, 5, 7, 8, 9, 11]

     7
   /   \
  4     9
 / \   / \
2   5 8  11
```

没错，跟一开始我们自己指定顺序一样，那么它是怎么做到的？这时候就该**AVL树**登场了。

# AVL 树

AVL 树取名自两位发明家 G. M. Adelson-Velsky 和 Evgenii Landis ，**它是一棵自平衡二叉搜索树， 任一节点对应的两棵子树的最大高度差为1（两棵子树的高度相减称为平衡因子）** ，增加和删除元素的操作可能需要借由一次或多次树旋转，以实现树的重新平衡。 

- 添加导致失衡

  ![add-lose_balance](https://raw.githubusercontent.com/aaronzzx/blog/main/images/add-lose_balance.png)

  在没添加 **13** 之前这棵树是平衡的，因为它各个节点的平衡因子绝对值都不大于 1 ，如 **9** 的平衡因子是 -1 ，**6** 的平衡因子是 0 ，**15** 的平衡因子是 1 ，**14** 的平衡因子是 1 。

  ![add-lose_balance2](https://raw.githubusercontent.com/aaronzzx/blog/main/images/add-lose_balance2.png)

  再来看添加 **13** 这个元素后，**9** 的平衡因子变成 -2 ，**6** 的平衡因子不变，**15** 的平衡因子变成 2 ，**14** 的平衡因子变成 2 。

  可以看到，只是添加了一个元素，直接导致多个节点失衡，并且从这里我们应该可以发现一个规律，失衡节点全是新添加节点的**祖父节点**或**祖先节点**，如 **14** 、**15** 、**9** ，也就是从新添加节点出发一路从节点的 parent 往上找，这条线才可能出现失衡，而失衡的根本原因其实就是子树的高度增加了。

  接着来进行旋转，使它恢复平衡，如下：

  ![rebalance1](https://raw.githubusercontent.com/aaronzzx/blog/main/images/rebalance1.png)

  这样旋转过后整棵树就恢复平衡了，并且我们只是调整了 **12，13，14** 这棵子树，旋转操作在后面讲到。

  另外，新添加节点后，无论如何都不会造成父节点失衡，因为父节点在添加前要么是度为 1 或 0 ，这种情况下它的平衡因子在添加后绝对值都不可能大于 1 。

- 删除导致失衡

  ![remove-lose_balance](https://raw.githubusercontent.com/aaronzzx/blog/main/images/remove-lose_balance.png)

  目前来看这棵树它是平衡的，因为各个节点的平衡因子绝对值都不大于 1 ，接着将 **16** 进行删除。

  ![remove-lose_balance2](https://raw.githubusercontent.com/aaronzzx/blog/main/images/remove-lose_balance2.png)

  可以看到删除 **16** 后导致了 **15** 这个节点失衡了，那我们来旋转一下：

  ![rebalance2](https://raw.githubusercontent.com/aaronzzx/blog/main/images/rebalance2.png)

  可以看到，**12，14，15** 这棵子树也恢复平衡了，但却导致 **11** 这个节点又失衡了，因为在旋转之前 **12，14，15** 的树高度是 3 ，而旋转后变成了 2 ，进而影响到了父节点的平衡，这样父节点又得进行平衡处理，并且可能在平衡处理后又又又影响到了父节点，一直向上可能直达根节点，因此删除元素后的平衡处理可能需要执行 **logn(n = 总节点数量)** 次，即跟整棵树的高度有关。

  基于上面的这种蝴蝶效应，那么我们在写代码的时候势必要进行向上循环进行平衡处理。

总结一下，**添加元素**只可能会导致父节点以上失衡，并且只要这棵失衡子树调整好平衡后整棵树就可以恢复平衡；而**删除元素**可能会导致父节点及以上的祖先节点都失衡，需要从父节点一路往上检查并恢复平衡。

# 平衡处理

先来学习一下旋转，我们需要知道旋转是对谁进行旋转，其实很简单，就是哪个节点失衡那就对哪个节点进行旋转，当然部分情况下旋转有分步骤，下面来看看导致失衡的所有情况。

- LL（left - left）

  ![LL](https://raw.githubusercontent.com/aaronzzx/blog/main/images/LL.png)
  
  可以看到，这种失衡情况是因为添加了 **1** 后导致 **7** 这个节点失衡了，然后看 **1，2，3， 7** 的路径，很明显是 left 到 left...，那么这就是 **LL** 失衡情况，在这种情况下只需要对 **7** 这个祖先节点进行右旋转即可恢复平衡。首先让 **3** 成为这棵树的根节点，然后将 **7** 变成 **3** 的右子节点，这样就完成了右旋转，如下：
  
  ![LL-rotate](https://raw.githubusercontent.com/aaronzzx/blog/main/images/LL-rotate.png)
  
  需要注意的点：原本是 **3** 右节点的 **4** 现在变成了 **7** 的左节点，说到底其实不只是更改根节点的指向就够了，参与旋转节点的子节点都需要被考虑到，总结一下需要考虑的点就是：
  
  1. 更改 **3** 和 **7** 的 parent 指向
  2. 更改 **3** 右子节点的 parent 指向
  3. 更改 **3** 的 right 指向，更改 **7** 的 left 指向
  
- RR（right - right）

  ![RR](https://raw.githubusercontent.com/aaronzzx/blog/main/images/RR.png)

  **RR** 的处理其实就和 **LL** 相反，它需要进行左旋转，即对 **4** 进行左旋转，结果是这样：

  ![RR-rotate](https://raw.githubusercontent.com/aaronzzx/blog/main/images/RR-rotate.png)

接着对这两种简单情况做下总结，只要判断是 **LL** 情况的都统一对失衡节点进行**右旋**，而判断是 **RR** 则对失衡节点进行**左旋**，并且由于它们都是单次操作因此也称为**单旋**。

那么既然说到单旋，那是不是有双旋呢？答案是肯定的。

- LR（left - right）

  ![LR](https://raw.githubusercontent.com/aaronzzx/blog/main/images/LR.png)

  可以看到，新元素 **4** 是添加在 **7.left.right** 下面，这种情况就是 **LR** 。针对这种情况就不能只做单旋了，必须先对 **3** 也就是 **7** 的子节点进行一次**左旋**，如下：

  ![LR-rotate1](https://raw.githubusercontent.com/aaronzzx/blog/main/images/LR-rotate1.png)

  可以发现，对 **3** 进行左旋后，**5** 顶替了 **3** 成为 **2，3，4，5** 这棵子树的根节点，并且失衡情况变成了 **LL** ，而 **LL** 的处理上面已经学习过了，那么再对 **7** 进行一次**右旋**，如下：

  ![LR-rotate2](https://raw.githubusercontent.com/aaronzzx/blog/main/images/LR-rotate2.png)

  这样整棵树就恢复了平衡，对于这种进行两次旋转操作的称为**双旋**。

- RL（right - left）

  ![RL](https://raw.githubusercontent.com/aaronzzx/blog/main/images/RL.png)

  **RL** 情况和 **LR** 相反，新元素被添加在 **7.right.left** 下面，那就需要对 **7** 的子节点 **12** 进行一次**右旋**：

  ![RL-rotate1](https://raw.githubusercontent.com/aaronzzx/blog/main/images/RL-rotate1.png)

  对 **12** 右旋后，失衡情况变成了 **RR** ，很明显需要对 **7** 再来一次左旋：

  ![RL-rotate2](https://raw.githubusercontent.com/aaronzzx/blog/main/images/RL-rotate2.png)

总结一下 **LR** 和 **RL** ：如果判断是 **LR** ，需要先对失衡节点的 **left** 进行一次**左旋**，然后再对失衡节点进行一次**右旋**；而 **RL** 则是对失衡节点的 **right** 先进行一次**右旋**，然后再对失衡节点进行一次**左旋**。

AVL 树的失衡情况只有这 4 种，掌握了这 4 种失衡情况后，我们就可以进入实战环节了。

在这之前先对上面提到的被我们操作的节点（上面用的是数字，可能不是很清晰）做一个身份定义，例如失衡节点我们统一称为**爷爷（grand）**节点，爷爷节点的子节点（失衡的那一方）称为**爸爸（papa）**节点，爸爸节点的子节点（失衡的那一方）称为**儿子（son）**节点，下面的编码阶段我们会用到这三个定义。

# 调整二叉搜索树

首先我们的 AVL 树是需要继承二叉搜索树的，并且由于平衡处理都是在添加、删除元素之后，那么前面写的二叉搜索树就应该修改一下，如下：

```kotlin
open class BST<E> : BinaryTree<E>(), MutableTree<E> {

	// ... 省略多余代码
	
	override fun add(element: E): Boolean {
        require(element != null) {
            "The element must not be null."
        }
        var node = root
        if (node == null) {
            node = createNode(element, null)
            root = node
            size++
            modCount++
            afterAdd(node)
            return true
        }
        var compare = 0
        var parent = node
        while (node != null) {
            parent = node
            compare = compare(element, node.item)
            if (compare < 0) {
                node = node.left
            } else if (compare > 0) {
                node = node.right
            } else {
                node.item = element
                return true
            }
        }
        val newNode = createNode(element, parent)
        if (compare < 0) {
            parent!!.left = newNode
        } else {
            parent!!.right = newNode
        }
        size++
        modCount++
        afterAdd(newNode)
        return true
    }
    
    private fun remove(node: TreeNode<E>?): Boolean {
        var _node = node ?: return false
        if (_node.hasTwoChildren) {
            val succ = successor(_node)!!
            _node.item = succ.item
            _node = succ
        }
        val replacement = _node.left ?: _node.right
        if (replacement != null) {
            replacement.parent = _node.parent
            if (_node.parent == null) {
                root = replacement
            } else if (_node.isLeftChild) {
                _node.parent!!.left = replacement
            } else {
                _node.parent!!.right = replacement
            }
        } else if (_node.parent == null) {
            root = null
        } else {
            if (_node.isLeftChild) {
                _node.parent!!.left = null
            } else {
                _node.parent!!.right = null
            }
        }
        size--
        modCount++
        afterRemove(_node)
        return true
    }
    
    protected open fun afterAdd(node: TreeNode<E>) = Unit
    
    protected open fun afterRemove(node: TreeNode<E>) = Unit
}
```

增加了两个可供子类重写的方法：`afterAdd()` 和 `afterRemove()` ，并在添加和删除后调用。

接下来就是 AVL 树的逻辑了。

# 实现 AVL 树

- 定义框架

  AVL 树需要继承二叉搜索树，那么下面的代码应该是这样：

  ```kotlin
  class AVLTree<E> : BST<E>() {
      
      override fun afterAdd(node: TreeNode<E>) {
      }
      
      override fun afterRemove(node: TreeNode<E>) {
      }
  }
  ```

- 定义 AVL 树的节点

  再者，AVL 树的节点是有**高度**、**平衡因子**等定义的，而在之前的二叉搜索树当中我们并没有写，那这里就需要定义一个属于 AVL 树的节点：

  ```kotlin
  class AVLTree<E> : BST<E>() {
  
      // ...
  
      override fun createNode(item: E, parent: TreeNode<E>?): TreeNode<E> {
          return AVLNode(item, parent)
      }
  
      private class AVLNode<E>(item: E, parent: TreeNode<E>?) : TreeNode<E>(item, parent) {
  
          // 平衡因子
          val balanceFactor: Int
              get() = leftHeight - rightHeight
  
          // 当前节点是否平衡
          val isBalanced: Boolean
              get() = abs(balanceFactor) <= 1
  
          // 如果当前节点失衡，那么新元素肯定添加在比较高的子节点中
          val tallerChild: AVLNode<E>?
              get() {
                  if (leftHeight > rightHeight) return left()
                  if (leftHeight < rightHeight) return right()
                  return if (isLeftChild) left() else right()
              }
  
          // 节点的高度
          var height = 1
              private set
  
          // 左子节点的高度
          val leftHeight: Int
              get() = left()?.height ?: 0
  
          // 右子节点的高度
          val rightHeight: Int
              get() = right()?.height ?: 0
  
          // 更新节点高度
          fun updateHeight() {
              height = 1 + max(leftHeight, rightHeight)
          }
  
          // 以下是为了少写强转代码
          fun parent(): AVLNode<E>? {
              return cast(parent)
          }
  
          fun left(): AVLNode<E>? {
              return cast(left)
          }
  
          fun right(): AVLNode<E>? {
              return cast(right)
          }
  
          private fun cast(node: TreeNode<E>?): AVLNode<E>? {
              return node as? AVLNode<E>
          }
      }
  }
  ```

  这里定义了 **AVLNode** ，并且继承了 **TreeNode** ，然后在 `createNode()` 方法中使用我们定义的节点创建对象后返回。到这一步其实整棵树的节点就替换成了 AVL 树的节点了，那么趁热打铁，来写平衡处理的代码。

- 平衡处理

  从前面可以知道，不管是添加导致的失衡还是删除导致的失衡，都需要从当前节点向上进行检查，唯一区别是添加导致的失衡只需要一次平衡操作（单旋或双旋）就可以恢复平衡，而删除导致的失衡则在每一次子树恢复平衡之后还需要向上检查，直到根节点，那么其实它们的处理代码是差不多的，可以这样写：

  ```kotlin
  class AVLTree<E> : BST<E>() {
  
      override fun afterAdd(node: TreeNode<E>) {
          checkBalanceAndFix(node, true)
      }
  
      override fun afterRemove(node: TreeNode<E>) {
          checkBalanceAndFix(node, false)
      }
      
      private fun checkBalanceAndFix(node: TreeNode<E>, forAdd: Boolean) {
          var avlNode = node as? AVLNode<E>
          while (avlNode != null) {
              if (avlNode.isBalanced) {
                  avlNode.updateHeight()
              } else {
                  rebalance(avlNode)
                  if (forAdd) break
              }
              avlNode = avlNode.parent()
          }
      }
  }
  ```

  可以看到添加和删除共用一套代码，然后对 **avlNode** 进行循环，如果它是平衡的那么则需要更新高度，因为添加和删除可能会导致高度变化，如果不平衡则进行修复，调用 `rebalance()` 处理，并且如果是添加情况则处理结束后就可以跳出循环了。

- `rebalance()` 判断失衡情况

  ```kotlin
  class AVLTree<E> : BST<E>() {
      
      // ...
      
      private fun rebalance(grand: AVLNode<E>) {
          val papa = grand.tallerChild!!
          val son = papa.tallerChild!!
          if (papa.isLeftChild) {
              if (son.isLeftChild) {
                  // LL
                  rotateRight(grand)
              } else {
                  // LR
                  rotateLeft(papa)
                  rotateRight(grand)
              }
          } else {
              if (son.isLeftChild) {
                  // RL
                  rotateRight(papa)
                  rotateLeft(grand)
              } else {
                  // RR
                  rotateLeft(grand)
              }
          }
      }
  }
  ```

  这里就列出了前面讲过的 4 种失衡情况，额外说一下 `isLeftChild` 和 `isRightChild` 是在父类 **TreeNode** 中定义的，用来判断节点是父节点的左还是右。

- 具体旋转操作

  ```kotlin
  class AVLTree<E> : BST<E>() {
      
      // ...
      
      // 旋转的代码可能需要借助前面的图以及自己画图才能更好的理解...
      private fun rotateLeft(grand: AVLNode<E>) {
          // 左旋中的爸爸只可能在爷爷的右边，无论是 RR 还是 LR
          val papa = grand.right()!!
          // 根据我们前面定义的爷爷、爸爸、儿子，LR 中的左旋传入的其实是爸爸，
          // 即这里的 grand 是爸爸，那么 son 就有可能为空，为了方便理解，
          // 以下的代码都用 RR 来做假设，理解了 RR 相信 LR 也没问题了。
          val son = papa.left()
          // 儿子成为爷爷右节点，爷爷成为爸爸左节点
          grand.right = son
          papa.left = grand
          // 接下来管理好 parent ，爷爷的 parent 一定不能先换指向
          // 先修改爸爸的父节点指向，原来是指向爷爷
          papa.parent = grand.parent
          if (grand.isLeftChild) {
              // 如果爷爷原来是它父节点的左边
              grand.parent?.left = papa
          } else if (grand.isRightChild) {
              // 如果爷爷原来是它父节点的右边
              grand.parent?.right = papa
          } else {
              // 爷爷没有父节点，意味着爷爷原来是整棵树的根节点
              root = papa
          }
          // 修改儿子和爷爷的 parent 指向
          son?.parent = grand
          grand.parent = papa
          // 更新节点高度，为什么 son 不用更新呢？因为我们没动过 son 的子节点
          grand.updateHeight()
          papa.updateHeight()
      }
  
      // 右旋其实就是原来的 left 变 right ，right 变 left
      private fun rotateRight(grand: AVLNode<E>) {
          val papa = grand.left()!!
          val son = papa.right()
          grand.left = son
          papa.right = grand
          
          papa.parent = grand.parent
          if (grand.isLeftChild) {
              grand.parent?.left = papa
          } else if (grand.isRightChild) {
              grand.parent?.right = papa
          } else {
              root = papa
          }
  
          son?.parent = grand
          grand.parent = papa
  
          grand.updateHeight()
          papa.updateHeight()
      }
  }
  ```

  通过上面的旋转代码，可以发现最后更改 parent 指向的那一段代码其实是一样的，那么可以抽出来。

  ```kotlin
  private fun rotateLeft(grand: AVLNode<E>) {
      // ...
      afterRotate(grand, papa, son)
  }
  
  private fun rotateRight(grand: AVLNode<E>) {
      // ...
      afterRotate(grand, papa, son)
  }
  
  private fun afterRotate(grand: AVLNode<E>, papa: AVLNode<E>, son: AVLNode<E>?) {
      papa.parent = grand.parent
      if (grand.isLeftChild) {
          grand.parent?.left = papa
      } else if (grand.isRightChild) {
          grand.parent?.right = papa
      } else {
          root = papa
      }
      son?.parent = grand
      grand.parent = papa
      grand.updateHeight()
      papa.updateHeight()
  }
  ```

  这样，整个 AVL 树的代码就写好了，接下来测试一下。

- 测试

  ```kotlin
  fun main() {
      val bst = BST<Int>()
      val avl = AVLTree<Int>()
      val random = Random(System.currentTimeMillis())
      repeat(9) {
          val int = random.nextInt(20) + 1
          bst.add(int)
          avl.add(int)
          Thread.sleep(10)
      }
      println("BinarySearchTree")
      BinaryPrintTree(bst).println()
      println("AVLTree")
      BinaryPrintTree(avl).println()
  }
  ```

  结果如下：

  ![compare-bst-avl](https://raw.githubusercontent.com/aaronzzx/blog/main/images/compare-bst-avl.png)

# 旋转代码的另一种写法

什么？还有另一种写法？是的，就是这么刺激，先来看一幅图，从李明杰老师那里截屏的：

![another-rotate](https://raw.githubusercontent.com/aaronzzx/blog/main/images/another-rotate.jpg)

仔细观察 4 种失衡情况在恢复平衡后的树结构，可以发现是一样的，这意味着可以写一套代码来适应这 4 种失衡情况，只要找对图中的 **a，b，c，d，e，f，g** 即可，下面直接贴出代码，自己对照一下，就不运行了，结果是一样的。

```kotlin
private fun rebalance2(grand: AVLNode<E>) {
    val papa = grand.tallerChild!!
    val son = papa.tallerChild!!
    if (papa.isLeftChild) {
        if (son.isLeftChild) {
            rotate(grand, son.left(), son, son.right(), papa, papa.right(), grand, grand.right())
        } else {
            rotate(grand, papa.left(), papa, son.left(), son, son.right(), grand, grand.right())
        }
    } else {
        if (son.isLeftChild) {
            rotate(grand, grand.left(), grand, son.left(), son, son.right(), papa, papa.right())
        } else {
            rotate(grand, grand.left(), grand, papa.left(), papa, son.left(), son, son.right())
        }
    }
}

private fun rotate(
    root: AVLNode<E>,
    a: AVLNode<E>?, b: AVLNode<E>, c: AVLNode<E>?,
    d: AVLNode<E>,
    e: AVLNode<E>?, f: AVLNode<E>, g: AVLNode<E>?
) {
    d.parent = root.parent
    if (root.isLeftChild) {
        root.parent?.left = d
    } else if (root.isRightChild) {
        root.parent?.right = d
    } else {
        this.root = d
    }
    
    a?.parent = b
    c?.parent = b
    b.left = a
    b.right = c
    b.updateHeight()

    e?.parent = f
    g?.parent = f
    f.left = e
    f.right = g
    f.updateHeight()

    b.parent = d
    f.parent = d
    d.left = b
    d.right = f
    d.updateHeight()
}
```

# 结尾

其实 AVL 树的代码量很少，这里说的是基于前面已经写好的二叉搜索树，最主要的就是弄清楚旋转的对象，和 4 种失衡情况的判断，还有就是交换节点的时候对于节点之间的关系的理解。

另外可能会有一个疑问，就是删除元素的时候，不是已经把节点删除了吗？为何还可以做后序的平衡处理，其实就是删除节点的时候，没有对节点的 parent 进行置空，且不说在 AVL 树中需要用到，即使节点依然持有 parent ，但是从 root 根节点出发的可达路径中已经到不了这个被删除节点了，那么只要 `afterRemove()` 中的代码执行完后，JVM 会自动回收它。