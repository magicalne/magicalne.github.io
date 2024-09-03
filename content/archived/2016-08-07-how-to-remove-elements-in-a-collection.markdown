---
layout: post
title: "How to remove elements in a collection"
categories: java, source code, collections, gc
date: 2016-08-07
---
前不久在微信公众号中看到一篇技术文章，内容是讨论“java中如何删除一个集合的多个元素”。

文章并没有把这个问题挖深，它的结论是，如果直接使用ArrayList，然后用forEach两层循环遍历，会报ConcurrentModificationException，尽管测试代码时单线程。原因是java的forEach使用iterator实现的循环遍历，而iterator在遍历的过程中是不允许删除元素的。也不能用传统for循环，因为使用for循环删除元素的过程中，数组下标有变动。

以上是文章的结论，现在呢，到底“删除集合的的多个元素”哪家强？

结论：

1. 最简单的就是removeAll()
2. 可以使用CopyOnWriteArrayList

我的测试代码：

```java
public class RemoveElementsInList {

    public void remove1(List<Integer> all, List<Integer> sub) {
        for (int i = 0; i < all.size(); i ++) {
            for (int j = 0; j < sub.size(); j ++) {
                if (all.get(i).equals(sub.get(j))) {
                    all.remove(i);
                }
            }
        }
        System.out.println(all);
    }

    public static void main(String[] args) {
        List<Integer> sub = new ArrayList<>(Arrays.asList(1,2,3));
        List<Integer> all = new ArrayList<>(Arrays.asList(1,2,3,4,5,6,7));
        all.removeAll(sub);
        List<Integer> sub1 = new CopyOnWriteArrayList<>(sub);
        List<Integer> all1 = new CopyOnWriteArrayList<>(all);
        RemoveElementsInList removeElementsInList = new RemoveElementsInList();
        removeElementsInList.remove1(all1, sub1);
    }
}
```

## 下面看一看ArrayList的removeAll源码

```java
public boolean removeAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, false);
    }

private boolean batchRemove(Collection<?> c, boolean complement) {
        final Object[] elementData = this.elementData;
        int r = 0, w = 0;
        boolean modified = false;
        try {
            for (; r < size; r++)
                if (c.contains(elementData[r]) == complement)
                    elementData[w++] = elementData[r];
        } finally {
            // Preserve behavioral compatibility with AbstractCollection,
            // even if c.contains() throws.
            if (r != size) {
                System.arraycopy(elementData, r,
                                 elementData, w,
                                 size - r);
                w += size - r;
            }
            if (w != size) {
                // clear to let GC do its work
                for (int i = w; i < size; i++)
                    elementData[i] = null;
                modCount += size - w;
                size = w;
                modified = true;
            }
        }
        return modified;
    }
```

这样的jdk源码，每个实现都是一道面试题。先抛出几个问题：

1. 为什么要创建一个elementData引用，而不是直接在this.elementData上操作？
2. removeAll的时间复杂度是多少？
3. "// clear to let GC do its work"都做了些啥？

回答：

1. 不知道。。。
2. O(n^2)
3. defragmentation。 手动设置引用为null，好让gc工作，使得内存中的碎片合并，腾出足以容纳大对象的空间。

我不确定1的答案是否跟3有关，反推一下，貌似也就是为了gc了吧。有待研究。

## 接下来，我们看看CopyOnWriteArrayList的removeAll方法的实现

```java
public boolean removeAll(Collection<?> c) {
        if (c == null) throw new NullPointerException();
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            if (len != 0) {
                // temp array holds those elements we know we want to keep
                int newlen = 0;
                Object[] temp = new Object[len];
                for (int i = 0; i < len; ++i) {
                    Object element = elements[i];
                    if (!c.contains(element))
                        temp[newlen++] = element;
                }
                if (newlen != len) {
                    setArray(Arrays.copyOf(temp, newlen));
                    return true;
                }
            }
            return false;
        } finally {
            lock.unlock();
        }
    }
```

忽略锁的部分不看，逻辑还是很简单的，就是将结果保存到新创建的数组中。

CopyOnWriteArrayList的remove方法与removeAll类似，都是创建一个临时数组，将修改的内容复制到这个数组中，最后将这个数组的引用赋值给this引用。
