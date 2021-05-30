## 零拷贝

### 堆内存和直接内存

#### 什么是直接内存

要说零拷贝，那我们先要知道什么是直接内存和堆内存。所谓**堆内存**就是分配给JVM堆区的内存，而**直接内存**是java通过调用navite方法可以直接分配JVM之外的内存区域，所以我们也把这块内存叫做**堆外内存**。而我们可以在堆区通过一个`DirectByteBuffer`去引用这块内存。因为这块内存是没有交给JVM去管理的，比较容易发生内存溢出，为了避免一直没有FULL GC，导致耗完物理内存，我们可以通过参数`-XX：MaxDirectMemorySize` 指定直接内存的大小，这样当直接内存到达一个阈值的时候，就会被动触发Full GC。直接内存和堆内存的关系如下所示：

![](img\直接内存和堆内存.png)



JDK给我们提供了Api可以查看默认的堆外内存大小：

```java
System.out.println("direct memory : "+ VM.maxDirectMemory() / 1024 / 1024);
System.out.println("heap memory : "+ Runtime.getRuntime().maxMemory() / 1024 / 1024);
```

#### 直接内存如何申请使用

那么了解了什么是直接内存，我们接着看一下如何使用直接内存，代码如下：

```java
// 通过这个方法我们可以申请1000字节的直接内存
ByteBuffer buffer = ByteBuffer.allocateDirect(1000);

// 跟踪这个方法到DirectByteBuffer.java:120行，我们看到他是调用了一个navite方法去申请的内存，返回来的是一个地址
// 如果有openJDK的代码，可以看到这个方法底层的代码
base = unsafe.allocateMemory(size);

// 这是navite方法调用操作系统的方法为我们申请内存  unsafe.cpp:628
void* x = os::malloc(sz, mtInternal);
```

#### 直接内存和堆内存对比

下面这段代码通过分别对堆内存和直接内存的反复申请读写，测试了一下耗时，发现直接内存的访问性能普遍高于堆内存的访问性能，这是因为使用直接内存只需要一次拷贝，而使用堆内存需要两次拷贝，详情参考下文零拷贝。而直接内存的申请效率普遍低于堆内存的申请。

```java
public class MemoryTest {

    /**
     * 操作堆内存
     */
    private static void heapAccess() {
        long startTime = System.currentTimeMillis();
        // 分配堆内存
        ByteBuffer buffer = ByteBuffer.allocate(1000);
        readAndWrite(buffer);
        long endTime = System.currentTimeMillis();
        System.out.println("堆内存访问:" + (endTime - startTime) + "ms");
    }

    /**
     * 操作直接内存
     */
    private static void directAccess() {
        long startTime = System.currentTimeMillis();
        // 分配直接内存
        ByteBuffer buffer = ByteBuffer.allocateDirect(1000);
        readAndWrite(buffer);
        long endTime = System.currentTimeMillis();
        System.out.println("直接内存访问:" + (endTime - startTime) + "ms");
    }

    private static void readAndWrite(ByteBuffer buffer) {
        for (int i = 0; i < 100000; i++) {
            for (int j = 0; j < 200; j++) {
                buffer.putInt(j);
            }
            buffer.flip();
            for (int j = 0; j < 200; j++) {
                buffer.getInt();
            }
            buffer.clear();
        }
    }

    private static void heapAllocate() {
        long startTime = System.currentTimeMillis();
        for (int i = 0; i < 100000; i++) {
            ByteBuffer.allocate(100);
        }
        long endTime = System.currentTimeMillis();
        System.out.println("堆内存申请:" + (endTime - startTime) + "ms");
    }

    private static void directAllocate() {
        long startTime = System.currentTimeMillis();
        for (int i = 0; i < 100000; i++) {
            ByteBuffer.allocateDirect(100);
        }
        long endTime = System.currentTimeMillis();
        System.out.println("直接内存申请:" + (endTime - startTime) + "ms");
    }

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            heapAccess();
            directAccess();
        }

        System.out.println("---------------------------");

        for (int i = 0; i < 10; i++) {
            heapAllocate();
            directAllocate();
        }
    }
}

// 执行结果
堆内存访问:54ms
直接内存访问:151ms
堆内存访问:84ms
直接内存访问:41ms
堆内存访问:78ms
直接内存访问:40ms
堆内存访问:83ms
直接内存访问:40ms
堆内存访问:99ms
直接内存访问:49ms
堆内存访问:166ms
直接内存访问:55ms
堆内存访问:123ms
直接内存访问:54ms
堆内存访问:102ms
直接内存访问:49ms
堆内存访问:79ms
直接内存访问:39ms
堆内存访问:78ms
直接内存访问:40ms
---------------------------
堆内存申请:13ms
直接内存申请:71ms
堆内存申请:6ms
直接内存申请:23ms
堆内存申请:23ms
直接内存申请:29ms
堆内存申请:2ms
直接内存申请:28ms
堆内存申请:7ms
直接内存申请:77ms
堆内存申请:3ms
直接内存申请:26ms
堆内存申请:2ms
直接内存申请:21ms
堆内存申请:213ms
直接内存申请:41ms
堆内存申请:2ms
直接内存申请:40ms
堆内存申请:2ms
直接内存申请:38ms
```

### 零拷贝

![](img\零拷贝.png)

何为零拷贝呢，不是说一次拷贝都没有，而是说相比较操作堆内存少了两次拷贝。平时发生一次读写，如果是操作堆内存的话，往往需要四次拷贝，读的时候将数据从直接内存拷贝到堆内存，再从堆内存拷贝到缓冲区，写的时候将数据从缓冲区拷贝到堆内存，在从堆内存拷贝到直接内存。而操作直接内存就不一样了，少去了堆内存和直接内存之间的拷贝过程。我们把这个叫做**零拷贝**。而Netty接受和发送数据用了直接内存来提升效率。

