### API 和 类结构

* 先看下bytebuf里面的一些主要的api：

```java
<pre>
 *     文档里面有一些描述，可见bytebuf会有一些游标，比如readerIndex:表示可读的开始
     	writerIndex:表示可写的开始，capacity:表示容量。
 *      +-------------------+------------------+------------------+
 *      | discardable bytes |  readable bytes  |  writable bytes  |
 *      |                   |    (CONTENT)可读  |      可写        |
 *      +-------------------+------------------+------------------+
 *      |                   |                  |                  |
 *      0      <=      readerIndex   <=   writerIndex    <=    capacity
 * </pre>
public abstract class ByteBuf implements ReferenceCounted, Comparable<ByteBuf> {

    /**
     * 获取容量
     */
    public abstract int capacity();
     /**
     * 最大容量 因为bytebuf有个最大容量值
     */
    public abstract int maxCapacity();
    /**
     * 分配
     */
    public abstract ByteBufAllocator alloc();
     /**
     * 获取可读开始索引
     */
    public abstract int readerIndex();
     /**
     * 读取bytebuf 并且设置当前可是读位置
     */
    public abstract ByteBuf readerIndex(int readerIndex);
	 /**
     * 获取写索引
     */
    public abstract int writerIndex();
    /**
     * 当前可读的长度：wirteIndex - readIndex
     */
    public abstract int readableBytes();
    /**
     * 当前可写的长度：capacity-writeIndex
     */
    public abstract int writableBytes();
     * 这个就是会记录你当前的readerIndex
     */
    public abstract ByteBuf markReaderIndex();
    /**
     * 这个一般是在上面那个方法记录后，在读取数据完后，对readerIndex进行复原，当然还有write也可以，就
     * 不在累赘了
     */
    public abstract ByteBuf resetReaderIndex();
    
    // 获取指定索引的数据 这个方法readerIndex位置是不会做修改的
    public abstract byte  getByte(int index);
     /**
     * 读取一个byte,读索引会加1 ，readShort 索引就+2，int+4，就不累赘了，后面还有很多read方法的。
     * 具体看下api和实现就好了
     */
    public abstract byte  readByte();
    /**
     * 写一个byte writeIndex+1，writeShort writeIndex+2,同样很多write方法也不累赘了
     */
    public abstract ByteBuf writeByte(int value);
}
```

上面的read write set这些api,主要要注意的是index的改变情况。



* 再看一个上面api实现的一个脚手架吧，就是这抽象类，`io.netty.buffer.AbstractByteBuf`，这个ByteBuf大部分api都有实现，当然有些交给子类去实现。

  

