# 可移植的C++指南
您有兴趣编写可移植和高效的C++代码吗？不要再看其他指南了，来看看这个指南吧！与其他指南（如[C++核心指南](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)）不同，我们的指南优先考虑可移植性并避免有问题的编码实践。

请记住，没有所谓的零成本或零开销（运行时）抽象。请提防任何声称否则的人！即使是借用检查器和C++异常这样的功能也有运行时开销（请参见：[当零成本抽象并非零成本时](https://blog.polybdenum.com/2021/08/09/when-zero-cost-abstractions-aren-t-zero-cost.html)和[零成本异常实际上并非零成本](https://devblogs.microsoft.com/oldnewthing/20220228-00/?p=106296))。

请注意，当前的C++标准是C++26。

## 独立

为了编写真正可移植的 C++ 代码，重要的是要坚持使用独立的 C++ 部分并使用平台宏来防范宿主环境。cppreference 网站页面[独立与宿主实现](https://zh.cppreference.com/w/cpp/freestanding)提供了一个有用的指南，介绍了独立实现和此环境中可用的函数。

但是，重要的是要注意，您可能认为应该可用的许多标头可能不存在于独立环境中。此外，某些可用的标头可能具有限制或问题，这可能会阻止您使用它们。

如果您想确保您的代码在独立环境中运行，请尝试使用 x86_64-elf-gcc 编译器。此编译器可在以下 GitHub 存储库上获得：[windows-hosted-x86_64-elf-toolchains](https://github.com/trcrsired/windows-hosted-x86_64-elf-toolchains)。

### 避免在 C++23 之前使用 ```std::addressof``` 以及避免使用 ```operator &```

这是因为 C++ 允许对 ```operator &``` 进行重载，这是一个历史错误。为了克服这个问题，ISO C++11 标准引入了 ```std::addressof```。然而，直到 C++17，```std::addressof``` 才成为 ```constexpr```。不幸的是，```std::addressof``` 是 ```<memory>``` 标头的一部分，而该标头不是独立的，这使得在 C++23 之前在 C++ 中实现它变得不可能，除非使用编译器魔法。在 C++23 中，```<memory>``` 标头终于变成了部分独立的，但是重要的是要更新到最新的工具链以确保其可用性。值得注意的是，函数指针不能使用 ```std::addressof```，必须使用 ```operator &``` 来获取它们的地址。

```cpp
//糟糕！在 C++23 之前可能无法使用 memory 头文件。
//#include <memory> 也可能会影响编译速度。
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

```cpp
//不好！utility头文件可能不可用。
//#include <utility>可能还会影响编译速度
#include <utility>

int main()
{
    int a{};
    int b{std::move(a)};
}

/*
D:\Desktop>x86_64-elf-g++ -c move.cc -O3 -std=c++23 -s -flto
move.cc:1:9: fatal error: utility: No such file or directory
    1 | #include<utility>
      |         ^~~~~~~~~
compilation terminated.
*/
```

### 避免在 C++26 之前使用 ```std::array```。

类似于应避免使用 ```std::addressof``` 和 ```std::move``` 的原因，```<array>``` 头文件在C++26之前也不是独立的。

```cpp
//糟糕！array头文件可能不可用。
//#include <array>也可能会影响编译速度
#include<array>

int main()
{
	std::array<int,4> a{};
}

/*
D:\Desktop>x86_64-elf-g++ -c move.cc -O3 -std=c++23 -s -flto
move.cc:1:9: fatal error: array: No such file or directory
    1 | #include<array>
      |         ^~~~~~~~~
compilation terminated.
*/
```

解决方案是使用 C 风格的数组。

```cpp
//好的！你应该使用C风格的数组，因为 <array> 不存在。
//cstdint 是存在的。
#include<cstdint>

int main()
{
	::std::int_least32_t arr[4]{};//Ok! ::std::int_least32_t is freestanding
}

/*
x86_64-elf-g++ -c carray.cc -O3 -std=c++23 -s -flto
*/
```

### 考虑使用C++容器、迭代器、算法和分配器的独立替代方案
为了实现最大的可移植性，建议避免所有C++容器、迭代器、算法和分配器。这些组件不是独立的，并且具有与堆和异常处理交互的复杂设计，当包含它们时会产生可移植性问题。尽管被推广为 "现代" C++ 书籍和C++核心指南中的默认容器，但 std::vector 不是独立的，包含它可能会导致可移植性问题。此外，分配器也不是独立的，即使 std::allocator 也不能被视为独立的，因为它与异常的交互。如果禁用异常，异常的依赖关系仍然存在，会导致链接错误或二进制膨胀问题。

```cpp
/// 错误！！！<vector> 不是独立的
#include<vector>
#include<cstdint>

int main()
{
	std::vector<::std::int_least32_t> vec;
}
/*
D:\Desktop>x86_64-elf-g++ -c vector.cc -O3 -std=c++23 -s -flto
vector.cc:1:9: fatal error: vector: No such file or directory
    1 | #include<vector>
      |         ^~~~~~~~
compilation terminated.
*/
```

### 避免在 C++23 之前使用 ```<ranges>``` 和 ```<iterator>```
在 C++23 之前的版本中避免使用 ```<ranges>``` 和 ```<iterator>``` 头文件。尽管诸如 ```std::random_access_range``` 和 ```std::contiguous_range``` 等概念可能很有用，但它们不是独立的，并且可能在所有上下文中都不可用。通常更简单的方法是坚持使用指针。

从 C++23 开始，```<ranges>``` 和 ```<iterator>``` 部分是独立的，意味着它们可以在大多数上下文中使用，除了与流迭代器相关的功能。

### 堆

虽然 ```<new>``` 是独立的，但在其当前状态下并不特别有用。这是因为异常以一种相当混乱的方式抛出，使得在某些上下文中难以依赖它。


### 避免使用 std::nothrow

在C++中避免使用 ```std::nothrow``` 和 ```std::nothrow_t```，因为标准库通过在内部使用抛出的 ```new``` 来实现 ```nothrow new```。这意味着 ```std::nothrow``` 无法提供内存分配不会抛出异常的保证。此外，C++标准不允许重写 ```operator new(std::nothrow_t const&)``` 的默认实现，不像 ```operator new()```。因此，使用 ```std::nothrow``` 没有实际的好处，应该避免使用。此外，在独立环境中应避免使用异常，因此建议实现不抛出异常的自定义内存分配函数。

```cpp
/*
在任何形式中都不要使用 new (std::nothrow_t)。这太糟糕了。

这是来自GCC中libsupc++的实现。
https://github.com/gcc-mirror/gcc/blob/aa2eb25c94cde4c147443a562eadc69de03b1556/libstdc%2B%2B-v3/libsupc%2B%2B/new_opnt.cc#L31
*/

_GLIBCXX_WEAK_DEFINITION void *
operator new (std::size_t sz, const std::nothrow_t&) noexcept
{
  // _GLIBCXX_RESOLVE_LIB_DEFECTS
  // 206. operator new(size_t, nothrow) may become unlinked to ordinary
  // operator new if ordinary version replaced
  __try
    {
      return ::operator new(sz);
    }
  __catch (...)
    {
      // N.B. catch (...) means the process will terminate if operator new(sz)
      // exits with a __forced_unwind exception. The process will print
      // "FATAL: exception not rethrown" to stderr before exiting.
      //
      // If we propagated that exception the process would still terminate
      // (because this function is noexcept) but with a less informative error:
      // "terminate called without active exception".
      return nullptr;
    }
}
```

我们可以看到 C++ 标准库通过在内部使用抛出版本的 ```operator new``` 来实现不抛出版本的 ```operator new```。这完全没有用处。

### 默认堆的实现可能不可用。

简而言之，您不能假设存在可用的默认堆。

原因：
1. 可能根本没有可用的堆。虽然您可以说您总是可以提供一个，但现实情况并非总是如此。
2. 在某些情况下，可能存在多个堆，例如在操作系统内核中或者在具有 ABI 兼容性问题的 Win32 应用程序中。在这些情况下，使用任何默认值可能都是错误的选择。例如，Windows 内核提供了不同的堆，其中一个是中断安全的但空间有限，而其他堆则不是中断安全的但空间充足。
3. C++ 不提供堆的线程本地实现，并且默认堆可能效率非常低下。当然，线程本地存储可能不可用，但这是一个单独的问题。

对于可移植的代码库，强制使用默认堆实现永远不会奏效，所以最好避免使用 ```new```。

### 避免处理栈或堆分配失败。
### 最好避免处理栈或堆分配失败。C 和 C++ 的一个设计缺陷是栈耗尽是未定义行为，而堆耗尽则不是。这造成了不一致性，并可能导致许多问题。

#### 没有办法总是处理堆分配失败以避免崩溃。

一般来说，如果 ```malloc(3)``` 失败，最好简单地调用 ```std::abort```。以下是一些原因：

1. 析构函数可能会再次分配内存，导致无法预测的结果。
2. GCC libsupc++ 实现使用了一个紧急堆，但即使你实现了一个紧急堆，它在许多情况下仍可能调用 ```std::terminate```，导致崩溃。为了验证这一点，我从 ```libsupc++``` 中删除了紧急堆，并没有发现问题，包括没有ABI问题。
3. 你下面的许多库，包括 glibc，都调用 ```xmalloc```，这仍然会导致 ```malloc(3)``` 失败时崩溃。除非你控制所有的源代码，否则你无法避免这个问题。
4. 类似 Linux 这样的操作系统会过度承诺并杀死你的进程，如果你遇到分配失败。一般来说，像 C++ 这样的编程语言无法以任何有意义的方式处理堆栈或堆上的分配失败。
5. C++ 的 ```new``` 操作符会抛出异常，C++ 代码库通常使用 new 在堆上分配内存，从而创建了看不见的代码路径和潜在的异常安全 bug。

#### 如果要加载大文件，请考虑使用内存映射 API 而不是将它们加载到内存中。
将大文件，如图像，加载到内存中可能看起来是一个合理的方法，但通常不是最佳解决方案。相反，建议使用系统调用，如 ```fstat(2)``` 或 Linux 的 ```statx``` 来获取有关文件的信息，并使用 ```mmap(2)``` 系统调用进行内存映射。

##### 优势
1. 内存映射避免了在 32 位机器上的文件大小溢出问题。
2. 它避免了对 CRT 堆的干扰，并且永远不会触发来自 ```malloc``` 或 ```new``` 的分配失败。
3. 与将整个文件加载到 ```std::string``` 中相比，它避免了将内容从内核空间复制到用户空间。
4. 内存映射允许文件在不同进程之间共享，节省物理内存。
5. 使用 ```fseek(3)``` 或 ```seek(2)``` 加载文件可能会导致更多 ```TOCTOU``` 安全漏洞。
6. 如果您不写入写时复制页面，过度承诺的可能性较小。
7. 内存映射创建了一个 ```std::contiguous_range```，对许多工作流程非常有用。
8. 如果使用私有页面加载页面，则可以在不更改文件内容的情况下向内存映射内存中写入。但是，向内存区域写入内容将触发页错误，并且操作系统内核将为您的进程分配新页面。

```cpp
/*
不好。使用 fseek(3) 和 fread(3) 加载文件
https://stackoverflow.com/questions/174531/how-to-read-the-content-of-a-file-to-a-string-in-c
*/

char * buffer = 0;
long length;
FILE * f = fopen (filename, "rb");

if (f)
{
  fseek (f, 0, SEEK_END);
  length = ftell (f);
  fseek (f, 0, SEEK_SET);
  buffer = malloc (length);
  if (buffer)
  {
    fread (buffer, 1, length, f);
  }
  fclose (f);
}

if (buffer)
{
  // 在这里开始处理你的数据 / 提取字符串...
}
```

像 ```fast_io``` 这样的库为文件加载提供了直接支持，并处理了所有的边界情况，包括对不提供内存映射的平台（如 WebAssembly）的透明支持。内存是可写的。

[fast_io](https://github.com/cppfastio/fast_io)

```cpp
/*
好！使用内存映射的 native_file_loader
https://github.com/cppfastio/fast_io/blob/master/examples/0006.file_io/file_loader.cc
*/
#include<fast_io.h>

int main()
{
	fast_io::native_file_loader loader(u8"a.txt");
	//这将通过内存映射将整个 a.txt 加载到内存中。
	/*
	这是文件的连续范围。
	你可以做这些事情：
	std::size_t sum{};
	for(auto e:loader)
		sum+=e;	
	*/
}
```

### 避免使用 ```std::unique_ptr```

1. 对 ```std::unique_ptr``` 的过度依赖鼓励过度使用面向对象编程，延续了之前存在的问题。
2. 根据 Google 的说法，使用 ```std::unique_ptr``` 可能导致显著的性能效率低下，在某些大型服务器宏基准测试中观察到的潜在改进可达到 1.6%。这还导致了稍微较小的二进制文件大小。有关更多信息，请参考 [Enable ```std::unique_ptr [[clang::trivial_abi]]```](https://libcxx.llvm.org/DesignDocs/UniquePtrTrivialAbi.html) 上的 libc++ 文档。这表明在微观性能方面考虑时，```std::unique_ptr``` 是非常低效的。
3. 当用于 pimpl 时，不推荐使用 ```std::unique_ptr```，因为会导致编译速度问题。相反，模块化的使用可以解决这个问题。此外，值得注意的是，由于前述问题中的 ABI bug，```std::unique_ptr``` 不是 ABI 稳定的。
4. 虽然 ```std::unique_ptr``` 有助于解决内存泄漏问题，但无法修复类型混淆或 vptr 注入等问题。
5. 使用 ```std::unique_ptr``` 管理超出内存的资源通常效果不佳，因为 API 通常不支持特定类型，如 Unix 文件描述符或 SQLite3。编写一个类来包装这些资源，以解决所有相关问题，如忽略错误代码。
6. 包含 ```<memory>``` 头文件可能会降低编译速度。
7. 过度使用 nullptr 可能会发生，当依赖 ```std::unique_ptr``` 表示空状态时。
8. ```std::unique_ptr``` 的删除器可能导致效率低下，并且很难避免意外性能下降，无论删除器是 lambda、函数指针还是函数对象。
9. 在使用 ```std::unique_ptr``` 将不可移动对象变为可移动之前，重要的是要质疑为什么该类型首先是不可移动的。
10. 对于 ```std::unique_ptr<T[]>```，```[]``` 很容易被忘记，导致未定义的行为。
11. 使用 ```std::unique_ptr``` 实现数据结构总是不正确的，并可能导致堆栈溢出。
12. 虽然 ```std::unique_ptr``` 在 C++23 中部分可独立存在，但并不实用，因为 std::make_unique 不是独立的，而 new 也不能保证可用。
13. 最重要的是，```std::unique_ptr<T>``` 缺乏类型丰富性，使得不清楚 ```std::unique_ptr<shape>``` 表示什么意思。写成 T 而不是 ```std::unique_ptr<T>``` 将提供更清晰和明确的含义。
14. 总的来说，```std::unique_ptr``` 不是一个智能指针，而是一个可疑的指针类型，可能会有害。

## 异常

C++ 异常处理可能是可移植性面临的最大问题之一。甚至 Linus Torvalds 也曾抱怨过 C++ 异常处理。

[Linus Torvalds on C++](http://harmful.cat-v.org/software/c++/linus)

### 许多平台不提供异常处理功能。

C++ 异常处理的问题不仅仅局限于可移植性。依赖操作系统支持和托管的 C++ 特性使得在各种情境下（包括嵌入式系统和操作系统开发）使用起来变得困难。这种限制可能会给希望编写可移植代码的开发人员带来重大挑战。

在某些架构上（如 AVR 和 wasm）不支持异常处理是另一个重大挑战。虽然一些工具（如 [wasm2lua](https://github.com/SwadicalRag/wasm2lua)）允许将 C++ 代码编译成其他语言，但在这些情境中实现 C++ 风格的异常处理可能是一个困难且性能密集的过程。

即使在支持异常处理的平台上，性能损失也可能会很大。对于许多平台来说，SJLJ 异常处理是一种常见的实现方式，它可能会减慢代码的执行速度，不管是在快速路径还是慢速路径上。这可能会使异常处理在对性能要求较高的应用中变得不切实际。

此外，并非每种架构都支持 C++ 异常处理，即使支持，实现难度也可能很大。异常处理需要堆内存，而 C++ 异常的运行时开销巨大（通常约为 100kb），这使得它在许多嵌入式系统中不可接受，因为二进制大小和内存使用是关键因素。

最终，C++ 异常处理引起的二进制文件大小膨胀是许多情境下开发人员面临的另一个挑战。考虑到所有这些问题，开发人员在代码中仔细考虑使用 C++ 异常处理，并在合适的情况下探索替代错误处理机制非常重要。

###  C++ 异常总是会减慢性能，不是“零额外开销”抽象。

1. 许多平台不实现基于表的异常处理模型，这是 C++ 异常处理所必需的。
2. 无论异常处理模型是基于表还是 SJLJ，C++ 异常都可能对编译器优化产生负面影响。
3. C++ 异常可能会对内存局部性产生负面影响，包括 TLB、BTB、缓存和页面局部性，无论异常处理模型如何。
4. C++ 异常处理与 ASLR（地址空间布局随机化）和 PIC（位置无关代码）不兼容，因为加载器必须重新定位所有异常表指针，这可能会破坏 ABI。最近的安全论文讨论了如何使其与 PIC 兼容的方法，但仍可能会破坏 ABI。
5. C 头文件通常没有正确使用 ```noexcept``` 修饰其函数，这不仅会导致性能问题，还会在技术上违反单一定义规则（ODR）。调用未标记为 ```noexcept``` 的 C API 函数会导致未定义行为，无论异常处理模型是什么。
6. 抛出 C++ 异常可能代价高昂，可能比 x86_64 上的 syscall 指令慢 200-300 倍。
7. C++ 异常在多线程系统中效果不佳，随着越来越多的软件变得多线程化，这已经成为一个重大问题。论文 C++ 异常越来越成为问题 提供了更多关于这个问题的信息。
8. 统计数据显示，95% 的异常是由于编程错误导致的，这些情况应该使用断言或甚至 "std::terminate" 来处理，而不是抛出异常，这样可以提高异常安全性和性能。
9. C++ 异常通常被认为是该语言的一个重大痛点，几乎没有什么可取之处。然而，C++ 社区的一些成员对像 Herb Sutter 的 P0709R0 这样的提案抱有希望。有关更多信息，您可以观看他的视频演示：[De-fragmenting C++: Making Exceptions and RTTI More Affordable and Usable - Herb Sutter CppCon 2019](https://www.youtube.com/watch?v=ARYP83yNAWk)

## 输入输出

由于它们的相关缺陷，建议避免使用iostream和stdio。

### iostream和stdio不是独立的。
即使像LLVM libcxx这样的工具链供应商也面临着依赖stdio的自举困难。

### 根据locale设置，iostream和stdio的行为是不一致的。
iostream和stdio的行为可能会因locale设置而随机变化，导致在一个机器上运行良好的程序在另一个机器上失败。

### iostream和stdio缺乏线程安全性。
在运行时更改locale设置可能会导致iostream和stdio出现线程安全问题。此外，iostream缺乏内置的线程感知或锁定机制，使其容易受到多个线程同时访问时的问题影响。

### ```printf```函数族的不当使用可能导致严重的安全漏洞。

```printf```函数族是用于格式化和将输出打印到控制台的强大工具，但如果不正确使用，它们也可能成为严重安全漏洞的来源。printf函数被误用的最常见方式之一是通过格式字符串攻击，这种攻击发生在攻击者能够向printf调用中注入格式说明符时，导致程序输出来自堆栈或其他敏感内存区域的数据。

为了避免这种类型的漏洞，重要的是要始终谨慎使用printf函数。这意味着要仔细验证和清理任何将在printf调用中使用的输入，并避免使用可能由攻击者控制的格式字符串。此外，重要的是始终为要打印的数据使用正确的格式说明符，并避免使用%n格式说明符，因为它可以用于写入任意内存位置。

总的来说，虽然printf函数族是强大且有用的工具，但重要的是要谨慎和细致地使用它们，以避免在代码中引入安全漏洞。

```cpp
inline void foo(std::string const& str)
{
//DANGER! format string vulnerability
	printf(str.c_str());
//DANGER ignore the return value of printf or scanf
}
```

### 避免在您的C++代码中使用```std::endl```。
使用```std::endl```的主要问题之一是它对性能的影响。由于```std::endl```在每次插入换行字符时都会刷新输出缓冲区，如果您需要打印大量输出，则可能效率低下。这可能会减慢程序运行速度，使其响应性降低，特别是如果您使用大型数据集或在性能较慢的计算机上运行程序时。

另一个使用```std::endl```的问题是它可能是多余的。C++输出流包含一种称为```tie```的机制，它会在必要时自动刷新输出缓冲区，例如当缓冲区已满或程序终止时。这意味着手动使用```std::endl```刷新缓冲区通常是不必要的，甚至可能干扰自动刷新机制。

为了避免这些问题，通常最好依赖于内置到C++输出流中的自动刷新机制，并完全避免使用```std::endl```。如果您确实需要在代码中的特定位置刷新缓冲区，您可以使用流对象本身的```flush()```方法。这个方法只会刷新缓冲区，不会插入换行字符，这在您不需要打印新行的情况下可以提高性能。

如果您想要立即打印输出而不使用缓冲，您可以使用无缓冲流。然而，C++流库并不提供对无缓冲流的健壮支持，因此您可能需要使用第三方IO库，比如```fast_io```来实现这一点。

### 避免使用 ```fmt::format```, ```std::format``` 和 C++23 的 ```<print>```

格式化字符串的概念固有地存在缺陷，会导致安全漏洞。尽管 C++23 规定 ```std::format``` 只接受文字量参数，但用户仍然可以通过构建系统中的宏引入代码注入。

此外，```format``` 中的格式符过于复杂，几乎不可能实现高效的实现。这些格式符依赖于 ```std::string```，这不是一个独立的特性。为了可移植性，建议避免使用它们。

### 避免假设 ```int8_t```、```int_least8_t``` 或 ```int_fast8_t``` 不是字符类型。

```cpp
int8_t *p{};
std::cout << p; // 危险！iostream 重载将p视为 char*，这将把p视为字符串。
```

不幸的是，```iostream``` 存在许多类似的隐患。

### 使用内存映射加载大文件，而不是使用 ```seek``` + ```read``` 组合。
参见前文： 如果要加载大文件，请考虑使用内存映射 API 而不是将它们加载到内存中。

## 面向对象编程（OOP）

考虑使用类型擦除而不是面向对象编程（OOP）。Herb Sutter的元类可以简化类型擦除的使用，并在大多数情况下消除对OOP的需求。

### Herb Sutter关于元类的提案
Herb Sutter的元类提案旨在通过一种更高效、更安全的方式扩展C++语言，使程序员能够定义自定义的语言扩展，如类型擦除。元类提供了一种基于用户定义规范在编译时生成代码的方式，而无需使用宏或外部工具。

借助元类，可能会更容易在C++中使用类型擦除，并且在某些情况下减少对面向对象编程(OOP)的依赖性。然而，元类目前尚未成为C++标准的一部分，其确切的设计和实现可能会在未来发生变化。

## 内存安全

确保代码的可靠性和安全性是软件开发的关键方面。为了实现这一目标，开发人员应遵循几个重要的实践。首先，建议始终使用调试器来运行代码。这是因为调试器可以检测到各种问题，如内存错误、未定义行为、数据竞争等。

另一个重要的实践是模糊测试。模糊测试涉及向程序生成随机输入，并监视是否出现意外行为或崩溃。这可以帮助发现在开发过程中可能不会立即显现的错误或安全漏洞。

在进行高效的边界检查方面，建议使用类似 ```_GLIBCXX_ASSERTIONS``` 的宏，而不是对整个C++标准库使用 ```at()``` 方法执行边界检查。这可以显着提高性能并减少开销。[How to make "modern" C++ programs safer](https://www.youtube.com/watch?v=FAt8KVlNB7E)

最后，内存标记是防止内存安全错误的强大工具。通过为内存分配添加标记，开发人员可以检测到缓冲区溢出、释放后使用错误等常见内存安全问题。这可以帮助防止利用并提高整体程序稳定性。

在使用GCC和Clang进行编译时，建议包含以下标志：

-Wall -Wextra -Wpedantic -Wmisleading-indentation -Wunused -Wuninitialized -Wshadow -Wconversion

以下是每个标志的扩展解释：

-Wall：启用默认情况下被认为足够安全的所有警告。但是，这并不启用GCC或Clang能够生成的所有警告。

-Wextra：启用更多的警告，包括一些未被 -Wall 启用的警告，如有关未初始化变量和未使用函数参数的警告。

-Wpedantic：启用关于非标准代码的警告，按照相关语言标准的定义。这有助于确保代码在不同的编译器和平台之间的可移植性。

-Wmisleading-indentation：警告可能具有误导性缩进的情况，这可能导致难以阅读和理解的代码。

-Wunused：警告未使用的变量、函数和其他实体。这有助于识别不再需要的代码，或尚未完成的代码。

-Wuninitialized：警告使用未初始化的变量。这有助于捕获可能导致未定义行为的潜在错误。

-Wshadow：警告变量在外部作用域中与同名变量重叠。这有助于避免混淆和意外行为。

-Wconversion：警告可能导致数据或精度丢失的隐式转换。这有助于捕获潜在的错误并提高代码质量。

## 其他问题

### C++中```inline```的作用：防止ODR违规，而非用于提示编译器优化或展开代码。

在C++中，```inline```关键字的目的是防止ODR（One Definition Rule）违规，而不是提示编译器优化或展开代码。然而，由于其与C语言的不同含义，该关键字的使用常常令人困惑。如果一个具有相同签名的函数在多个翻译单元中被定义，链接器将无法将它们链接起来。当一个函数被标记为```inline```时，链接器可以舍弃任何具有相同签名的函数并仅保留一个副本，假设它们的作用相同。然而，如果它们的作用不同，则会导致未定义的行为。此外，将函数标记为```inline```允许GCC和clang在未使用时避免发出该函数，这对于防止死代码很重要。为了更好地理解```inline```，请观看[What does the keyword "inline" REALLY mean in C++?](https://www.youtube.com/watch?v=6WjKIStrc80)视频并参考这篇文章[A noinline inline function? What sorcery is this?](https://devblogs.microsoft.com/oldnewthing/20200521-00/?p=103777)。

### 为确保最大的可移植性，建议在尽可能多的GCC跨/加拿大工具链上构建和测试代码。

这将帮助您及早发现潜在问题，并允许您开发更健壮和可移植的代码库。通过在各种平台上进行测试，您可以确保您的代码在不同的系统和架构上表现一致，从而最大程度地减少意外行为和错误的风险。
