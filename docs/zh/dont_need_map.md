# 您实际上需要使用std::map吗？

`std::map`是C++中最著名的数据结构之一，对于我们大多数人来说是默认的关联容器，但随着年份的增加，它的流行度已经逐渐下降。

当您有键/值对并且想要通过密钥查找值时，使用关联容器是非常合适的。

但是，由于[红黑树](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree)的节点创建方式，`std::map`与`std::list`（也就是一个不受欢迎的内存分配操作，并且具有非常不适合缓存的内存布局）之间没有多少区别。

在选择此数据结构之前，请自问以下问题：

- 我需要所有对按其键**排序**吗？
- 我需要经常遍历容器中的所有项吗？

如果第一个答案是否定的，则可能希望默认切换到`std::unordered_map`。

在我的所有基准测试中，这总是胜出的。也许我很幸运，也许有些情况下`std::map`表现更好，但我尚未找到这些情况。

如果你回答"yes"，意味着你会面临些许挑战。

请坐在我的膝盖上，孩子，加入我的优化历险。

## 优化Velodyne驱动程序

这是我特别自豪的Pull Request：

[Avoid unnecessary computation in RawData::unpack](https://github.com/ros-drivers/velodyne/pull/194)

要理解如何通过小的更改产生巨大差异，请考虑此驱动程序正在执行的操作。

![](../img/velodyne.png)

Velodyne是一种传感器，可以每秒测量数十万个点（障碍物距离），它是大多数自动驾驶汽车中最重要的传感器。

Velodyne驱动程序将极坐标中的测量值转换为三维笛卡尔坐标（即所谓的“PointCloud”）。

我使用**Hotspot**分析了Velodyne驱动程序并发现了一个与`std::map::operator[]`相关的高CPU使用率的问题。

因此，我探索了代码并找到了以下内容：

```C++
 std::map<int, LaserCorrection> laser_corrections;
```

`LaserCorrection`包含一些校准信息，需要进行调整测量。

映射的`int`键是范围为 [0, N-1] 的数字，其中N可能为16、32或64。也许有一天它会达到128！

此外，`laser_corrections`只创建一次（没有进一步插入），并在循环中反复使用，如下所示：

```C++
 // 代码简化
 for (int i = 0; i < BLOCKS_PER_PACKET; i++) {
    //一些代码
    for (int j = 0; j < NUM_SCANS; j++) 
    {   
        int laser_number = // 简化代码，省略了一些内容
        const LaserCorrection& corrections = laser_corrections[laser_number];
        //一些代码
    }
 }
```

事实上，在这个无辜的代码行背后：

      laser_corrections[laser_number];

有一个在红黑树中进行搜索的操作！

记住：索引**不是**随机数，它的值始终在0和N-1之间，其中N非常小。

因此，我提出了以下更改，您无法想象接下来发生了什么：

```C++
 std::vector<LaserCorrection> laser_corrections;
```

![](../img/quote.png)

总结一下，没有必要使用关联容器，因为向量本身的位置（向量中的索引）已经完美地工作。

我绝不会归咎于Velodyne驱动程序的开发人员，因为像这样的更改只有在回顾过程中才有意义：直到您对应用程序进行分析并进行一些实际测量，才能找到一个明显隐藏的不必要开销。

当你想到函数的其余部分执行了**大量**的数学运算时，你就可以理解实际瓶颈是微小的`std::map`有多么不合逻辑。

## 更进一步：键值对的向量

由于其非常方便的整数键，即0到N之间的小数字，因此此示例相当“极端”。

尽管如此，在我的代码中，我经常使用这样的结构，而不是“真正的”关联容器：

```C++
std::vector< std::pair<KeyType, ValueType> > my_map;
```

如果您需要**频繁遍历所有元素**，则这是最佳数据结构。

大多数情况下，您不能击败它！

> “但是Davide，我需要按顺序排列这些元素，这就是我使用`std::map`的原因！”

好吧，如果您需要排序...那就排序呗！

```C++
std::sort( my_map.begin(), my_map.end() ) ;
```
> “但是Davide，有时我需要在我的映射中搜索元素”

在那种情况下，您可以通过在**有序**向量中使用函数[std::lower_bound](http://www.cplusplus.com/reference/algorithm/lower_bound/)按其键查找元素。

`lower_bound` / `upper_bound`的复杂度为**O(log n)**，与`std::map`相同，但遍历所有元素要快得多。

## 总结

- 考虑访问数据的方式。
- 问自己是否频繁或不频繁插入/删除。
- 不要低估关联容器的成本。
- 默认使用`std::unordered_map`...当然也可以使用`std::vector`！
