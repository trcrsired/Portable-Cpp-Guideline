# 可移植的C++指南
您有兴趣编写可移植和高效的C++代码吗？不要再看其他指南了，来看看这个指南吧！与其他指南（如[C++核心指南](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)）不同，我们的指南优先考虑可移植性并避免有问题的编码实践。

请记住，没有所谓的零成本或零开销（运行时）抽象。请提防任何声称否则的人！即使是借用检查器和C++异常这样的功能也有运行时开销（请参见：[当零成本抽象并非零成本时](https://blog.polybdenum.com/2021/08/09/when-zero-cost-abstractions-aren-t-zero-cost.html)和[零成本异常实际上并非零成本）](https://devblogs.microsoft.com/oldnewthing/20220228-00/?p=106296)。

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

