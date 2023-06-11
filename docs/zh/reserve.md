# Vectors are awesome...

`std::vector<>`s have a huge advantage when compared to other data structures:
their elements are packed in memory one next to the other.

We might have a long discussion about how this may affect performance, based on how memory
works in modern processors.

If you want to know more about it, just Google "C++ cache aware programming". For instance:

- [CPU Caches and why you Care](https://www.aristeia.com/TalkNotes/codedive-CPUCachesHandouts.pdf)
- [Writing cache friendly C++ (video)](https://www.youtube.com/watch?v=Nz9SiF0QVKY) 

Iterating through all the elements of a vector is very fast and they work really really well when we have to
append or remove an element from the back of the structure.

# ... when you use `reserve`

We need to understand how vectors work under the hood.
When you push an element into an empty or full vector, we need to:

- allocate a new block of memory that is larger.
- move all the elements we have already stored in the previous block into the new one. 

Both these operations are expensive and we want to avoid them as much as possible, if you can, 
sometimes you just accept things the way they are.

The size of the new block is **2X the capacity**. Therefore, if you have 
a vector where both `size()` and  `capacity()` are 100 elements and you `push_back()` element 101th,
the block of memory (and the capacity) will jump to 200. 

To prevent these allocations, that may happen multiple times, we can **reserve** the capacity that 
we know (or believe) the vector needs.

Let's have a look to a micro-benchmark.

```C++
static void NoReserve(benchmark::State& state) 
{
  for (auto _ : state) {
    // create a vector and add 100 elements
    std::vector<size_t> v;
    for(size_t i=0; i<100; i++){  v.push_back(i);  }
  }
}

static void WithReserve(benchmark::State& state) 
{
  for (auto _ : state) {
    // create a vector and add 100 elements, but reserve first
    std::vector<size_t> v;
    v.reserve(100);
    for(size_t i=0; i<100; i++){  v.push_back(i);  }
  }
}


static void ObsessiveRecycling(benchmark::State& state) {
  // create the vector only once
  std::vector<size_t> v;
  for (auto _ : state) {
    // clear it. Capacity is still 100+ from previous run
    v.clear();
    for(size_t i=0; i<100; i++){  v.push_back(i);  }
  }
}
```

![](img/vector_reserve.png)

Look at the difference! And these are only 100 elements.

The number of elements influence the final performance gain a lot, but one thing is sure: it **will** be faster.

Note also as the `ObsessiveRecycling` brings a performance gain that is probably visible for small vectors, but negligible with bigger ones.

Don't take me wrong, though: `ObsessiveRecycling` will always be faster, even if according to the size of the object you are storing
you may or may not notice that difference.


## Recognizing a vector at first sight

This is the amount of memory an applications of mine was using over time (image obtained with **Heaptrack**):

![](img/growing_vector.png)

Look at that! Something is doubling the amount of memory it is using by a factor of two every few seconds...

I wonder what it could be? A vector, of course, because other data structures would have a more "linear" growth.

That, by the way, **is a bug in the code that was found thanks to memory profiling**: that vector was not supposed to grow at all.




# 向量是很棒的……

与其他数据结构相比，`std::vector<>` 有一个巨大的优势：它们的元素在内存中依次紧密排列。

我们可能会对这如何影响性能进行长时间讨论，这要基于现代处理器中内存的工作方式。

如果您想更多了解，只需搜索“C++ cache aware programming”。例如：

- [CPU Caches and why you Care](https://www.aristeia.com/TalkNotes/codedive-CPUCachesHandouts.pdf)
- [Writing cache friendly C++(视频)](https://www.youtube.com/watch?v=Nz9SiF0QVKY)

遍历向量的所有元素非常快，当我们需要从结构的后面添加或删除元素时，它们非常非常有效。

# ……当你使用 `reserve` 时

我们需要了解向量在幕后工作的方式。
当您将一个元素推入一个空或满的向量时，我们需要：

- 分配一个新的更大的内存块。
- 将我们已经存储在上一个块中的所有元素移动到新块中。

这两个操作都很昂贵，我们要尽可能地避免它们。如果可以的话，有时您只需接受事实。

新块的大小为 **容量的2倍**。因此，如果您有一个向量，其中 `size()` 和 `capacity()` 都是100个元素，
并且您将第101个元素 `push_back()` 到该向量中，那么内存块（及容量）将跳到200。

为了防止这些可能多次发生的分配，我们可以 **reserve** 我们知道（或相信）向量需要的容量。

让我们看一个微型基准测试。

```C++
static void NoReserve(benchmark::State& state) 
{
  for (auto _ : state) {
    // 创建一个向量并添加100个元素
    std::vector<size_t> v;
    for(size_t i=0; i<100; i++){  v.push_back(i);  }
  }
}

static void WithReserve(benchmark::State& state) 
{
  for (auto _ : state) {
    // 创建一个向量并添加100个元素，但是预留空间
    std::vector<size_t> v;
    v.reserve(100);
    for(size_t i=0; i<100; i++){  v.push_back(i);  }
  }
}


static void ObsessiveRecycling(benchmark::State& state) {
  // 只创建一次向量
  std::vector<size_t> v;
  for (auto _ : state) {
    // 清除它。容量仍然比前一次运行多100+
    v.clear();
    for(size_t i=0; i<100; i++){  v.push_back(i);  }
  }
}
```

![](img/vector_reserve.png)

看看差异！这些只有100个元素。

元素的数量对最终的性能收益影响很大，但有一件事是肯定的：它 **会** 更快。

还要注意，`ObsessiveRecycling` 带来的性能增益在小向量上可能是可见的，但对于更大的向量则可以忽略不计。

不过请别误解：即使根据您存储的对象的大小，您可能会或可能不会注意到这种差异，`ObsessiveRecycling` 总是会更快的。

## 一眼识别一个向量

这是我的一个应用程序随时间使用的内存量（图像使用 **Heaptrack** 获取）：

![](img/growing_vector.png)

看那个！某些东西每隔几秒钟就会将它使用的内存量增加两倍...

我在想，那到底是什么呢？当然是向量，因为其他数据结构的增长速度通常更为“线性”。

顺便说一句，这其实是代码中的一个错误，是通过内存分析找到的：该向量根本不应该增长。



