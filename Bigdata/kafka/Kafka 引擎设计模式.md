# Kafka 引擎设计模式

记录阅读kafka源码过程见到的设计模式

## Kafka Client 缓存的代理模式

这里用到代理模式主要是要实现一个**线程安全的LRU缓存**。

既然是代理模式，很显然会有一个Cache接口，定义基本的方法。

```java
public interface Cache<K, V> {
    V get(K key);
    void put(K key, V value);
    boolean remove(K key);
    long size();
}
```

接口好了，就需要有真实的角色去实现这个接口，也就是LRU缓存。

```java
public class LRUCache<K, V> implements Cache<K, V> {
    private final LinkedHashMap<K, V> cache;

    public LRUCache(final int maxSize) {
        cache = new LinkedHashMap<K, V>(16, .75f, true) {// 直接使用LinkedHashMap来实现LRU
            @Override
            protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                return this.size() > maxSize; // require this. prefix to make lgtm.com happy
            }
        };
    }

    @Override
    public V get(K key) {
        return cache.get(key);
    }

    @Override
    public void put(K key, V value) {
        cache.put(key, value);
    }

    @Override
    public boolean remove(K key) {
        return cache.remove(key) != null;
    }

    @Override
    public long size() {
        return cache.size();
    }
}
```

到这里其实已经可以使用了LRU缓存了，但是还存在线程安全问题，这时就需要我们代理角色来帮我们实现同步，只需要很简单的加上同步关键字synchronized就可以了。

```java
public class SynchronizedCache<K, V> implements Cache<K, V> {
    private final Cache<K, V> underlying;	// 真实角色

    public SynchronizedCache(Cache<K, V> underlying) {
        this.underlying = underlying;
    }

    @Override
    public synchronized V get(K key) {
        return underlying.get(key);
    }

    @Override
    public synchronized void put(K key, V value) {
        underlying.put(key, value);
    }

    @Override
    public synchronized boolean remove(K key) {
        return underlying.remove(key);
    }

    @Override
    public synchronized long size() {
        return underlying.size();
    }
}
```

很经典的代理模式，可以直接运用到以后的工作中。

