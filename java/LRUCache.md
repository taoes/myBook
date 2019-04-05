# 实现LRUCache缓存机制

+ 基于LinkedHashMap实现LRUCache



```java
public class LRUCache2<K, V> extends LinkedHashMap {

  private int size;

  private HashMap<K, V> map;

  public LRUCache2(int size) {
    this.size = size;
    this.map =
        new LinkedHashMap<K, V>(size, 0.75f, true) {
          @Override
          protected boolean removeEldestEntry(Map.Entry eldest) {
            return size() > size;
          }
        };
  }

  public V getValue(K key) {
    if (this.map.containsKey(key)) {
      return this.map.get(key);
    } else {
      return null;
    }
  }

  public void set(K key, V value) {
    this.map.put(key, value);
  }

  public void print() {
    this.map.forEach((k, v) -> System.out.println(k + "\t" + v));
  }
}
```



+ 测试案例

```java
public class LRUCacheDemo {

  public static void main(String[] args) {
    LRUCache2<Integer, String> cache = new LRUCache2(3);
    cache.set(1, "One");
    cache.set(2, "Two");
    cache.set(3, "Three");
    cache.print();
    System.out.println("----------------------");
    // 尝试获取，提高1和3的使用率
    cache.getValue(1);
    cache.getValue(3);
    cache.getValue(3);
    // 容器已经满了，插入4的时候会覆盖最少使用的为2
    cache.set(4, "Four");
    cache.print();
  }
}
```

