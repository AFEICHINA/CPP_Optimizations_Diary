# 玩弄回文词

这篇文章可能没有其他文章那么有趣和可重复利用，但我认为它仍然是一个不错的示例，可以展示如何通过减少分支数量来加速代码。

“if”语句非常快，通常我们不应该担心其运行时成本，但在某些情况下，避免它可能会提供明显的改进。

## 编码面试...

在撰写本文时，我发现自己处于美妙的求职世界中。因此，您可能知道这意味着什么：编码面试！

我对此感到满意，但前几天一个面试我的人问了以下问题：

> 您能否编写一个函数以查找字符串是否为[回文](https://en.wikipedia.org/wiki/Palindrome)?

然后我想起了... 

![fizzbuss](../img/fizzbuzz.jpg)

公平地说，我相信他是个好人，他只是打破僵局。他肯定是出于最好的意图，但我认为如果我们看一些我在实际生产中编写的真实代码，那对我们两个人都更有成效！

尽管如此，这就是答案：

```C++

bool IsPalindrome(const std::string& str)
{
    const size_t N = str.size();
    const size_t N_half = N / 2;
    for(size_t i=0; i<N_half; i++)
    {
        if( str[i] != str[N-1-i])
        {
            return false;
        }
    }
    return true;
}
```

简单易行 :)

无聊！

我们来玩一下！

## 更快的IsPalindrome()

原始版本显然是我们可以做到的最好：

- 零拷贝。
- 尽可能快地停止循环。
- 处理所有边缘情况。

但我意识到有一种方法可以使它更快，减少“if”条款的数量。 这可以通过使用整个“单词”即将单个字节存储在更大的数据类型中轻松实现。

例如，让我们使用类型`uint32_t`一次操作4个字节。

得到的实现将是：

```C++
#include <byteswap.h>

inline bool IsPalindromeWord(const std::string& str)
{
    const size_t N = str.size();
    const size_t N_half = (N/2);
    const size_t S = sizeof(uint32_t);
    // number of words of size S in N_half
    const size_t N_words = (N_half / S);

    // example: if N = 18, half string is 9 bytes and
    // we need to compare 2 pairs of words and 1 pair of chars

    size_t index = 0;

    for(size_t i=0; i<N_words; i++)
    {
        uint32_t word_left, word_right;
        memcpy(&word_left, &str[index], S);
        memcpy(&word_right, &str[N - S - index], S);

        if( word_left != bswap_32(word_right))
        {
            return false;
        }
        index += S;
    }
    // remaining bytes.
    while(index < N_half)
    {
        if( str[index] != str[N-1-index])
        {
            return false;
        }
        index++;
    }
    return true;
}
```

这里发生了什么？

我们存储4个字节的“块”（单词）并仅为每个单词调用比较运算符**一次**。

为了以“镜像”的方式颠倒字节的顺序，我们使用内置函数[bswap_32](https://man7.org/linux/man-pages/man3/bswap_32.3.html)。

请注意，在我的基准测试中，手工制作的此函数的实现速度**同样快**。

以下是您提供的代码：

```C++
inline uint32_t Swap(const uint32_t& val)
{
  union {
    char c[4];
    uint32_t n;
  } data;
  data.n = val;
  std::swap(data.c[0], data.c[3]);
  std::swap(data.c[1], data.c[2]);
  return data.n;
}
```

在您的基准测试中，对于足够长的字符串（超过8个字节），相对较短字符串的性能提升约为50％，而对于长字符串的性能提升高达150％。实际上，对于非常长的字符串，我们可以使用128位或256位的字来进行操作。这可以使用 [SIMD](https://stackoverflow.blog/2020/07/08/improving-performance-with-simd-intrinsics-in-three-use-cases/) 实现，但这不是本文的目的。

总结一下，尽管本文展示了一种优化分支数以提高性能的例子，但除非出于性能原因必须这么做，否则应该始终将代码的简洁性和可读性放在首位。



