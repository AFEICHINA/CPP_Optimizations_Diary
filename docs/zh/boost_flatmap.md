# 通过异类关联容器进行优化



![](img/twitter_unordered.png)

来吧，迈克尔，你小看我了；）

## 问题

上周，我决定调查使用CPU时间过多的模块或者至少比我预期的多的部分。

我不会用细节麻烦你，但它是一个低级硬件接口，用于控制机器人的电机。

我对软件系统的这一部分已经知道足够多的东西，以相信它不应该让整个CPU内核忙碌。 

不用说，我使用了我的好朋友**Hotspost**来查看CPU时间花费在哪里：

![](img/motor_profile1.png)

看起来很乱，不是吗？请理解。

如果您是[火焰图](http://www.brendangregg.com/flamegraphs.html)的新手，请不要惊慌。简单来说，调用函数位于金字塔底部，它们的被调用函数位于其顶部。

引起我的注意的是，在方法**std::unordered_map<>::operator[]**中有30%的CPU浪费。

![](img/motor_profile2.png)

右侧有一个大块，然后左侧的代码中有许多多次调用。

有问题的容器大致如下：

```C++
// 简化的代码
std::unordered_map<Address, Entry*> m_dictionary;

//其中
struct Address{
    int16_t index;
    unt8_t subindex;
};
```

## 解决方案

我检查了代码并发现了可怕的事情。例如：

```C++
// 简化的代码
bool hasEntry(const Address& address) 
{
    return m_dictionary.count(address) != 0;
}

Value getEntry(const Address& address) 
{
    if( !hasEntry(address) {
        throw ...
    }
    Entry* = m_dictionary[address];
    // 其他代码
}
```

![](img/two_lookups.jpg)

正确的做法是：

```C++
// 简化的代码。仅一次查找
Value getEntry(const Address& address) 
{
    auto it = m_dictionary.find(address);
    if( it ==  m_dictionary.end() ) {
        throw ...
    }
    Entry* = it->second;
    // 其他代码
}
```

这个改变单独减少了`std::unordered_map`在火焰图中观察到的开销的一半。

## 拥抱缓存

非常重要的是要注意，一旦创建，字典在运行时永远不会更改。

这使我们可以像图片中所述那样，使用缓存来优化右侧的大块。

实际上，映射查找在回调内执行，该回调具有访问相关地址的功能。但是，如果**[Address，Entry *]**对从不更改，为什么不直接存储`Entry*`呢？

如预期的那样，在其中一个以前的文章中，[关于2D转换的文章](2d_transforms.md)，这将完全消除大块上的开销，仅使用15%的CPU。

## 使用`boost::container_flat_map`获得巨大收益

在其余代码中使用缓存将是一个痛苦。这是“千刀万剐”的典型例子。

此外，有些东西在我脑海中告诉我，有些事情不对劲。

为什么`std::hash<Address>`需要那么长时间？这可能是**少见的情况之一**，我不应该查看大O()符号吗？

`std::unordered_map`查询是O(1)，这是个好事情，不是吗？

我们是否应该尝试某些横向思考，使用另一个具有O(logn)查询复杂度但没有哈希函数成本的容器呢？

请轻敲鼓点欢迎 [boost::container_flat_map](https://www.boost.org/doc/libs/1_74_0/doc/html/container/non_standard_containers.html#container.non_standard_containers.flat_xxx)。

我不会重复文档解释的实现方式，它主要是一种普通的有序向量，类似于我讨论的[关于std::map的文章](dont_need_map.md)末尾。

结果让我感到惊讶：我不仅“减少了”开销，而且完全消除了它。`flat_map<>::operator[]`的成本几乎无法衡量。

基本上，只需切换到`flat_map`即可通过更改一行代码解决整个问题！




