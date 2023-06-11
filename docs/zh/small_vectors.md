# 小型向量优化

到目前为止，我希望我已经说服了你，`std::vector` 是你应该考虑使用的第一个数据结构，除非你需要一个关联容器。

但是即使我们巧妙地使用 `reserve` 来防止不必要的堆分配和复制，在开始时仍会有至少**一次**堆分配。我们能做得更好吗？

当然可以！如果您已经阅读了关于[小字符串优化](../../small_strings)的内容，您就知道这将引申出哪些内容。

# “静态”向量和“小”向量

当您确定您的向量很小并且即使在最坏的情况下也将保持相对较小的大小时，您可以在栈中分配整个元素数组，跳过昂贵的堆分配。

您可能认为这很不可能，但事实上它发生的频率比您所想象的要高得多。就在两周前，我在我们的库中识别出了完全相同的模式，其中某些向量的大小可以是 0 至 8 之间的任何数字。

30 分钟的重构提高了我们软件的总体速度 20%！

总结一下，您希望拥有这位先生熟悉的 API：
```C++
std::vector<double> my_data; // 至少有一个堆分配，除非大小为 0 
```
当实际上，在幕后，您想要这个：
```C++
double my_data[MAX_SIZE]; // 没有堆分配 
int size_my_data;
```

让我们看一个简单而朴素的 `StaticVector` 实现：

```C++
#include <array>
#include <initializer_list>

template <typename T, size_t N>
class StaticVector
{
public:

  using iterator       = typename std::array<T,N>::iterator;
  using const_iterator = typename std::array<T,N>::const_iterator;

  StaticVector(uint8_t n=0): _size(n) {
    if( _size > N ){
      throw std::runtime_error("SmallVector overflow");
    }
  }

  StaticVector(const StaticVector& other) = default;
  StaticVector(StaticVector&& other) = default;

  StaticVector(std::initializer_list<T> init)
  {
    _size = init.size();
    for(int i=0; i<_size; i++) { _storage[i] = init[i]; }
  }

  void push_back(T val){
    _storage[_size++] = val;
    if( _size > N ){
      throw std::runtime_error("SmallVector overflow");
    }
  }

  void pop_back(){
    if( _size == 0 ){
      throw std::runtime_error("SmallVector underflow");
    }
    back().~T(); // 调用析构函数
    _size--;
  }

  size_t size() const { return _size; }

  void clear(){ while(_size>0) { pop_back(); } }

  T& front() { return _storage.front(); }
  const T& front() const { return _storage.front(); }

  T& back() { return _storage[_size-1]; }
  const T& back() const { return _storage[_size-1]; }

  iterator begin() { return _storage.begin(); }
  const_iterator begin() const { return _storage.begin(); }

  iterator end() { return _storage.end(); }
  const_iterator end() const { return _storage.end(); }

  T& operator[](uint8_t index) { return _storage[index]; }
  const T& operator[](uint8_t index) const { return _storage[index]; }

  T& data() { return _storage.data(); }
  const T& data() const { return _storage.data(); }

private:
  std::array<T,N> _storage;
  uint8_t _size = 0;

```

**StaticVector** 看起来像是一个 `std::vector`，但实际上它是……

![](../img/inconceivably.jpg)

有时候，向量式容器最多只会有 **N** 个元素，但我们并不“绝对确定”。

我们仍然可以使用一个通常称为 **SmallVector** 的容器，它将使用来自栈的预分配内存作为其前 N 个元素，并且仅当容器需要进一步增长时，才会使用堆分配创建新的存储块。

## 在实际应用中的StaticVector和SmallVector

事实证明，这些技巧是众所周知的，并且可以在许多流行的库中找到已经实现并准备好使用：

- [Boost::container](https://www.boost.org/doc/libs/1_73_0/doc/html/container.html)。如果存在，Boost 当然拥有。
- [Abseil](https://github.com/abseil/abseil-cpp/tree/master/absl/container)。它们被称为 `fixed_array` 和 `inlined_vector`。
- 出于教学目的，您可以查看 LLVM 内部使用的 [SmallVector](https://github.com/llvm/llvm-project/blob/master/llvm/include/llvm/ADT/SmallVector.h)。