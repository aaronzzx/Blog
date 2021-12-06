# 前言
关于 **ArrayList** 我们知道它是一个动态数组，底层基于数组且可扩容，那么它是如何做到的？现在就走进源码了解一下。

本文基于 **JDK 1.8** ，将从标准的 **List** 接口方法进行了解。

在此之前先来看看几个主要的 **ArrayList** 成员变量：

```java
// 默认的初始化数组容量
private static final int DEFAULT_CAPACITY = 10;

// 节省内存用的空数组，在不同构造函数时用到
private static final Object[] EMPTY_ELEMENTDATA = {};

// 节省内存用的空数组，在不同构造函数时用到，与上面的区分开
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

// 实际装载数据的数组本身
transient Object[] elementData;

// 当前 ArrayList 所持有元素的数量（非 elementData 的数量）
private int size;
```

对于 **EMPTY_ELEMENTDATA** 和 **DEFAULTCAPACITY_EMPTY_ELEMENTDATA** 抱有疑惑的将在后面进行说明，接下来开始对常用方法进行剖析。

# 1. add(E) & add(int, E)

```java
public boolean add(E e) {
    // 扩容逻辑
    ensureCapacityInternal(size + 1);
    // 将元素追加至数组末尾，并使 size 加一
    elementData[size++] = e;
    return true;
}

public void add(int index, E element) {
    // 检查是否合法索引，对于 add 合法索引是 0<=index<=size ，对于其他操作是 0<=index<size
    rangeCheckForAdd(index);
    // 扩容逻辑
    ensureCapacityInternal(size + 1);
    // 为 index 腾出空间，需要将 index 及后面的元素向后挪动一个位置
    System.arraycopy(elementData, index, elementData, index + 1, size - index);
    // 将元素添加进数组
    elementData[index] = element;
    // size 加一
    size++;
}
```

添加的逻辑分为两种，分别是加到数组尾部和根据传入索引插入到数组中间（或头部），可以看到 `ensureCapacityInternal()` 这个方法传入了 **size + 1** ，表示最少需要 **size + 1** 的容量才能满足当前添加操作，进去看下扩容内部逻辑。

```java
private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

private static int calculateCapacity(Object[] elementData, int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}
```

`ensureCapacityInternal()` 内部可以看到又调用了 `ensureExplicitCapacity()` 和 `calculateCapacity()` ，先看一下 `calculateCapacity()` ，在 `calculateCapacity()` 中我们看到了前言中提到的空数组，如果 **elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA** 则在 **DEFAULT_CAPACITY(10)** 和 **minCapacity** 中选择一个最大的，否则直接返回 **minCapacity** 。

扩容逻辑到这里先暂停一下，现在先回过头来了解一下 **DEFAULTCAPACITY_EMPTY_ELEMENTDATA** ，通过在源码中查找，可以发现它第一次被使用是在无参构造函数中，这里将所有构造函数列出来并进行逐一解释，如下：

```java
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}

public ArrayList(Collection<? extends E> c) {
    Object[] a = c.toArray();
    if ((size = a.length) != 0) {
        if (c.getClass() == ArrayList.class) {
            elementData = a;
        } else {
            elementData = Arrays.copyOf(a, size, Object[].class);
        }
    } else {
        // replace with empty array.
        elementData = EMPTY_ELEMENTDATA;
    }
}
```

1. **ArrayList()** ：直接将 **elementData** 指向 **DEFAULTCAPACITY_EMPTY_ELEMENTDATA** ；
2. **ArrayList(int initialCapacity)** ：传入指定容量，在指定容量大于 0 时直接创建一个相应容量的数组，如果等于 0 则将 **elementData** 指向 **EMPTY_ELEMENTDATA** ，小于 0 直接抛异常；
3. **ArrayList(Collection<? extends E> c)** ：对传入集合不为 0 的情况通通存入数组，否则使 **elementData** 指向 **EMPTY_ELEMENTDATA**。

