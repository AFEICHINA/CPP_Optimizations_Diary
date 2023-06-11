# 字符串连接

警告：在阅读本文之前，请记住我们一开始提到的规则 #1。

**仅当您使用性能分析工具观察到明显的开销时，才优化您的代码。**
说完这个...

正如我们所说，字符串比字符向量多了一些东西，因此可能需要堆分配来存储所有元素。

在 C++ 中连接字符串非常容易，但有一些东西需要我们注意。

## “默认”连接

看看这个熟悉的代码行。

```C++
std:string big_string = first + " " + second + " " + third;

// Where...
// std::string first("This is my first string.");
// std::string second("This is the second string I want to append.");
// std::string third("This is the third and last string to append."); 
```

注意到什么可疑的地方了吗？考虑堆分配...

![](img/spider_senses.png)

让我这样重写它：

```C++
std:string big_string = (((first + " ") + second) + " ") + third;
```

希望您已经明白了。为了连接这样长度的字符串，您将需要多次进行堆分配，并从旧内存块复制到新内存块。

如果只有 `std::string` 有一个类似于 `std::vector::reserve()` 的方法 :(

等等... [这是什么](https://en.cppreference.com/w/cpp/string/basic_string/reserve)？

## “手动”连接

让我们使用 `reserve` 函数来将堆分配的次数减少到一次。

我们可以提前计算 `big_string` 需要的总字符数，并像这样进行预留：

```C++
    std::string big_one;
    big_one.reserve(first_str.size() + 
                    second_str.size() + 
                    third_str.size() + 
                    strlen(" ")*2 );

    big_one += first;
    big_one += " ";
    big_one += second;
    big_one += " ";
    big_one += third;
```

我知道你在想什么，你是100%正确的。

![](img/feel_bad.jpg)

那是一段可怕的代码......比默认字符串连接方式**快了 2.5 倍**！

## 可变参数连接

我们能否创建一个字符串连接函数，既快速，又可重用，同时阅读性强？

我们可以，但我们需要使用一些现代 C++ 的重兵：**可变模板（variadic templates）**。

如果您不熟悉它们，这里有一篇非常[好的关于可变模板的文章](https://arne-mertz.de/2016/11/more-variadic-templates/)，您应该阅读一下。

```C++
//--- functions to calculate the total size ---
size_t StrSize(const char* str) {
  return strlen(str);
}

size_t StrSize(const std::string& str) {
  return str.size();
}

template <class Head, class... Tail>
size_t StrSize(const Head& head, Tail const&... tail) {
  return StrSize(head) + StrSize(tail...);
}

//--- functions to append strings together ---
template <class Head>
void StrAppend(std::string& out, const Head& head) {
  out += head;
}

template <class Head, class... Args>
void StrAppend(std::string& out, const Head& head, Args const&... args) {
  out += head;
  StrAppend(out, args...);
}

//--- Finally, the function to concatenate strings ---
template <class... Args> 
std::string StrCat(Args const&... args) {
  size_t tot_size = StrSize(args...);
  std::string out;
  out.reserve(tot_size);

  StrAppend(out, args...);
  return out;
}
```

对于训练有素的眼睛，这是一段非常复杂的代码。但好消息是很容易使用：

```C++
std:string big_string = StrCat(first, " ", second, " ", third );
```

那么这到底有多快呢？

![](img/string_concatenation.png)

使用可变模板的版本比“丑陋”的手动连接略慢的原因是......

我不知道！

我知道的是，它比默认的快两倍，并且不是一个难以阅读的混乱。

## 在你复制和粘贴我的代码之前...

我的 `StrCat` 实现非常有限，我只想说明一点：在 C++ 中要注意字符串连接。

尽管如此，请不要再想两次，使用 [{fmt}](https://github.com/fmtlib/fmt) 代替。

它不仅是一个易于集成、文档完备且非常快速的格式化字符串库。

它还是 [C++20 std::format](https://en.cppreference.com/w/cpp/utility/format) 的一个实现。

这意味着您可以编写可读性强、性能优异且具有未来兼容性的代码！







