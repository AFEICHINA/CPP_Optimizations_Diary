# 它只是一个字符串：我应该担心吗？

相比在 **C** 中处理时必须面对的指针和长度的混乱，`std::string` 是一个很好的抽象。

我开玩笑的，C 开发者，我们爱你们！
> 或者说我们同情你们，这取决于你想如何看待它。

如果您仔细考虑一下，`std::string` 不就是 `std::vector<char>` 的伪装形式吗？还带有一些有用的文本工具，但没有太多其他东西。

一方面，**确实如此**，但接下来就是所谓的**小字符串优化（SSO）**。

[在此阅读有关 SSO 的更多信息](small_strings.md)。

我想在这里向您展示的是，作为任何可能需要内存分配的对象，您都必须使用类似容器的最佳实践（即使可以争辩地说，往往您需要担心得少）。

## ToString

```c++
enum Color{
    BLUE,
    RED,
    YELLOW
};

std::string ToStringBad(Color c)
{
    switch(c) {
    case BLUE:   return "BLUE";
    case RED:    return "RED";
    case YELLOW: return "YELLOW";
    }
}

const std::string& ToStringBetter(Color c)
{
    static const std::string color_name[3] ={"BLUE", "RED", "YELLOW"};
    switch(c) {
    case BLUE:   return color_name[0];
    case RED:    return color_name[1];
    case YELLOW: return color_name[2];
    }
}
```

这只是一个例子，展示了如果可能的话，您应该不要一遍又一遍地创建字符串。当然，我可以听到您争论：

“Davide，你忘记了返回值优化吗？” 

没有。但 `const&` 始终保证是最高效的选择，所以为什么要冒险呢？

![](../img/tostring.png)


## 重用临时字符串

下面是一个类似的例子，在其中我们**潜在地**回收先前分配的内存。

您不能保证使用后者速度更快，但您可能需要尝试。

```c++
// Create a new string every time (even if return value optimization may help)
static std::string ModifyString(const std::string& input)
{
    std::string output = input;
    output.append("... indeed");
    return output;
}
// Reuse an existing string that MAYBE, have the space already reserved
// (or maybe not..)
static void ModifyStringBetter(const std::string& input, std::string& output)
{
    output = input;
    output.append("... indeed");
}
```

然后，就如预期的那样……

![](../img/modifystring.png)

