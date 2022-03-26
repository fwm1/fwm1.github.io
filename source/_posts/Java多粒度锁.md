---
title: Java细粒度锁
date: 2022-03-24 20:38:21
tags: [lock]
categories: java
---

### 从WeakHashMap实现分段锁

当弱引用指向的对象引用被释放之后 Java 会在下一次的 GC 将这弱引用指向的对象回收掉，在经过 GC 之后，当弱引用指向的对象被回收时，弱引用将会进入创建时指定的队列。

WeakHashMap使用**弱引用**作为 key，通过轮询referenceQueue清理被gc的弱引用对象。

```java
private void expungeStaleEntries() {
        for (Object x; (x = queue.poll()) != null; ) {
            synchronized (queue) {
                @SuppressWarnings("unchecked")
                    Entry<K,V> e = (Entry<K,V>) x;
                int i = indexFor(e.hash, table.length);

                Entry<K,V> prev = table[i];
                Entry<K,V> p = prev;
                while (p != null) {
                    Entry<K,V> next = p.next;
                    if (p == e) {
                        if (prev == e)
                            table[i] = next;
                        else
                            prev.next = next;
                        // Must not null out e.next;
                        // stale entries may be in use by a HashIterator
                        e.value = null; // Help GC
                        size--;
                        break;
                    }
                    prev = p;
                    p = next;
                }
            }
        }
    }
```



为什么不能直接用WeakHashMap实现细粒度锁：线程不安全。

替代方案：

1. Collections.synchronizedMap 性能差
2. ConcurrentReferenceHashMap

### ConcurrentReferenceHashMap实现细粒度锁

```java
public class WeakHashLock<T> {
    private final ConcurrentReferenceHashMap<T, ReentrantLock> referenceHashMap;

    public WeakHashLock() {
        this.referenceHashMap = new ConcurrentReferenceHashMap<>(
                16, 0.75f, 16, ConcurrentReferenceHashMap.ReferenceType.WEAK);
    }

    public ReentrantLock get(T key) {
        return this.referenceHashMap.computeIfAbsent(key, lock -> new ReentrantLock());
    }

}
```

### Guava Striped实现细粒度锁

```java
Striped<Lock> lockStriped = Striped.lazyWeakLock(1024);
```

Striped能保证hashcode相等的对象拿到的是同一个锁，但是**hashcode不同的对象也可能拿到同一把锁**，由trip个数决定并发度。

