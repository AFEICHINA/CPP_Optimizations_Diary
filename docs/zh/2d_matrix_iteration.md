# 迭代2D矩阵

在我的领域（机器人技术）中，2D矩阵非常常见。我们使用它们来表示图像、网格地图/成本地图等。

最近，我意识到我们处理成本地图的算法相当慢，并且我使用Hotspot进行了分析。

我意识到瓶颈之一是将两个成本地图合并以创建第三个地图的函数。该函数看起来大致如下：

```C++
//这是过于简化的代码，请不要与我争论
for( size_t y = y_min; y < y_max; y++ ) 
{
    for( size_t x = x_min; x < x_max; x++ ) 
    {
        matrix_out( x,y ) = std::max( mat_a( x,y ), mat_b( x,y ) ); 
    }
}
```

很简单明了，是优雅的编写方式。但由于我的测量结果表明它使用了太多CPU，因此我决定进入优化洞穴内部跟踪白兔。

## 如何在C ++中编写良好的2D矩阵？

您曾经编写过此类代码吗？

```C++
//我的计算机科学教授就是这样做的
float** matrix = (float**) malloc( rows_count * sizeof(float*) );
for(int r=0; r<rows_count; r++) 
{
    matrix[r] = (float*) malloc( columns_count * sizeof(float) );
}
//像这样访问矩阵元素：
matrix[row][col] = 42;
```

然后你可以休息了。您可以将此代码带到[Mount Doom in Mordor](https://en.wikipedia.org/wiki/Mount_Doom)，并将其扔进volcano里。

![](../img/mordor.jpg)

首先，请使用[Eigen](http://eigen.tuxfamily.org)。它是一个优秀的库，具有出色的性能和美丽的API。

其次，仅出于教学目的，以下是您应该实现有效矩阵的方式（但不要这样做，请使用Eigen，认真考虑）。在此处显示它是相关的，因为这是包括Eigen在内的所有人都遵循的方法。

```C++
template <typename T> class Matrix2D
{
public:
    Matrix2D(size_t rows, size_t columns):  _num_rows(rows)
    {
        _data.resize( rows * columns );
    }
    
    size_t rows() const
    { 
    	return _num_rows; 
    }
    
    T& operator()(size_t row, size_t col)  
    {
        size_t index = col*_num_rows + row; 
        return _data[index];
    }
    
    T& operator[](size_t index)  
    {
        return _data[index];
    }
    
    // all the other methods omitted
private:
    std::vector<T> _data;
    size_t _num_rows;
};

//像这样访问矩阵元素：
matrix(row, col) = 42;
```

这是最适合构建矩阵的方式，具有单个内存分配和以[列为主](https://www.geeksforgeeks.org/row-wise-vs-column-wise-traversal-matrix/)方式存储的数据。

要将行/列对转换为向量中的索引，我们需要进行一次乘法和加法。

## 回到我的问题

您还记得我们在开头编写的代码吗？

我们的迭代次数等于`(x_max-x_min)*(y_max-y_min)`。通常，这是很多像素/单元。

在每次迭代中，我们使用公式三次计算索引：

     size_t index = col*_num_rows + row;

天哪，那是很多乘法！

结果表明，将代码重写如下就值得了：

```C++
//手动计算索引
for(size_t y = y_min; y < y_max; y++) 
{
    size_t offset_out =  y * matrix_out.rows();
    size_t offset_a   =  y * mat_a.rows();
    size_t offset_b   =  y * mat_b.rows();
    for(size_t x = x_min; x < x_max; x++) 
    {
        size_t index_out =  offset_out + x;
        size_t index_a   =  offset_a + x;
        size_t index_b   =  offset_b + x;
        matrix_out( index_out ) = std::max( mat_a( index_a ), mat_b( index_b ) ); 
    }
}
```

所以我知道你在想什么，**我的眼睛也很疼**。这很难看。但是性能提升太多了，不能忽略。

考虑到乘法的数量被因子`(x_max-x_min)*3`大大减少，这并不令人惊讶。