看到这里需要思考一个问题，为什么需要使用两个空数组？使用一个不行吗？这里暂时先鸽着，等走完扩容逻辑回来就都明白了。

回到 `ensureCapacityInternal()` ，我们已经分析完了 `calculateCapacity()` ，接下来就轮到 `ensureExplicitCapacity()` 了，在里面可以看到检查了 **minCapacity** ，确保它比数组容量还要大，才能走到里面的 `grow()` （真正的扩容操作）。

```java
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ? Integer.MAX_VALUE : MAX_ARRAY_SIZE;
}
```

关于 `grow()` ，除去一些处理 OOM 的情况，其实核心就是对于扩容后的容量计算，可以看到新容量就是**旧容量+(旧容量/2)**，也就是原来的 **1.5倍** ，并且如果计算出来的新容量比 **minCapacity** 还小的话就使用 **minCapacity** ，当当前数组容量太小就会出现计算后的新容量偏小，如当前容量为 0 或 1 。

**两个空数组问题**：

从 `add` 操作一路走到扩容，可以知道 **minCapacity** 要么是 **size + 1** ，要么是 **DEFAULT_CAPACITY** ，而使用 **DEFAULT_CAPACITY** 只有一种情况，就是创建 ArrayList 对象时调用的是无参构造，即当扩容时发现扩容容量太小时（小于 **DEFAULT_CAPACITY** ）直接将容量提到 **DEFAULT_CAPACITY** 这个级别，为了避免短时间内多次扩容影响效率。

而当我们自己指定容量进行创建 ArrayList 时，意味着这个操作是有意识进行的，我们不希望在容量较小的情况下直接被拔高到 **DEFAULT_CAPACITY** 。

如果只使用一个空数组就会无法区分扩容逻辑的不同处理，要么走常规的 1.5 倍扩容，要么就先将小容量提升到 **DEFAULT_CAPACITY** 。

Ps：addAll 情况其实差不多，就不加入一起了。

# 2. remove(int) & remove(Object)

```java
public E remove(int index) {
    rangeCheck(index);
    modCount++;
    E oldValue = elementData(index);
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
    return oldValue;
}

public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}

private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```

移除逻辑就比较简单了，第一种根据索引移除，其实就是删除该索引元素后，如果被删除元素不是数组末尾就需要将索引之后的所有元素向前移动一位；第二种分了两种情况，主要处理 null 和 不 null ，都是从头开始遍历数组，找到有这个元素就将其删除，当然也是可能存在数组挪动的，看 `fastRemove()` 就知道了。

# 3. get(int)

```java
public E get(int index) {
    rangeCheck(index);
    return elementData(index);
}

E elementData(int index) {
    return (E) elementData[index];
}
```

# 4. indexOf(Object)

```java
public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```

和移除差不多，分别为两种情况进行处理，然后遍历数组比较对象是否相等并返回索引，没找到返回 -1 。

# 5. clear()

```java
public void clear() {
    modCount++;
    // clear to let GC do its work
    for (int i = 0; i < size; i++)
        elementData[i] = null;
    size = 0;
}
```

遍历数组将元素置为 null ，size 赋值为 0 。

# 6. trimToSize()

```java
public void trimToSize() {
    modCount++;
    if (size < elementData.length) {
        elementData = (size == 0)
          ? EMPTY_ELEMENTDATA
          : Arrays.copyOf(elementData, size);
    }
}
```

除了扩容，当然还有缩容，ArrayList 也提供了缩容机制，只不过不是动态缩容而已，这是一个可选项。调用这个方法会将数组的长度缩短到 size 这么长。

其他的都比较简单，就不分析了。

# 结尾

可以看到，ArrayList 除了添加逻辑比较多，其他的其实都是比较常规的，毕竟核心也是动态扩容，而动态扩容就发生在添加元素的过程中。

