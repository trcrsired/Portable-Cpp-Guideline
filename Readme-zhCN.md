# 可移植的C++指南
您有兴趣编写可移植和高效的C++代码吗？不要再看其他指南了，来看看这个指南吧！与其他指南（如[C++核心指南](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)）不同，我们的指南优先考虑可移植性并避免有问题的编码实践。

请记住，没有所谓的零成本或零开销（运行时）抽象。请提防任何声称否则的人！即使是借用检查器和C++异常这样的功能也有运行时开销（请参见：当零成本抽象并非零成本时和零成本异常实际上并非零成本）。

请注意，当前的C++标准是C++23。

## 独立

为了编写真正可移植的 C++ 代码，重要的是要坚持使用独立的 C++ 部分并使用平台宏来防范宿主环境。cppreference 网站页面[独立与宿主实现](https://zh.cppreference.com/w/cpp/freestanding)提供了一个有用的指南，介绍了独立实现和此环境中可用的函数。

但是，重要的是要注意，您可能认为应该可用的许多标头可能不存在于独立环境中。此外，某些可用的标头可能具有限制或问题，这可能会阻止您使用它们。

如果您想确保您的代码在独立环境中运行，请尝试使用 x86_64-elf-gcc 编译器。此编译器可在以下 GitHub 存储库上获得：[windows-hosted-x86_64-elf-toolchains](https://github.com/trcrsired/windows-hosted-x86_64-elf-toolchains)。

