# 不要计算它两次

我将要展示的例子会让你们中的一些人做出这样的反应：
![really](img/really.jpg)

我这样说是因为这将是绝对显而易见的...在回顾中。

另一方面，我已经看到在多个开源项目中使用了**相同的代码**。

拥有数百颗Github star的项目错过了这个（显然很明显的）优化机会。

一个值得注意的例子是：[加快laserOdometry和scanRegister模块计算的速度 （20%）](https://github.com/laboshinl/loam_velodyne/pull/20)

## 2D transforms

让我们来看看这部分代码:

```c++
double x1 = x*cos(ang) - y*sin(ang) + tx;
double y1 = x*sin(ang) + y*cos(ang) + ty;
```

具有训练有素的眼睛（和一些几何学背景）的人会立即识别到这是[2D点的仿射变换]，通常用于计算机图形学和机器人技术。

难道你没有看到我们可以做得更好的地方吗？答案是肯定的：
```c++
const double Cos = cos(angle);
const double Sin = sin(angle);
double x1 = x*Cos - y*Sin + tx;
double y1 = x*Sin + y*Cos + ty;
```

三角函数的计算成本相对较高，没有任何理由重复计算相同的值。与“sin（）”和“cos（）”相比，乘法和加法的成本非常低，因此后者的代码将比前者快2倍。

总的来说，如果您需要测试的潜在角度数量不是非常高，请考虑使用查找表，在其中可以存储预先计算的值。例如，激光扫描数据需要从极坐标转换为笛卡尔坐标系，这就是其中一个情况。

这就是激光扫描数据的情况，例如需要将其从极坐标转换为笛卡尔坐标。

![laser_scan_matcher.png](img/laser_scan_matcher.png)

一个自然的实现将会为每个点调用三角函数（每秒数千次）。

```c++
// Conceptual operation (inefficient)
// Data is usually stored in a vector of distances
std::vector<double> scan_distance; // the input
std::vector<Pos2D> cartesian_points; // the output

cartesian_points.reserve( scan_distance.size() );

for(int i=0; i<scan_distance.size(); i++)
{
    const double dist = scan_distance[i];
    const double angle = angle_minimum + (angle_increment*i);
    double x = dist*cos(angle);
    double y = dist*sin(angle);
    cartesian_points.push_back( Pos2D(x,y) );
}
```

但我们应该考虑：

- **angle_minimum** 和 **angle_increment** 是永远不会改变的常数。
- **scan_distance** 的大小也是恒定的（当然不包括其内容）。

这是一个完美的例子，使用查找表就很有意义，并且将显著提高性能。
 
```C++
 
//------ To do only ONCE -------
std::vector<double> LUT_cos;
std::vector<double> LUT_sin;

for(int i=0; i<scan_distance.size(); i++)
{
    const double angle = angle_minimum + (angle_increment*i);
    LUT_cos.push_back( cos(angle) );
    LUT_sin.push_back( sin(angle) );
}

// ----- The efficient scan conversion ------
std::vector<double> scan_distance;
std::vector<Pos2D> cartesian_points;

cartesian_points.reserve( scan_distance.size() );

for(int i=0; i<scan_distance.size(); i++)
{
    const double dist = scan_distance[i];
    double x = dist*LUT_cos[i];
    double y = dist*LUT_sin[i];
    cartesian_points.push_back( Pos2D(x,y) );
}
```

# 总结要点

这是一个简单的例子；从中您应该学到的是，每当计算操作成本较高（如SQL查询或无状态数学运算）时，都应考虑使用缓存值和/或构建查找表。

但是，一如既往地，请先进行**测量**以确保优化实际上是相关的；)