关于 ArrayList 其实还可以优化一些，由于耗时主要在于添加删除时移动数组元素导致效率低下，因此可以改造成一个环形动态数组，增加一个 first 指针表示首元素，写了一个 Kotlin 简单版本的，如下：

```kotlin
class RingArrayList<E> : AbstractList<E> {

    companion object {
        private const val DEF_CAPACITY = 8
    }

    private var items: Array<E?>

    private var first = 0

    private var _size = 0

    override val size: Int
        get() = _size

    constructor() : this(DEF_CAPACITY)

    constructor(capacity: Int) {
        val safeCapacity = when {
            capacity <= 0 -> DEF_CAPACITY
            else -> capacity
        }
        items = arrayOfNulls<Any>(safeCapacity) as Array<E?>
    }

    override fun add(index: Int, item: E): Boolean {
        checkIndexForAdd(index)
        ensureCapacity(size + 1)
        val items = items
        val size = size
        if (index < size shr 1) {
            // index 小于 size/2 ，元素向前挪，并重置 first 指针
            for (i in 0 until index) {
                items[mapIndex(i - 1)] = items[mapIndex(i)]
            }
            first = mapIndex(-1)
        } else {
            // index 大于等于 size/2 ，元素向后挪
            for (i in (size - 1) downTo index) {
                items[mapIndex(i + 1)] = items[mapIndex(i)]
            }
        }
        items[mapIndex(index)] = item
        _size++
        return true
    }

    private fun ensureCapacity(capacity: Int) {
        val items = items
        if (capacity <= items.size) {
            return
        }
        val newCapacity = max(capacity, capacity + (capacity shr 1))
        migrate(newCapacity)
    }

    override fun remove(item: E): Boolean {
        val index = indexOf(item)
        if (index < 0) {
            return false
        }
        removeAt(index)
        return true
    }

    override fun removeAt(index: Int): E {
        checkIndex(index)
        val items = items
        val oldVal = items[mapIndex(index)]
        if (index >= size shr 1) {
            // index 大于等于 size/2 ，元素向前挪
            for (i in index until size - 1) {
                items[mapIndex(i)] = items[mapIndex(i + 1)]
            }
            items[mapIndex(size - 1)] = null
        } else {
            // index 小于 size/2 ，元素向后挪，并重置 first 指针
            for (i in index downTo 1) {
                items[mapIndex(i)] = items[mapIndex(i - 1)]
            }
            items[first] = null
            first = mapIndex(1)
        }
        _size--
        if (_size == 0) {
            first = 0
        }
        return oldVal as E
    }

    override fun set(index: Int, item: E): E {
        checkIndex(index)
        val mapped = mapIndex(index)
        val oldVal = items[mapped]
        items[mapped] = item
        return oldVal as E
    }

    override fun get(index: Int): E {
        checkIndex(index)
        return items[mapIndex(index)] as E
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

    override fun indexOf(item: E): Int {
        for (i in 0 until _size) {
            if (items[mapIndex(i)] == item) {
                return i
            }
        }
        return -1
    }

    override fun clear() {
        for (i in 0 until _size) {
            items[mapIndex(i)] = null
        }
        _size = 0
    }

    private fun mapIndex(index: Int): Int {
        var mapped = first + index
        val arraySize = items.size
        if (mapped < 0) {
            mapped += arraySize
        } else {
            mapped = if (mapped < arraySize) mapped else mapped - arraySize
        }
        return mapped
    }

    fun trimToSize() {
        migrate(size)
    }

    private fun migrate(capacity: Int) {
        if (first != 0) {
            val newArray = arrayOfNulls<Any>(capacity)
            for (i in 0 until _size) {
                newArray[i] = items[mapIndex(i)]
            }
            this.items = newArray as Array<E?>
            first = 0
        } else {
            this.items = items.copyOf(capacity)
        }
    }

    override fun toString(): String {
        return items.contentToString()
    }
}
```