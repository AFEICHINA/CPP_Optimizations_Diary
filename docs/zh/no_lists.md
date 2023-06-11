# 如果您正在使用std::list<>，那么您就错了

![](img/linked_list.png)

我不会浪费时间重复一些已经由很多人做过的基准测试。

- [std::vector vs std::list benchmark](https://baptiste-wicht.com/posts/2012/11/cpp-benchmark-vector-vs-list.html)

- [链表有害吗？Bjarne Stroustrup](https://isocpp.org/blog/2014/06/stroustrup-lists)

- [Bjarne Stroustrup主题演讲视频](https://www.youtube.com/watch?v=YQs6IC-vgmo)

你认为你的情况很特殊，是一个独特的雪花。 **它不是**。

您有另一个STL数据结构比`std::list`更好：
[std::deque <>](https://es.cppreference.com/w/cpp/container/deque)几乎99％的时间。

在某些情况下，即使简陋的`std::vector`也比链表更好。

如果您喜欢非常奇特的替代方案，请查看[plf :: colony](https://plflib.org/colony.htm)。

但是说真的，只要使用`vector`或`deque`。

## 实际示例：改进Intel RealSense驱动程序

这是我不久前向[RealSense](https://github.com/IntelRealSense)存储库发送的Pull Request的实际示例。

![](img/realsense.png)

他们出于我无法理解的原因使用了`std::list<>`这种可恶的东西。

开玩笑，英特尔开发人员，我们爱你们！

在这里，您可以找到Pull Request的链接：
- [Considerable CPU saving in BaseRealSenseNode::publishPointCloud()](https://github.com/IntelRealSense/realsense-ros/pull/1097)

简而言之，整个PR仅包含两个微小的更改：

```C++
// 我们更改了每个相机帧创建的此列表
std::list<unsigned> valid_indices;

// 使用这个向量：一个类成员，在重新使用之前被清除
//（但是分配的内存还在那里）
std::vector<unsigned> _valid_indices;
```

另外，我们有一个相当大的对象名为`sensor_msgs :: PointCloud2 msg_pointcloud`
转换为重复使用的类成员。

报告的速度提高为20％-30％，如果您考虑到这一点，这是巨大的。