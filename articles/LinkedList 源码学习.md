# 前言

上一篇文章了解了 **ArrayList** 内部的工作逻辑，这一篇接着学习 **LinkedList** 。数据结构的入门学习可以知道数组和链数据结构是编程中最基本的物理结构，队列、栈、树等结构都是在这之上延伸出来的逻辑概念，数组代表的是连续的地址空间，在初始化阶段就必须指定空间大小；而今天的主角——链表，则是用到则“链接”，不多占用你一点内存，现在来了解一下内部是怎么做到的。

本文基于 **JDK 1.8** 的 **LinkedList** ，通过标准 List 接口进行学习。

简单了解下 **LinkedList** 内部的几个成员：

```java
transient int size = 0;

// 头节点
transient Node<E> first;

// 尾节点
transient Node<E> last;

// 通过前驱和后继明显可以知道这是一个双向链表
private static class Node<E> {
    E item; // 元素
    Node<E> next; // 后继
    Node<E> prev; // 前驱
    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

# 1. add(E) & add(int, E)

先来看下添加逻辑，由于 `add(E)` 逻辑在 `add(int, E)` 中也有处理，因此直接来看 `add(int, E)` 。

```java
public void add(int index, E element) {
    checkPositionIndex(index);
    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index));
}
```

逻辑非常简单，除去索引越界判断（下文不再单独聊越界判断），判断索引是否等于 size ，等于表示追加到链表尾部，否者就是插入中间（或最前面）。

先来看追加到尾部的处理逻辑 `linkLast()` ：

```java
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```

由于是追加到尾部，因此需要拿到尾节点，并使待插入节点（**newNode**）的前驱指向当前尾节点，然后让待插入节点（**newNode**）成员新的尾节点；接下来判断 **l** 是否为空，如果为空表示当前链表没有元素，需要将 **first** 头节点也指向 **newNode** ，不然就是 **l** 的后继指向 **newNode** ，最后 **size++** ，**modCount** 代表着修改次数，与迭代相关，这里就不牵扯进来了。

回到 `add(int, E)` ，接着再看 `linkBefore()` ，可以看到 `linkBefore()` 传入了两个参数，第一个表示插入位置的索引，第二个表示当前 **index** 位置的节点，来看看 `node()` 方法是如何获取一个指定位置的节点的：

```java
Node<E> node(int index) {
    // assert isElementIndex(index);
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

可以看到是采用的遍历的方式，其实也正常，链表的每个节点都是通过链串起来的，如果想找指定位置的元素那就只能遍历了，这里的逻辑由于是双向链表，因此可以通过判断 **index** 处在 **size** 范围内的左半部分还是右半部分，然后分别从头节点向后遍历一半或从尾节点向前遍历一半，这里对遍历的优化虽然还是 **O(n)** 的复杂度，但在实际的查找过程中是可以省去一半的遍历时间的。

`node()` 方法了解完后就轮到 `linkBefore()` 了，接着往下看：

```java
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```

这里断定了传入的 **succ** 后继不为空，因为在 `add` 方法时已经进行了索引检查，不合法的直接抛出异常，而剩下的都是合法索引，也就意味着一定能获取到某个位置的节点，废话不多说，往下看逻辑。

首先是取出了 **succ** 的前驱 **pred** ，接着创建新节点 **newNode** ，将新节点的前驱和后继分别指向 **pred** 和 **succ** ，然后让新节点成员 **succ** 的前驱，按链表来表示差不多是这样：

- Before ：pred <--> succ
- After ：pred <-- newNode <--> succ

逻辑走到这里元素就被插入到 **succ** 的 before 了，但还没完，如果取出来的 **pred** 是空的，意味着 **succ** 是头节点，那么 **newNode** 还得成为新的头节点，让 **first** 指向 **newNode** ；如果不为空则 **pred** 的后继指向 **newNode** 然后 **size++** 。

# 2. remove(int) & remove(Object)

```java
public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}

public boolean remove(Object o) {
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
```

`remove(int)` 比较简单，`node()` 我们已经知道了它的逻辑，而 `unlink()` 在 `remove(Object)` 中也出现了，因此直接看 `remove(Object)` 👇。

`remove(Object)` 中主要是对空值进行了特别处理，除此之外就是对整个链表进行遍历，找到与 **o** 相等的元素，然后 `unlink()` 它，如果找到那么循环就可以直接退出了，返回 true ，接着看 `unlink()` 👇：

```java
E unlink(Node<E> x) {
    // assert x != null;
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;
    if (prev == null) {
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }
    if (next == null) {
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }
    x.item = null;
    size--;
    modCount++;
    return element;
}
```

`unlink()` 看起来比前面的方法都要长一些，但其实逻辑也很简单，首先取出 **x** 的旧元素，因为最后要返回旧元素，接着取出 **x** 的前驱和后继，开始删除操作：

```java
if (前驱为空)
    表示 x 是头节点，那么需要将 first 指向 x 的后继 next
else
    x 不是头节点，直接将 x 的前驱（prev）的后继指向 x 的后继（next）
    x 前驱置空
    
if (后继为空)
    表示 x 是尾节点，需要将 last 指向 x 的前驱 prev
else
    x 不是尾节点，将 x 的后继（next）的前驱指向 x 的前驱（prev）
    x 后继置空    
```

- Before ：prev <--> x <--> next
- After ：prev <--> next

最后将 **x** 的元素置空然后 **size--** 。

# 3. get(int)

```java
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}
```

通过 `node()` 方法获取节点并将元素返回

# 4. indexOf(Object)

```java
public int indexOf(Object o) {
    int index = 0;
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null)
                return index;
            index++;
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item))
                return index;
            index++;
        }
    }
    return -1;
}
```

区分空值的遍历逻辑，在 `remove(Object)` 中已经见过了。

# 5. clear()

```java
public void clear() {
    // Clearing all of the links between nodes is "unnecessary", but:
    // - helps a generational GC if the discarded nodes inhabit
    //   more than one generation
    // - is sure to free memory even if there is a reachable Iterator
    for (Node<E> x = first; x != null; ) {
        Node<E> next = x.next;
        x.item = null;
        x.next = null;
        x.prev = null;
        x = next;
    }
    first = last = null;
    size = 0;
    modCount++;
}
```

清空元素，**first** 和 **last** 都置空，一般来说这样就可以了，因为 GC Root 的线已经断掉了，但上面还是遍历链表将所有节点的元素前驱后继关系清空，原因就是 2 点，上面注释说了：

1. 如果丢弃的节点存在于多代，则有助于分代 GC
2. 即使在迭代过程中也要确保内存先释放掉

# 结尾

在学习过程中可以发现添加元素时总是需要针对首尾节点执行不同的逻辑，那能不能统一一下呢？其实使用哨兵节点就可以优化这种问题。

简单来说，哨兵节点它是空节点，不持有元素只持有链接关系，为了使链表“永不为空”，这样我们就不需要对边界问题做特别处理了，这里用 **Kotlin** 写了一个简单的哨兵节点版本，如下：

```kotlin
class LinkedList<E> : AbstractList<E>() {

    private var first: Node<E>
    private var last: Node<E>

    override var size: Int = 0
        private set

    init {
        val f = Node<E>(null, null, null)
        val l = Node<E>(null, null, null)
        f.next = l
        l.prev = f
        first = f
        last = l
    }

    override fun add(index: Int, item: E): Boolean {
        checkIndexForAdd(index)
        val node = if (index == size) last else node(index)
        linkBefore(node, item)
        return true
    }

    private fun linkBefore(node: Node<E>, item: E) {
        val prev = node.prev
        val newNode = Node(item, prev, node)
        prev?.next = newNode
        node.prev = newNode
        size++
    }

    override fun remove(item: E): Boolean {
        val index = indexOf(item)
        if (index < 0) {
            return false
        }
        val node = node(index)
        return unlink(node) != null
    }

    override fun removeAt(index: Int): E {
        return unlink(node(index))
    }

    private fun unlink(node: Node<E>): E {
        val oldVal = node.item
        val prev = node.prev
        val next = node.next
        prev?.next = next
        next?.prev = prev
        size--
        return oldVal as E
    }

    override fun set(index: Int, item: E): E {
        val node = node(index)
        val oldVal = node.item
        node.item = item
        return oldVal as E
    }

    override fun get(index: Int): E {
        return node(index).item as E
    }

    override fun indexOf(item: E): Int {
        for (i in 0 until size) {
            if (node(i).item == item) {
                return i
            }
        }
        return -1
    }

    private fun node(index: Int): Node<E> {
        checkIndex(index)
        var node: Node<E>?
        if (index < size shr 1) {
            node = first.next
            for (i in 0 until index) {
                node = node?.next
            }
            return node!!
        }
        node = last.prev
        for (i in (size - 1) downTo (index + 1)) {
            node = node?.prev
        }
        return node!!
    }

    private fun checkIndexForAdd(index: Int) {
        if (index < 0 || index > size) {
            throw IndexOutOfBoundsException(errorMsg(index))
        }
    }

    private fun checkIndex(index: Int) {
        if (index < 0 || index >= size) {
            throw IndexOutOfBoundsException(errorMsg(index))
        }
    }

    private fun errorMsg(index: Int): String {
        return "Index: $index, Size: $size"
    }

    override fun clear() {
        first.next = last
        last.prev = first
        size = 0
    }

    override fun toString(): String {
        if (isEmpty()) {
            return "[]"
        }
        val sb = StringBuilder().also {
            it.append("[")
        }
        for (i in 0 until size) {
            val item = node(i).item
            sb.append("$item")
            if (i == size - 1) {
                sb.append("]")
            } else {
                sb.append(", ")
            }
        }
        return sb.toString()
    }

    private class Node<E>(
        var item: E?,
        var prev: Node<E>?,
        var next: Node<E>?
    )
}
```