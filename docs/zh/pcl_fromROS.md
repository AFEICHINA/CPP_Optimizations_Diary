# 案例研究：将ROS消息转换为PCL

[点云库（PCL）](https://pointclouds.org/) 似乎是优化的一个丰富源泉。

即使这是一种毫无根据的批评，让我们记住 **Bjarne Stroustrup** 的话：

> “只有两种语言：人们抱怨的和没人使用的。”

因此，请记住，PCL 在使每个人轻松处理点云方面发挥了 **巨大** 的作用。开发人员和维护者应得到我所有的尊重！

说过了这些，让我们进入我的下一个牢骚。

![](img/davide_yells_at_PCL.jpg)


## 使用 pcl::fromROSMsg()

如果您在 ROS 中使用 PCL，则以下代码是您的日常工作：

```c++
void cloudCallback(const sensor_msgs::PointCloud2ConstPtr& msg)
{
  pcl::PointCloud<pcl::PointXYZ> cloud;
  pcl::fromROSMsg(*msg, cloud);

  //...
}
```

现在，我无法计算出有多少人抱怨这个转换单独使用了很多 CPU！

我查看了它的实现和 Hotspot 的结果（性能分析），立即发现了一个问题：

```c++
template<typename T>
void fromROSMsg(const sensor_msgs::msg::PointCloud2 &cloud,
                pcl::PointCloud<T> &pcl_cloud)
{
  pcl::PCLPointCloud2 pcl_pc2;
  pcl_conversions::toPCL(cloud, pcl_pc2);
  pcl::fromPCLPointCloud2(pcl_pc2, pcl_cloud);
}
```

我们转换/复制了两次数据：

- 首先，我们从 `sensor_msgs::msg::PointCloud2` 转换为 
`pcl::PCLPointCloud2`
- 然后，从 `pcl::PCLPointCloud2` 转换为 `pcl::PointCloud<T>`。

深入研究 `pcl_conversions::toPCL` 的实现，我发现了这个：

```c++
void toPCL(const sensor_msgs::msg::PointCloud2 &pc2,
           pcl::PCLPointCloud2 &pcl_pc2)
{
  copyPointCloud2MetaData(pc2, pcl_pc2);
  pcl_pc2.data = pc2.data;
}
```

将原始数据从一种类型复制到另一种类型是可以轻松避免的开销。

这种重构并不特别有趣，因为我基本上 "复制粘贴" 了 `pcl::fromPCLPointCloud2` 的代码以使用不同的输入类型。

快进到解决方案，让我们看看结果：

![](img/pcl_fromros.png)

## 这个故事的经验教训是什么？

**测量、测量、再测量！** 不要认为 "聪明人" 实现了最佳解决方案，您无法积极地采取任何措施。

在这种情况下，清晰的代码和重用现有函数优先于性能。

但是这个决定的影响绝对不能被忽视。




