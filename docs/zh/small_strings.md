# 小字符串优化

还记得我说过“字符串实际上是伪装成 `std::vector<char>` 的吗”？

实际上，聪明的人意识到你可能会将小的字符串存储在已经分配的内存中。

考虑到在 64 位平台上，`std::string` 的大小为 **24 字节**（用于存储数据指针、大小和容量），一些非常酷的技巧允许我们在需要分配内存之前**静态地**存储多达 23 个字节。

这在性能方面有巨大的影响！

![](img/relax_sso.jpg)

对于好奇的人，以下是一些关于实现的详细信息：

- [SSO-23](https://github.com/elliotgoodrich/SSO-23)
- [CppCon 2016: “The strange details of std::string at Facebook"](https://www.youtube.com/watch?v=kPR8h4-qZdk)

根据编译器的版本，您可能只能拥有少于 23 个字节，这是理论上的限制。

## 示例

```C++
const char* SHORT_STR = "hello world";

void ShortStringCreation(benchmark::State& state) {
  // 反复创建一个字符串。
  // 这是因为“短字符串优化”有效，没有内存分配。
  for (auto _ : state) {
    std::string created_string(SHORT_STR);
  }
}

void ShortStringCopy(benchmark::State& state) {
  // 在这里我们只创建一次字符串，但反复复制。
  // 为什么它比 ShortStringCreation 慢得多呢？
  // 编译器，显然，比我聪明。
  std::string x; // 创建一次
  for (auto _ : state) {
    x = SHORT_STR; // 复制
  }
}

const char* LONG_STR = "this will not fit into small string optimization";

void LongStringCreation(benchmark::State& state) {
  // 长字符串肯定会触发内存分配
  for (auto _ : state) {
    std::string created_string(LONG_STR);
  }
}

void LongStringCopy(benchmark::State& state) {
  // 现在我们看到实际的加速，当多次重复使用同一个字符串时
  std::string x;
  for (auto _ : state) {
    x = LONG_STR;
  }
}
```

正如您可能注意到的那样，如果字符串很短但却需要分配内存，我的聪明尝试“我不会每次都创建一个新字符串”将非常失败，但是如果字符串需要分配内存，则会产生巨大的影响。

![](img/sso_in_action.png)



