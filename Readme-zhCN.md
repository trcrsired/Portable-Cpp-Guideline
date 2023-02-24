# 可移植的C++指南
您有兴趣编写可移植和高效的C++代码吗？不要再看其他指南了，来看看这个指南吧！与其他指南（如[C++核心指南](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)）不同，我们的指南优先考虑可移植性并避免有问题的编码实践。

请记住，没有所谓的零成本或零开销（运行时）抽象。请提防任何声称否则的人！即使是借用检查器和C++异常这样的功能也有运行时开销（请参见：[当零成本抽象并非零成本时](https://blog.polybdenum.com/2021/08/09/when-zero-cost-abstractions-aren-t-zero-cost.html)和[零成本异常实际上并非零成本](https://devblogs.microsoft.com/oldnewthing/20220228-00/?p=106296))。

请注意，当前的C++标准是C++23。

## 独立

为了编写真正可移植的 C++ 代码，重要的是要坚持使用独立的 C++ 部分并使用平台宏来防范宿主环境。cppreference 网站页面[独立与宿主实现](https://zh.cppreference.com/w/cpp/freestanding)提供了一个有用的指南，介绍了独立实现和此环境中可用的函数。

但是，重要的是要注意，您可能认为应该可用的许多标头可能不存在于独立环境中。此外，某些可用的标头可能具有限制或问题，这可能会阻止您使用它们。

如果您想确保您的代码在独立环境中运行，请尝试使用 x86_64-elf-gcc 编译器。此编译器可在以下 GitHub 存储库上获得：[windows-hosted-x86_64-elf-toolchains](https://github.com/trcrsired/windows-hosted-x86_64-elf-toolchains)。

### 避免在 C++23 之前使用 ```std::addressof``` 以及避免使用 ```operator &```

这是因为 C++ 允许对 ```operator &``` 进行重载，这是一个历史错误。为了克服这个问题，ISO C++11 标准引入了 ```std::addressof```。然而，直到 C++17，```std::addressof``` 才成为 ```constexpr```。不幸的是，```std::addressof``` 是 ```<memory>``` 标头的一部分，而该标头不是独立的，这使得在 C++23 之前在 C++ 中实现它变得不可能，除非使用编译器魔法。在 C++23 中，```<memory>``` 标头终于变成了部分独立的，但是重要的是要更新到最新的工具链以确保其可用性。值得注意的是，函数指针不能使用 ```std::addressof```，必须使用 ```operator &``` 来获取它们的地址。

```cpp
//bad! memory header may not be available before C++23.
//#include <memory> may also hurt compilation speed
#include<memory>

int main()
{
	int a{};
	int *pa{std::addressof(a)};//bad! memory header may does not exist
	int *pb{&a};//ok but still bad! operator & can be overloaded for class object types.
}
/*
Compilers give us errors.

x86_64-elf-g++ -v
gcc version 13.0.0 20220520 (experimental) (GCC)

D:\Desktop>x86_64-elf-g++ -c addressof.cc -O3 -std=c++23 -s -flto
addressof.cc:1:9: fatal error: memory: No such file or directory
    1 | #include<memory>
      |         ^~~~~~~~
compilation terminated.
*/
```

```cpp
int main()
{
	int a{};
	int *pc{__builtin_addressof(a)};
//Ok. GCC, clang and MSVC all support __builtin_addressof. It is safe to use it.
//Maybe some compilers don't support __builtin_addressof? Then we are screwed up for sure. That is the C++ standard to blame
}

/*
D:\Desktop>x86_64-elf-g++ -c builtin_addressof.cc -O3 -std=c++23 -s -flto

Compilation success
*/

```

### 避免在 C++23 之前使用 ```std::move```、```std::forward``` 和 ```std::move_if_noexcept```

这些函数被定义在 ```<utility>``` 头文件中，该头文件直到 C++23 才成为完全独立的。为确保最大的可移植性，建议您自己编写这些函数。然而，值得注意的是，最近版本的 Clang 编译器（从版本 15 开始）已经添加了一个补丁，将这些函数视为编译器的魔法。因此，通过自己编写这些函数可能无法实现 100% 的效率。


## 其他问题

### C++中```inline```的作用：防止ODR违规，而非用于提示编译器优化或展开代码。

在C++中，```inline```关键字的目的是防止ODR（One Definition Rule）违规，而不是提示编译器优化或展开代码。然而，由于其与C语言的不同含义，该关键字的使用常常令人困惑。如果一个具有相同签名的函数在多个翻译单元中被定义，链接器将无法将它们链接起来。当一个函数被标记为```inline```时，链接器可以舍弃任何具有相同签名的函数并仅保留一个副本，假设它们的作用相同。然而，如果它们的作用不同，则会导致未定义的行为。此外，将函数标记为```inline```允许GCC和clang在未使用时避免发出该函数，这对于防止死代码很重要。为了更好地理解```inline```，请观看[What does the keyword "inline" REALLY mean in C++?](https://www.youtube.com/watch?v=6WjKIStrc80)视频并参考这篇文章[A noinline inline function? What sorcery is this?](https://devblogs.microsoft.com/oldnewthing/20200521-00/?p=103777)。

### 为确保最大的可移植性，建议在尽可能多的GCC跨/加拿大工具链上构建和测试代码。

这将帮助您及早发现潜在问题，并允许您开发更健壮和可移植的代码库。通过在各种平台上进行测试，您可以确保您的代码在不同的系统和架构上表现一致，从而最大程度地减少意外行为和错误的风险。
