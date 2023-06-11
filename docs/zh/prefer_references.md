# 值语义 vs 引用

我要在这里说的内容太平凡了，以至于任何有经验的开发人员可能已经知道了。

尽管如此，我仍然经常看到人们编写如下代码：

```C++
bool OpenFile(str::string filename);

void DrawPath(std::vector<Points> path);

Pose DetectFace(Image image);

Matrix3D Rotate(Matrix3D mat, AxisAngle axis_angle);

```

我编造了这些函数，但是**我确实看到**生产代码中有这样的代码。

这些函数有什么共同点？您正在通过值传递参数。

换句话说，每次调用其中一个函数时，您都会在您的范围内复制输入，并将 **副本** 传递给函数。

![](img/why_copy.jpg)

根据对象的大小或是否需要动态堆内存分配，副本可能是昂贵的，也可能不是昂贵的。

在这些示例中，当按值传递时，可能具有可以忽略的开销的对象是 `Matrix3D` 和 `AngleAxis`，
因为我们可以假设它们不需要堆分配。

但是，即使开销很小，如果我们可以避免浪费 CPU 循环，那么有什么理由吗？

这是更好的 API：

```C++
bool OpenFile(const str::string& filename); // string_view is even better

void DrawPath(const std::vector<Points>& path);

Pose DetectFace(const Image& image);

Matrix3D Rotate(const Matrix3D& mat, const AxisAngle& axis_angle);

```

在后者的版本中，我们使用所谓的 **"引用语义"**。

您可以使用 **C-style**（非拥有）指针而不是引用，并获得相同的性能优势，但在这里，
我们告诉编译器参数是：

- 常量。我们不会在被调用函数的范围内更改它们，也不会在被调用的函数内部更改它们。
- 作为引用，参数 *引用* 存在的对象。裸指针可能具有值 `nullptr`。
- 不是指针，因此我们确定我们没有转移对象的所有权。

成本可能会有很大不同，如下所示：
```C++
size_t GetSpaces_Value(std::string str)
{
    size_t spaces = 0;
    for(const char c: str){
        if( c == ' ') spaces++;
    }
    return spaces;
}

size_t GetSpaces_Ref(const std::string& str)
{
    size_t spaces = 0;
    for(const char c: str){
        if( c == ' ') spaces++;
    }
    return spaces;
}

const std::string LONG_STR("a long string that can't use Small String Optimization");

void PassStringByValue(benchmark::State& state) {
    for (auto _ : state) {
        size_t n = GetSpaces_Value(LONG_STR);
    }
}

void PassStringByRef(benchmark::State& state) {
    for (auto _ : state) {
        size_t n = GetSpaces_Ref(LONG_STR);
    }
}

//----------------------------------
size_t Sum_Value(std::vector<unsigned> vect)
{
    size_t sum = 0;
    for(unsigned val: vect) { sum += val; }
    return sum;
}

size_t Sum_Ref(const std::vector<unsigned>& vect)
{
    size_t sum = 0;
    for(unsigned val: vect) { sum += val; }
    return sum;
}

const std::vector<unsigned> vect_in = { 1, 2, 3, 4, 5 };

void PassVectorByValue(benchmark::State& state) {
    for (auto _ : state) {
        size_t n = Sum_Value(vect_in);
    }
}

void PassVectorByRef(benchmark::State& state) {
    for (auto _ : state) {
        size_t n = Sum_Ref(vect_in);
        benchmark::DoNotOptimize(n);
    }
}
```

![](img/const_reference.png)

显然，引用传递完胜。

## 规则的例外

> "这很酷，Davide，我会到处使用 `const&`。"

让我们先看另一个例子。

```C++
struct Vector3D{
    double x;
    double y;
    double z;
};

Vector3D MultiplyByTwo_Value(Vector3D p){
    return { p.x*2, p.y*2, p.z*2 };
}

Vector3D MultiplyByTwo_Ref(const Vector3D& p){
    return { p.x*2, p.y*2, p.z*2 };
}

void MultiplyVector_Value(benchmark::State& state) {
    Vector3D in = {1,2,3};
    for (auto _ : state) {
        Vector3D out = MultiplyByTwo_Value(in);
    }
}

void MultiplyVector_Ref(benchmark::State& state) {
    Vector3D in = {1,2,3};
    for (auto _ : state) {
        Vector3D out = MultiplyByTwo_Ref(in);
    }
}
``` 

![](img/multiply_vector.png)

总之，我们应该根据情况选择“值语义”和“引用语义”，以实现更好的性能和简化代码。

有趣！这次使用 `const&` 没有任何好处。

当您复制一个不需要堆分配并且小于几十个字节的对象时，您将不会注意到传递它们的引用带来的任何好处。

另一方面，它永远不会变慢，因此，如果您有疑问，请始终使用 `const&`。 

而将基本类型通过 const 引用传递会生成额外的指令（请参见https://godbolt.org/z/-rusab），但编译时使用 `-O3` 选项可以优化掉。

我的经验法则是：永远不要传递大小为 8 字节或更少（整数、双精度浮点数、字符串等）的参数的引用。

由于我们确定其好处为 0%，编写像这样的东西毫无意义，而且“丑陋”：

```C++
void YouAreTryingTooHardDude(const int& a, const double& b);
```