* 看下类的继承关系

 ![bytebuf1](https://github.com/tryingpfq/tryingpfq.github.io/blob/master/picture/bytebuf1.jpg?raw=true)

这个图要先有个印象，最好能够记住，后面有些地方，会根据这个继承关系读来做说明的。



### netty 内存类别

这个问题想必面试的时候，问的也比较多，笼统的回答就是，堆内存和堆外内存。但我们可以在细分一下，其实看上面那个类图应该大概就能看的出，ByteBuf是一个抽下类，提供了一些基础性的api,而AbstractByteBuf则就是ByteBuf的一个基础实现的脚手架，接着下面这层，就是更加类别的一个实现类。所以我们可以从三个维度进行分类。

#### pooled和upooled

pooled是池化的，也就是对应的实现类，在分配byteBuf内存的时候，会从**预先分配好**的内存取一段连续内存，而upooled是非池化的，每次进行内存分配的时候，直接调用系统API，向**操作系统**申请一块内存。

从这维度，就只有两种，一个是pooled，另一个就是unpooled。

 #### unsafe 和 非unsafe

在上面的实现中，没有标明的就是非unsafe。在jdk中，有个unsafe对象，直接可以拿到对象的一个内存地址，再基于内存地址直接进行操作，相比速度会比较快一些。在这里，unsafe也就是直接操作分配的内存的地址，非unsafe是基于内存数据数组角标的一个访问(基于数组角标的访问，最后也是要算出内存地址的)。

可以简单看下getByte(),这两种类型不同的实现。

> unsafe
>
> buf的内存地址+偏移量 直接范围
>
> 非unsafe
>
> buf数组 + 下表
>
> 这里的buf，可能是ByteBuf 或者jdk的bytebuffer

`io.netty.buffer.PooledUnsafeHeapByteBuf#_getByte`

```java
  @Override
    protected byte _getByte(int index) {
        return UnsafeByteBufUtil.getByte(memory, idx(index));
    }

	 static byte getByte(byte[] array, int index) {
        return PlatformDependent.getByte(array, index);
    }

	public static byte getByte(byte[] data, int index) {
        return PlatformDependent0.getByte(data, index);
    }

	static byte getByte(byte[] data, int index) {
        //你看这里就是直接拿到内存地址了,并调用unsafe对象获取
        return UNSAFE.getByte(data, BYTE_ARRAY_BASE_OFFSET + index);
    }

```

`io.netty.buffer.PooledHeapByteBuf#_getByte`

```java
 @Override
    protected byte _getByte(int index) {
        return HeapByteBufUtil.getByte(memory, idx(index));
    }

  static byte getByte(byte[] memory, int index) {
      	// 你看这里就是直接数组下标
        return memory[index];
    }
```



#### Heap 和 Direct

这两个应该是平时用的最多的一个，看名字也应该知道意思，heap就是在jvm上进行分配的，Direct则是调用jdk的api进行分配的，这个内存是不被jvm控制的，所以不参与gc的，需要手动释放的，所以要注意这个问题，以防出现内存问题。继续看下这两个的实现代码。

`io.netty.buffer.UnpooledHeapByteBuf`

```java
byte[] array;// 这里是基于byte数组的
```

`io.netty.buffer.UnpooledDirectByteBuf`

```java
// 这个是jdk的buffer,堆外内存
ByteBuffer buffer; // accessed by UnpooledUnsafeNoCleanerDirectByteBuf.reallocateDirect()

// 可以看下这个分配过程
io.netty.buffer.Unpooled#directBuffer(int)
  
 public static ByteBuf directBuffer(int initialCapacity) {
        return ALLOC.directBuffer(initialCapacity);
    }

@Override
    public ByteBuf directBuffer(int initialCapacity, int maxCapacity) {
        if (initialCapacity == 0 && maxCapacity == 0) {
            return emptyBuf;
        }
        validate(initialCapacity, maxCapacity);
        return newDirectBuffer(initialCapacity, maxCapacity);
    }

 @Override
    protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
        final ByteBuf buf;
        if (PlatformDependent.hasUnsafe()) {
            buf = noCleaner ? new InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf(this, initialCapacity, maxCapacity) :
                    new InstrumentedUnpooledUnsafeDirectByteBuf(this, initialCapacity, maxCapacity);
        } else {
            buf = new InstrumentedUnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
        }
        return disableLeakDetector ? buf : toLeakAwareBuffer(buf);
    }

 protected ByteBuffer allocateDirect(int initialCapacity) {
     // 这里面就是jdk分配了
        return ByteBuffer.allocateDirect(initialCapacity);
    }
```

分类就讲到这里了，后面还会对这些类型进行分析。



### 内存分配器

同样，可以先看下分配器的主要API

```java
public interface ByteBufAllocator {

    ByteBufAllocator DEFAULT = ByteBufUtil.DEFAULT_ALLOCATOR;

    /**
     * Allocate a {@link ByteBuf}. If it is a direct or heap buffer
     * depends on the actual implementation.
     * 看下英文注意 就是是堆内还是堆外内存，依靠的是具体的实现
     * 重载的方法就不看了，可能就是指定大小了
     */
    ByteBuf buffer();
    
    /**
     * Allocate a {@link ByteBuf}, preferably a direct buffer which is suitable for I/O.
     */
    ByteBuf ioBuffer();
    
     /**
     * Allocate a heap {@link ByteBuf}.
     */
    ByteBuf heapBuffer();
     /**
     * Allocate a direct {@link ByteBuf}.
     */
    ByteBuf directBuffer();
     /**
     * Allocate a {@link CompositeByteBuf}.
     * If it is a direct or heap buffer depends on the actual implementation.
     * 日常用的并不多
     */
    CompositeByteBuf compositeBuffer();

}
```

看完上面，相比基本的有了解，对于直接内存还是堆内内存，是依靠具体的实现来完成。看到上面，好像只看到区分一个维度那就是堆内和直接内存的区分，并没有看到pooled和unpooled,以及unsafe和非unsafe，netty是怎么区分的呢？看后面的分析，会有提到。

可以下看下这个抽象的实现。`io.netty.buffer.AbstractByteBufAllocator`，大部分功能都有实现。

```java
 	//这里就是依靠具体的实现 来区分是heap还是direct
	@Override
    public ByteBuf buffer() {
        if (directByDefault) {
            return directBuffer();
        }
        return heapBuffer();
    }

//看下directBuffer(),
 @Override
    public ByteBuf directBuffer(int initialCapacity, int maxCapacity) {
        if (initialCapacity == 0 && maxCapacity == 0) {
            return emptyBuf;
        }
        validate(initialCapacity, maxCapacity);
        //会调用这个方法来分配，这个其实是一个抽象方法，也是依靠子类的具体实现
        //有pooled 和 unpooled
        return newDirectBuffer(initialCapacity, maxCapacity);
    }
	
	//看下这两个抽象方法，其实现有pooled 和 unpooled,
	//可见 pooled和unpooled这个维度的区分，也是要依靠子类具体实现类
	protected abstract ByteBuf newHeapBuffer(int initialCapacity, int maxCapacity);
    protected abstract ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity);

```

* UML图
![bytebuf2](https://github.com/tryingpfq/tryingpfq.github.io/blob/master/picture/bytebuf2.jpg?raw=true)

  这个类结构继承关系还是比较简单一下，其实主要就是两个具体的实现Unpooled和pooled。其实这两个主要就是实现上面说的那两个抽象方法。unpooled直接是调用系统api进行分配， pooled分配是直接从一个预先分配好的内存(相当于是一个池子)中进行分配。那看到这，是不是还有一个unsafe这个维度没有区分，这里是netty自动区分的，也就是说如果底层可以拿到unsafe这个对象，就会分配unsafe,否则就是非unsafe，这里我们可以简单看下代码。

  `io.netty.buffer.UnpooledByteBufAllocator`

  ```java
   @Override
      protected ByteBuf newHeapBuffer(int initialCapacity, int maxCapacity) {
          return PlatformDependent.hasUnsafe() ?
                  new InstrumentedUnpooledUnsafeHeapByteBuf(this, initialCapacity, maxCapacity) :
                  new InstrumentedUnpooledHeapByteBuf(this, initialCapacity, maxCapacity);
      }
  //这里要注意点一点是，这个分配的返回还是一个ByteBuf对象，但是里面持有的是jdk
  //一个 java.nio.DirectByteBuffer#DirectByteBuffer(int)
   @Override
      protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
          final ByteBuf buf;
          if (PlatformDependent.hasUnsafe()) {
              buf = noCleaner ? new InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf(this, initialCapacity, maxCapacity) :
                      new InstrumentedUnpooledUnsafeDirectByteBuf(this, initialCapacity, maxCapacity);
          } else {
              buf = new InstrumentedUnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
          }
          return disableLeakDetector ? buf : toLeakAwareBuffer(buf);
      }
  
  // 其实很明显了，就是看操作系统底层有没有unsafe对象
  //PlatformDependent.hasUnsafe()
   public static boolean hasUnsafe() {
          return UNSAFE_UNAVAILABILITY_CAUSE == null;
      }
  ```

  后面就具体分析这两个实现类，pooled和unpooled

  

#### UnpooledByteBufAllocator

 `io.netty.buffer.UnpooledByteBufAllocator`

* Heap的分配

  `io.netty.buffer.UnpooledByteBufAllocator#newHeapBuffer`

  ```java
  //对于heap上的分配，分配器会调用这个方法。
  	@Override
      protected ByteBuf newHeapBuffer(int initialCapacity, int maxCapacity) {
          return PlatformDependent.hasUnsafe() ?
                  new InstrumentedUnpooledUnsafeHeapByteBuf(this, initialCapacity, maxCapacity) :
                  new InstrumentedUnpooledHeapByteBuf(this, initialCapacity, maxCapacity);
          //其实上面是通过调用子类的构造
      }
  
  // 只能跟着构造进去看了 InstrumentedUnpooledHeapByteBuf
  // 最后会进到这个父类中
    public UnpooledHeapByteBuf(ByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
          super(maxCapacity);
  
          if (initialCapacity > maxCapacity) {
              throw new IllegalArgumentException(String.format(
                      "initialCapacity(%d) > maxCapacity(%d)", initialCapacity, maxCapacity));
          }
  
          this.alloc = checkNotNull(alloc, "alloc");
        	//这里面，其实就是new 一个数组，在堆中，并且初始化大小
          setArray(allocateArray(initialCapacity));
        	// 设置索引
          setIndex(0, 0);
      }
  // 对于unasfe 和 非unsafe,这里就不再累赘了，上面其实有说到，你进去看下_getByte()方法就行，一个是直接操作内存地址，一个是操作数组
  ```

* Directd分配

  `io.netty.buffer.UnpooledByteBufAllocator#newDirectBuffer`

  ```java
  	//对于Direct上的分配，分配器会调用这个方法。
  	@Override
      protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
          final ByteBuf buf;
          if (PlatformDependent.hasUnsafe()) {
              buf = noCleaner ? new InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf(this, initialCapacity, maxCapacity) :
                      new InstrumentedUnpooledUnsafeDirectByteBuf(this, initialCapacity, maxCapacity);
          } else {
              buf = new InstrumentedUnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
          }
          return disableLeakDetector ? buf : toLeakAwareBuffer(buf);
      }
  
  // 同样，我们看下这个InstrumentedUnpooledDirectByteBuf
  // 会进入这个构造方法。
   public UnpooledDirectByteBuf(ByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
          super(maxCapacity);
          ObjectUtil.checkNotNull(alloc, "alloc");
          checkPositiveOrZero(initialCapacity, "initialCapacity");
          checkPositiveOrZero(maxCapacity, "maxCapacity");
          if (initialCapacity > maxCapacity) {
              throw new IllegalArgumentException(String.format(
                      "initialCapacity(%d) > maxCapacity(%d)", initialCapacity, maxCapacity));
          }
  		//设置分配器
          this.alloc = alloc;
       	// 然后看下这个方法就好
       	// 把jdk创建的堆外内存进行保存
          setByteBuffer(allocateDirect(initialCapacity), false);
      }
  	 /**
       * Allocate a new direct {@link ByteBuffer} with the given initialCapacity.
       */
      protected ByteBuffer allocateDirect(int initialCapacity) {
          return ByteBuffer.allocateDirect(initialCapacity);
      }
  	
  	// 这里可以再分析一下，如果是unsafe 和 非unsafe,其实上面内容都是一直的，就是
  	//在setByteBuffer这个方法中，有一些特殊操作，比如要保持内存地址这些。
  		
  	// 非unsafe
  	//io.netty.buffer.UnpooledDirectByteBuf#setByteBuffer
  	void setByteBuffer(ByteBuffer buffer, boolean tryFree) {
          if (tryFree) {
              ByteBuffer oldBuffer = this.buffer;
              if (oldBuffer != null) {
                  if (doNotFree) {
                      doNotFree = false;
                  } else {
                      freeDirect(oldBuffer);
                  }
              }
          }
  
          this.buffer = buffer;	//这里直接保存的
          tmpNioBuf = null;
          capacity = buffer.remaining();
      }
  	
  	// unsafe
  	//io.netty.buffer.UnpooledUnsafeDirectByteBuf#setByteBuffe    
      @Override
      final void setByteBuffer(ByteBuffer buffer, boolean tryFree) {
          super.setByteBuffer(buffer, tryFree);
          //计算内存地址 里面是通过unsafe 获取这个buffer的内存地址，并保存，
          // 在get的时候，可以直接拿到这个 内存地址+偏移量 就可以直接访问，速度更快。
          memoryAddress = PlatformDependent.directBufferAddress(buffer);
      }
  ```



#### PooledByteBufAllocator

上面分析了unpooled的内存分配器，其实相对来说还是比较简单的，这里就要开始更为复杂的Pooled。

同样，先看下那两个抽象的API实现。`io.netty.buffer.PooledByteBufAllocator`

再分析之前，有必要先来看看netty自己实现的一个线程局部缓存。因为后面的两个方法中，是没有加锁的，并且这里会有多线程调用问题，所以就要解决并发问题。毕竟是操作同一个池子，如果每个调用线程，有一个自己的cache，那么这个问题就解决了。下面先说下，分配的流程。



**1：线程局部缓存，PoolThreadLocalCache**

```java
final class PoolThreadLocalCache extends FastThreadLocal<PoolThreadCache> {
    private final boolean useCacheForAllThreads;
    PoolThreadLocalCache(boolean useCacheForAllThreads) {
        this.useCacheForAllThreads = useCacheForAllThreads;
    }
	
    //其实主要是这个方法了。 
    @Override
    protected synchronized PoolThreadCache initialValue() {
        //这里的heapArenas 和 directArenas 是传进来的，那么创建的地方是哪里呢
        //当然是在这个构造中 io.netty.buffer.PooledByteBufAllocator#PooledByteBufAllocator
        // 这两个其实是一个数组，那么大小是多少呢，这里自己找下代码就好，其实就是cpu核数两倍，
        // 为什么是两倍呢，在创建NioEvetLoop的时候，默认也是这个值，也就是这样的话每个线程都有独享Area
        final PoolArena<byte[]> heapArena = leastUsedArena(heapArenas);
        final PoolArena<ByteBuffer> directArena = leastUsedArena(directArenas);

        final Thread current = Thread.currentThread();
        //useCacheForAllThreads 这个默认是true的
        if (useCacheForAllThreads || current instanceof FastThreadLocalThread) {
            // 创建这个cache
            final PoolThreadCache cache = new PoolThreadCache(
                    heapArena, directArena, tinyCacheSize, smallCacheSize, normalCacheSize,
                    DEFAULT_MAX_CACHED_BUFFER_CAPACITY, DEFAULT_CACHE_TRIM_INTERVAL);

            if (DEFAULT_CACHE_TRIM_INTERVAL_MILLIS > 0) {
                final EventExecutor executor = ThreadExecutorMap.currentExecutor();
                if (executor != null) {
                    executor.scheduleAtFixedRate(trimTask, DEFAULT_CACHE_TRIM_INTERVAL_MILLIS,
                            DEFAULT_CACHE_TRIM_INTERVAL_MILLIS, TimeUnit.MILLISECONDS);
                }
            }
            return cache;
        }
        // No caching so just use 0 as sizes.
        return new PoolThreadCache(heapArena, directArena, 0, 0, 0, 0, 0);
    }
```

从这里应该也可以看出，netty是如何减少多线程内存分配之间的竞争

**2：在线程局部缓存的Area上进行内存分配**

​	里面的实现细节，我们后面再分析。

* Heap分配

  `io.netty.buffer.PooledByteBufAllocator#newHeapBuffer`

  ```java
   @Override
      protected ByteBuf newHeapBuffer(int initialCapacity, int maxCapacity) {
          // 获取局部缓存
          PoolThreadCache cache = threadCache.get();
          // 获取缓存中的Area
          PoolArena<byte[]> heapArena = cache.heapArena;
  
          final ByteBuf buf;
          if (heapArena != null) {
              //分配
              buf = heapArena.allocate(cache, initialCapacity, maxCapacity);
          } else {
              buf = PlatformDependent.hasUnsafe() ?
                      new UnpooledUnsafeHeapByteBuf(this, initialCapacity, maxCapacity) :
                      new UnpooledHeapByteBuf(this, initialCapacity, maxCapacity);
          }
  
          return toLeakAwareBuffer(buf);
      }
  
  ```

  

* Direct的分配

  `io.netty.buffer.PooledByteBufAllocator#newDirectBuffer`

```java
	 @Override
    protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
        //首先是拿到这个threadCache对象，相比之前应该有了解过ThreadLocal，也就是线程局部缓存
        // 其实netty 有自己实现一个FastThreadLocal ,这个是在构造中初始化的
        PoolThreadCache cache = threadCache.get();
        //再拿到directArena
        PoolArena<ByteBuffer> directArena = cache.directArena;

        final ByteBuf buf;
        if (directArena != null) {
            // 这里就是分配了（这里后面再分析吧）
            buf = directArena.allocate(cache, initialCapacity, maxCapacity);
        } else {
            buf = PlatformDependent.hasUnsafe() ?
                    UnsafeByteBufUtil.newUnsafeDirectByteBuf(this, initialCapacity, maxCapacity) :
                    new UnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
        }

        return toLeakAwareBuffer(buf);
    }
```



### 内存分配

上面已经分析内存分配器，其实主要就上面两种，upooled来说是相对比较简单和容易理解的，对于pooled Netty的实现就比较复杂，因为他做了很多优化，比如:避免内存创建的消耗等，用到一些算法和策略。这里先了解一些基础变量和类。

```java
 	private static final int DEFAULT_TINY_CACHE_SIZE;//tinyCacheSize
    private static final int DEFAULT_SMALL_CACHE_SIZE;//smallCacheSize
    private static final int DEFAULT_NORMAL_CACHE_SIZE;//normalCacheSize
	
	//对应的默认值
DEFAULT_TINY_CACHE_SIZE = SystemPropertyUtil.getInt("io.netty.allocator.tinyCacheSize", 512);
        DEFAULT_SMALL_CACHE_SIZE = SystemPropertyUtil.getInt("io.netty.allocator.smallCacheSize", 256);
        DEFAULT_NORMAL_CACHE_SIZE = SystemPropertyUtil.getInt("io.netty.allocator.normalCacheSize", 64);
	
	// pageSize 
	private static final int DEFAULT_PAGE_SIZE;
	 int defaultPageSize = SystemPropertyUtil.getInt("io.netty.allocator.pageSize", 8192);// 默认大小是8K
        Throwable pageSizeFallbackCause = null;
        try {
            validateAndCalculatePageShifts(defaultPageSize);
        } catch (Throwable t) {
            pageSizeFallbackCause = t;
            defaultPageSize = 8192;
        }
        DEFAULT_PAGE_SIZE = defaultPageSize;


 private static final int DEFAULT_MAX_ORDER; // 8192 << 11 = 16 MiB per chunk
// chuckSize 默认16M ,就是上面公式计算出来的。
final int defaultChunkSize = DEFAULT_PAGE_SIZE << DEFAULT_MAX_ORDER;
//当然这个地方也是去计算chuckSize的
io.netty.buffer.PooledByteBufAllocator#validateAndCalculateChunkSize
```

大概的流程

>1. new一个ByteBuf，这里是分析pooled
>2. 从缓存中查找，没有可用的缓存进行下一步
>3. 从内存池中查找可用的内存，查找的方式如上所述（tiny、small、normal）
>4. 如果找不到则重新申请内存，并将申请到的内存放入内存池
>5. 使用申请到的内存初始化ByteBuf

首先，要了解的是，在poole中，对于heap和direct都有一个自己的对象池。可以看下代码

```java
 private static final ObjectPool<PooledUnsafeHeapByteBuf> RECYCLER = ObjectPool.newPool(
            new ObjectCreator<PooledUnsafeHeapByteBuf>() {
        @Override
        public PooledUnsafeHeapByteBuf newObject(Handle<PooledUnsafeHeapByteBuf> handle) {
            return new PooledUnsafeHeapByteBuf(handle, 0);
        }
    });

 private static final ObjectPool<PooledUnsafeDirectByteBuf> RECYCLER = ObjectPool.newPool(
            new ObjectCreator<PooledUnsafeDirectByteBuf>() {
        @Override
        public PooledUnsafeDirectByteBuf newObject(Handle<PooledUnsafeDirectByteBuf> handle) {
            return new PooledUnsafeDirectByteBuf(handle, 0);
        }
    });

// 看源码应该知道 这个ObjectPool 也就是 RECYCLER是 RecyclerObjectPool
	 public static <T> ObjectPool<T> newPool(final ObjectCreator<T> creator) {
        return new RecyclerObjectPool<T>(ObjectUtil.checkNotNull(creator, "creator"));
    }
//当在获取对象池的时候，可以直接RECYCLER.get()，如果有这个对象的对象池，就会直接返回，否则就新建一个。
// 这里面的逻辑 具体还是要自己跟一下源码 这里只看下get方法。
 @SuppressWarnings("unchecked")
    public final T get() {
        if (maxCapacityPerThread == 0) {
            return newObject((Handle<T>) NOOP_HANDLE);
        }
        // 这其实也是线程的一个局部缓存
        Stack<T> stack = threadLocal.get();
       
        DefaultHandle<T> handle = stack.pop();
        if (handle == null) {
             //这个handler 就是用来负责回收的
            handle = stack.newHandle();
            // 把这个handler传进去 这个newObject 就是自己实现的方法，会返回具体的buf
            handle.value = newObject(handle);
        }
        return (T) handle.value;
    }
```



 

#### directArena分配

这个主要是用来先看下这个Arena的分配流程是如何的，上面已经上过了，pageSize,和chuckSize的大小。

1：在ThreadCache上拿到directArena

 `io.netty.buffer.PooledByteBufAllocator#newDirectBuffer`

```java
tected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
        PoolThreadCache cache = threadCache.get();
        PoolArena<ByteBuffer> directArena = cache.directArena;

```

2: 获取bytebuf

`io.netty.buffer.PoolArena#allocate(io.netty.buffer.PoolThreadCache, int, int)`

```java
 PooledByteBuf<T> allocate(PoolThreadCache cache, int reqCapacity, int maxCapacity) {
     	//这里我们看这个方法
        PooledByteBuf<T> buf = newByteBuf(maxCapacity);
        allocate(cache, buf, reqCapacity);
        return buf;
    }

 	@Override
        protected PooledByteBuf<ByteBuffer> newByteBuf(int maxCapacity) {
            if (HAS_UNSAFE) {
                // 接着看这个
                return PooledUnsafeDirectByteBuf.newInstance(maxCapacity);
            } else {
                return PooledDirectByteBuf.newInstance(maxCapacity);
            }
        }

 //io.netty.buffer.PooledUnsafeDirectByteBuf#newInstance
 static PooledUnsafeDirectByteBuf newInstance(int maxCapacity) {
     	// RECYCLER 这个就是带有回收的对象次，也就是说，如果这个对像池中有就直接返回。
     	// 否则就创建一个
        PooledUnsafeDirectByteBuf buf = RECYCLER.get();
     	// 设置引用 和索引
        buf.reuse(maxCapacity);
        return buf;
    }
```

3: 分配

`io.netty.buffer.PoolArena#allocate(io.netty.buffer.PoolThreadCache, io.netty.buffer.PooledByteBuf<T>, int)`，这里的代码有点长，其实也是核心的地方，但后面会具体分析。

```java
private void allocate(PoolThreadCache cache, PooledByteBuf<T> buf, final int reqCapacity) {
        final int normCapacity = normalizeCapacity(reqCapacity);
        if (isTinyOrSmall(normCapacity)) { // capacity < pageSize
            int tableIdx;
            PoolSubpage<T>[] table;
            boolean tiny = isTiny(normCapacity);
            if (tiny) { // < 512
                if (cache.allocateTiny(this, buf, reqCapacity, normCapacity)) {
                    // was able to allocate out of the cache so move on
                    return;
                }
                tableIdx = tinyIdx(normCapacity);
                table = tinySubpagePools;
            } else {
                if (cache.allocateSmall(this, buf, reqCapacity, normCapacity)) {
                    // was able to allocate out of the cache so move on
                    return;
                }
                tableIdx = smallIdx(normCapacity);
                table = smallSubpagePools;
            }

            final PoolSubpage<T> head = table[tableIdx];

            synchronized (head) {
                final PoolSubpage<T> s = head.next;
                if (s != head) {
                    assert s.doNotDestroy && s.elemSize == normCapacity;
                    long handle = s.allocate();
                    assert handle >= 0;
                    s.chunk.initBufWithSubpage(buf, null, handle, reqCapacity, cache);
                    incTinySmallAllocation(tiny);
                    return;
                }
            }
            synchronized (this) {
                allocateNormal(buf, reqCapacity, normCapacity, cache);
            }

            incTinySmallAllocation(tiny);
            return;
        }
        if (normCapacity <= chunkSize) {
            if (cache.allocateNormal(this, buf, reqCapacity, normCapacity)) {
                // was able to allocate out of the cache so move on
                return;
            }
            synchronized (this) {
                allocateNormal(buf, reqCapacity, normCapacity, cache);
                ++allocationsNormal;
            }
        } else {
            // Huge allocations are never served via the cache so just call allocateHuge
            allocateHuge(buf, reqCapacity);
        }
    }
```

上面看到`cache.allocateXXX`就是表明是从缓存上进行分配，这里先知道就好，分配成功就返回，否则，就会内存堆分配。如果需要分配的大小大于16M，就会allocateHuge()来分配，也不是在缓存上。上面就是一个内存分配的流程，后面要分析的肯定是和缓存有关了。

#### 内存规格

先看图说明

![bytebuf3](https://github.com/tryingpfq/tryingpfq.github.io/blob/master/picture/bytebuf3.jpg?raw=true)

内存申请是以chuck为单位向操作系统申请，也就是16M，后续的所有的内存分配都是在chuck中分配。比如需要分配一个1M内存，首先需要向操作系统申请一个chuck,然后在这个chuck中，取一段1M大小的内存进行分配，放到bytebuf中。但这样又个问题，假如有些内存，比较小，16M就会比较大，所以用chuck进行切分成Page,也就是2^11=2028个page，比如这个是否分配16K大小的内存，就只需要两个连续page就行。从0---8K也有一个单位就是Subpage,比如以8k为单位进行分配的，在申请比较小的，比如20字节，以page分配的话，就会很浪费。假如分配2K内存，就只需要把Page切分4分，然后在这个page中取取一段就好，同理如果是512B，就分成16个片段就好。这里就是netty中内存规格的一个介绍。



#### 基本数据结构



#### 缓存命中流程



#### 不同大小的内存的分配策略



### 内存回收过程



