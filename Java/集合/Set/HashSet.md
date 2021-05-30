# Set

## HashSet

### 基本属性

```java
// 基于HashMap实现，把hashset组合进来
private transient HashMap<E,Object> map;

// key是hashset的key，这个作为hashmap的value
private static final Object PRESENT = new Object();
```

### 初始化

```java
// 默认构造方法
public HashSet() {
    map = new HashMap<>();
}

// 给定数据初始化
public HashSet(Collection<? extends E> c) {
    // 初始容量取括号中两个数的最大值（期望的值 / 0.75+1，默认值 16）
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    addAll(c);
}

// 给定初始容量和扩容加载因子
public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
}

// 给定初始容量
public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
}
```



