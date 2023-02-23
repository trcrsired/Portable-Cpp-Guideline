# Portable C++ Guideline

Are you interested in writing portable and efficient C++ code? Look no further than this guideline! Unlike other guidelines, such as the C++ Core Guideline, our guidelines prioritize portability and avoid problematic coding practices.

It's important to remember that there's no such thing as a zero-cost or zero-overhead (runtime) abstraction. Beware of anyone who claims otherwise! Even features like borrow checkers and C++ exceptions have runtime overhead (see here: https://blog.polybdenum.com/2021/08/09/when-zero-cost-abstractions-aren-t-zero-cost.html and here: https://devblogs.microsoft.com/oldnewthing/20220228-00/?p=106296).

Note that the current C++ standard is C++23.

## Freestanding

To write truly portable C++ code, it's important to stick to the freestanding C++ part and use platform macros to guard against the hosted environment. The cppreference website (https://en.cppreference.com/w/cpp/freestanding) provides a useful guide on freestanding implementations and the functions available in this environment.


However, it's important to note that many headers that you may assume should be available may not be present in a freestanding environment. Additionally, certain headers that are available may have limitations or issues that could prevent you from using them.


If you want to ensure that your code works in a freestanding environment, you can try using the x86_64-elf-gcc compiler. This compiler is available on the following GitHub repository: https://github.com/trcrsired/windows-hosted-x86_64-elf-toolchains

### Avoid ```std::addressof``` and ```operator &``` before C++23

This is because C++ allows ```operator &``` to be overloaded, which is a mistake from a historical perspective. To address this issue, the ISO C++ 11 standard introduced ```std::addressof```. However, in C++17, ```std::addressof``` was made constexpr. Unfortunately, ```std::addressof``` is part of the ```<memory>``` header, which is not freestanding. As a result, implementing ```std::addressof``` in C++ without compiler magic is impossible.

```cpp
//bad! memory header may not be available.
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

Update:

C++23 finally adds ```<memory>``` as partial freestanding header and ```std::addressof``` is now freestanding. However you need to ensure you use latest compilers that support C++23.

### Avoid ```std::move```, ```std::forward``` and ```std::move_if_noexcept``` Before C++23

The functions ```std::move```, ```std::forward```, and ```std::move_if_noexcept``` are defined in the <utility> header, which is not freestanding. To ensure maximum portability, it's recommended to write these functions yourself. However, it's worth noting that recent versions of the Clang compiler (version 15 onwards) have added a patch that treats these functions as compiler magics. As a result, it may not be possible to achieve 100% efficiency by writing these functions yourself.

```cpp
//bad! utility header may not be available.
//#include <utility> may also hurt compilation speed
#include<utility>

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

Update:

C++23 finally adds ```<utility>``` as freestanding header and ```std::move```, ```std::forward``` and ```std::move_if_noexcept``` are now freestanding. However, you need to ensure you use latest compilers that support C++23.

### Avoid ```std::array```

Similar to the reasons why you should avoid using ```std::addressof``` and ```std::move```, the ```<array>``` header is also not freestanding

```cpp
//bad! array header may not be available.
//#include <array> may also hurt compilation speed
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

The solution is to use C style array.

```cpp
//Good! You should use C style array because <array> does not exist.
//cstdint does exist.
#include<cstdint>

int main()
{
	::std::int_least32_t arr[4]{};//Ok! ::std::int_least32_t is freestanding
}

/*
x86_64-elf-g++ -c carray.cc -O3 -std=c++23 -s -flto
*/
```

Update:
In C++23, std::array is not considered freestanding. However, it is possible that it will become freestanding in future versions of the C++ standard.

### Consider Freestanding Alternatives to C++ Containers, Iterators, Algorithms, and Allocators

To achieve maximum portability, it is recommended to avoid all C++ containers, iterators, algorithms, and allocators. These components are not freestanding and have complex designs that interact with heap and exception handling, creating portability issues when included. Despite being promoted as the default container in "modern" C++ books and the C++ Core Guidelines, ```std::vector``` is not freestanding, and including it can cause portability issues. Additionally, allocators are not freestanding, and even ```std::allocator``` cannot be considered freestanding due to its interaction with exceptions. If exceptions are disabled, dependencies to exceptions remain, causing linkage errors or binary bloat issues.

```cpp
/// WRONG!!! <vector> is not freestanding
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

### Avoid ```<ranges>``` and ```<iterator>``` before C++23

You may want to use concepts like ```std::random_access_range```, ```std::contiguous_range```. Unfortunately, they are not freestanding either. A suggestion is that you should keep your design simple and stick to pointers as much as possible.

Update:
```<ranges>``` and ```<iterator>``` are partial freestanding since C++23. Everything can be used besides stream iterator things.

## Heap

Yes, I know ```<new>``` is freestanding. However, C++ standard does not design it to be truly useful. They throw exceptions very chaotically.

### Avoid ```std::nothrow```

Avoid using ```std::nothrow``` and ``std::nothrow_t`` in C++, as the standard library implements nothrow new by internally using thrown new. This means that ```std::nothrow``` provides no guarantee that memory allocation will not throw an exception. Additionally, the C++ standard does not allow overriding the default implementation of ```operator new(std::nothrow_t const&)```, unlike ```operator new()```. Therefore, there is no practical benefit to using ```std::nothrow```, and it should be avoided. Additionally, exceptions should be avoided in freestanding environments, so it is recommended to implement custom memory allocation functions that do not throw exceptions.

```cpp
/*
Do not use new (std::nothrow_t) in any form. It is just horrible.

Here is the implementation from libsupc++ in GCC.
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

We can see C++ standard library implements nothrow version's operator new with thrown version's new. It is completely useless.


### The default implementation of the heap may be unavailable.

In short, you cannot assume there is a default heap that is available.

Reasons:
1. There might not be a heap available at all. While you could argue that you could always provide one, the reality is that this is not always possible.
2. In some situations, multiple heaps might be available, such as within operating system kernels or even in Win32 applications with ABI compatibility issues between msvcrt and Universal CRT. In these cases, using any of the defaults might be the wrong choice. For example, the Windows kernel provides different heaps, one of which is interrupt-safe but has limited space, while the others are not interrupt-safe but have sufficient space.
3. C++ does not provide a thread-local implementation of the heap, and the default heap might be highly inefficient. Of course, thread-local storage might not be available, but that is a separate issue.

For a portable codebase, forcing a default heap implementation never works out, so it's best to avoid using new.

### Avoid handling stack or heap allocation failure.

It's best to avoid handling stack or heap allocation failure. One design flaw with C and C++ is that stack exhaustion is undefined behavior, while heap exhaustion is not. This creates inconsistency and can lead to a lot of issues.

#### There is no way that you can always handle heap allocation failure to avoid crashing.

In general, it's preferable to simply call std::abort if malloc(3) fails. Here are some reasons why:

1. Destructors can allocate memory again, creating a vicious cycle that can lead to unpredictable results.
2. The GCC libsupc++ implementation uses an emergency heap, but even if you implement an emergency heap, it might still call std::terminate in many situations, leading to crashes. To verify this, I removed the emergency heap from the libsupc++ and found no issues, including no ABI issues.
3. Many libraries beneath you, including Glibc, call xmalloc, which will still crash for malloc failure. You cannot avoid the issue unless you control all your source code.
4. Operating systems like Linux will overcommit and kill your process if you hit allocation failures. In general, programming languages like C++ cannot handle allocation failures in any meaningful way, whether on the stack or the heap.
5. C++'s new operator throws exceptions, and C++ codebases usually allocate memory on the heap using new, creating invisible code paths and potential exception-safety bugs.

#### If you want to load large files, consider using the memory mapping API instead of loading them into memory.

While you might argue that you want to load large files, such as images, to memory, this is usually not the best approach. Instead, you should use fstat(2) or Linux's statx system call and memory mapping.

##### Advantages
1. Memory mapping avoids the issue of file size overflow on 32-bit machines.
2. It avoids messing up the CRT heap and never triggers allocation failure from malloc or new.
3. It avoids copying the content from kernel space to user space, compared to loading the entire file to a std::string.
4. Memory mapping allows file sharing among different processes, saving your physical memory.
5. fseek(3) or seek(2) to load a file may create more TOCTOU security vulnerabilities.
6. The overcommit is less likely if you do not write to the copy-on-write pages.
7. Memory mapping creates a std::contiguous_range, which is extremely useful for many workflows.
8. You can write to memory-mapped memory without changing the file's content if you load the pages with private pages. However, writing content to the memory region will trigger a page fault, and the operating system kernel will allocate new pages for your process.

```cpp
/*
BAD. fseek(3) and fread(3) to load file
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
  // start to process your data / extract strings here...
}

```

Libraries like ```fast_io``` provide direct support for file loading and handling all the corner cases for you, including transparent support for platforms that do not offer memory mapping (like web assembly). The memory is writable.

https://github.com/cppfastio/fast_io

```cpp
/*
Good! native_file_loader with memory mapping
https://github.com/cppfastio/fast_io/blob/master/examples/0006.file_io/file_loader.cc
*/
#include<fast_io.h>

int main()
{
	fast_io::native_file_loader loader(u8"a.txt");
	//This will load entire a.txt to memory through memory mapping.
	/*
	This is a contiguous range of the file.
	You can do these things:
	std::size_t sum{};
	for(auto e:loader)
		sum+=e;	
	*/
}
```

## Exceptions

C++ exception is probably the largest issue for portability. Even Linus Torvalds complaint about C++ EH before.

http://harmful.cat-v.org/software/c++/linus

### Many platforms do not provide exceptions.
The issue with C++ exception handling goes beyond just portability. The reliance on operating system support and hosted C++ features makes it difficult to use in various contexts, including embedded systems and operating system development. This limitation can pose significant challenges for developers who want to write portable code.

The lack of support for exceptions on certain architectures, such as AVR and wasm, is another major challenge. While some tools, such as wasm2lua (https://github.com/SwadicalRag/wasm2lua), allow for the compilation of C++ code to other languages, implementing C++-style exception handling in these contexts can be a difficult and performance-intensive process.

Even on platforms that do support exceptions, the performance hit can be significant. The SJLJ exception, a common implementation for many platforms, can slow down both happy and slow paths. This can make exception handling impractical for performance-critical applications.

Additionally, not every architecture supports C++ exceptions, and even when they do, the implementation difficulty can be significant. The need for heap memory in exception handling, as well as the enormous runtime of C++ exceptions (typically around 100kb), can make it unacceptable for many embedded systems, where binary size and memory usage are critical factors.

Ultimately, the bloat in binary size caused by C++ exceptions is yet another challenge faced by developers in many contexts, including embedded systems. With all of these issues in mind, it is important for developers to carefully consider the use of C++ exception handling in their code and to explore alternative error-handling mechanisms where appropriate.

### C++ exceptions always slow down performance and are not "zero-overhead" abstractions.

1. Many platforms do not implement table-based exception handling models, which are required for C++ exception handling.
2. C++ exceptions can negatively impact compiler optimizations, regardless of whether the exception handling model is table-based or SJLJ-based.
3. C++ exceptions can hurt memory locality, including TLB, BTB, cache, and page locality, regardless of the exception handling model.
4. C++ exception handling does not work well with ASLR (Address Space Layout Randomization) and PIC (Position Independent Code), as loaders must relocate all exception table pointers, which can break ABIs. Recent security papers have discussed ways to make this work with PIC, but it may still break ABIs.
5. C header files often do not correctly mark their functions with the "noexcept" specifier, creating not only performance issues but also technically violating the One-Definition Rule (ODR). Calling C APIs that are not "noexcept"-marked results in undefined behavior, regardless of the exception handling model.
6. Throwing C++ exceptions can be extremely costly, potentially taking 200-300 times longer than a syscall instruction on x86_64.
7. C++ exceptions do not work well in multithreaded systems, which has become a significant problem as more and more software becomes multithreaded. The paper "C++ exceptions are becoming more and more problematic" (https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2544r0.html) provides more information on this issue.
8. Statistics show that 95% of exceptions are due to programming bugs, and these cases should be dealt with using assertions or even "std::terminate" rather than throwing exceptions, as this can result in improved exception-safety and performance.

C++ Exceptions are widely considered to be a significant pain point in the language, with few redeeming qualities. However, some members of the C++ community see hope in proposals like Herb Sutter's P0709R0. For more information, you can watch his video presentation here: https://www.youtube.com/watch?v=ARYP83yNAWk

## IO

Avoid using iostream and stdio as they come with many pitfalls.

### stdio and iostream are NOT freestanding.

Even toolchain vendors like LLVM libcxx cannot build without stdio, which makes bootstrapping difficult.

### stdio and iostream do not provide consistent input and output due to locale.

The behavior of stdio and iostream can change randomly with locale settings, making a program that works on one machine fail on another.

### stdio and iostream are not thread-safe.

Changes to locale settings at runtime can cause thread-safety issues with stdio and iostream. Additionally, iostream does not have any built-in thread awareness or locking mechanisms, leading to potential issues when multiple threads access the stream.

### Improper usage of the ```printf``` family of functions can lead to severe security vulnerabilities.

The ```printf``` family of functions are powerful tools for formatting and printing output to the console, but they can also be a source of serious security vulnerabilities if used improperly. One of the most common ways that printf functions can be misused is through format string attacks, which occur when an attacker is able to inject format specifiers into a printf call, causing the program to output data from the stack or other sensitive areas of memory.

To avoid these kinds of vulnerabilities, it's important to always use printf functions with caution. This means carefully validating and sanitizing any input that will be used in a printf call, and avoiding the use of format strings that can be controlled by an attacker. Additionally, it's important to always use the correct format specifiers for the data being printed, and to avoid using the %n specifier, which can be used to write to arbitrary memory locations.

Overall, while printf functions can be a powerful and useful tool, it's important to use them with care and attention to detail to avoid introducing security vulnerabilities into your code.

```cpp
inline void foo(std::string const& str)
{
//DANGER! format string vulnerability
	printf(str.c_str());
//DANGER ignore the return value of printf or scanf
}
```

### Avoid using ```std::endl``` in your C++ code.

One of the primary issues with using ```std::endl``` is the impact it can have on performance. Since ```std::endl``` flushes the output buffer every time you insert a new line character, it can be inefficient if you are printing a lot of output. This can slow down your program and make it less responsive, especially if you are working with large data sets or running your program on a slow machine.

Another problem with using ```std::endl``` is that it can be redundant. C++ output streams include a mechanism called "tie" that automatically flushes the output buffer when necessary, such as when the buffer is full or when the program terminates. This means that using std::endl to manually flush the buffer is often unnecessary and can even interfere with the automatic flushing mechanism.

To avoid these issues, it's generally better to rely on the automatic flushing mechanism built into C++ output streams and avoid using ```std::endl``` altogether. If you do need to flush the buffer at a specific point in your code, you can use the ```flush()``` method on the stream object itself. This method only flushes the buffer and does not insert a new line character, which can improve performance in cases where you do not need to print a new line.

If you want to print your output immediately without buffering, you can use unbuffered streams. However, the C++ stream library does not provide robust support for unbuffered streams, so you may need to use a third-party IO library like ```fast_io``` to achieve this.

### Never assume int8_t or int_least8_t or int_fast8_t to be either integers or characters.

```cpp
int8_t *p{};
std::cout<<p; // DANGER! iostream overloads treats p as char* and this will treat p as a string.
```

Unfortunately, random pitfalls like that are everywhere for iostream.

### Use memory mapping for loading large files. Not ```seek``` + ```read``` combos.

### Prefer ```fast_io``` over stdio and iostream

It is recommended to use the ```fast_io``` library over ```stdio``` and ```iostream``` for improved performance and additional features. Unlike ```stdio``` and ```iostream```, fast_io is not plagued with issues such as buffering and provides more comprehensive functionality.

Furthermore, ```fast_io``` offers a deep understanding and integration of ```stdio``` and ```iostream```, exposing more internal details that are not readily available with these standard libraries.

Perhaps the most significant advantage of using ```fast_io``` is the dramatic increase in speed. ```fast_io``` is generally 10-100 times faster than ```stdio``` and ```iostream``` due to the minimal redundant work that it performs in comparison. This enhanced performance can be particularly beneficial for applications that require a large amount of data processing or need to handle data in real-time.

### Do not assume ```write(2)``` or ```read(2)``` would do "real IO"

It is important to avoid assuming that ```write(2)``` or ```read(2)``` system calls will always perform "real IO". In many cases, the operating system kernel caches files in memory to allow for fast file reads and writes without blocking the process. As a result, ```write(2)``` and ```read(2)``` often function similarly to ```memcpy()``` and may not perform actual IO in the traditional sense. The data will eventually be flushed out to disk, but this process may not happen immediately.

As a consequence, IO operations typically do not take much time, with most of the time being spent on the overhead associated with the abstractions provided by iostream and stdio. It is therefore important to understand the underlying mechanisms involved in IO operations and to avoid making assumptions that can lead to inefficiencies or unexpected behavior.

### Avoid assuming consistent performance of stdio and iostream across various platforms.

It is important to avoid assuming that the performance of iostream and stdio is consistent across different platforms. Different operating systems and toolchains may provide different implementations of these libraries, resulting in significant performance gaps between platforms. As a result, it is important to test and measure the performance of IO operations on each platform to ensure optimal performance.

Moreover, while iostream and stdio are standard libraries and offer cross-platform compatibility, they may not be the fastest options for IO operations. As mentioned before, the ```fast_io``` library can often outperform these libraries by a significant margin due to its optimization for IO operations and reduced overhead. Therefore, it is worth considering alternative options such as ```fast_io``` for IO-intensive applications or performance-critical scenarios.

### All C++ standard library implementations (libstdc++, libcxx, and msvc stl) implement ```<iostream>``` using ```<stdio.h>```.

All implementations of ```<iostream>``` such as libstdc++, libcxx and msvc stl, use ```<stdio.h>``` in their implementation. This can be confirmed by examining the source code of each implementation. Due to this fact, it may not make sense to use iostream since it simply wraps around stdio.

[GCC libstdc++](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/std/fstream#L113)

[LLVM libcxx](https://github.com/llvm/llvm-project/blob/main/libcxx/include/fstream#L284)

[MSVC STL](https://github.com/microsoft/STL/blob/main/stl/inc/fstream#L781)

### Prefer ```stdio``` over ```iostream``` if you cannot use a third-party library.

Iostream can significantly increase binary size due to its object-oriented design, making it unsuitable for embedded systems. Toolchain vendors rarely optimize iostream for such systems, resulting in a typical implementation that can cost up to 1MB of binary size. Additionally, iostream needs to include all of stdio's code, making it even more bloated. In such cases, it is better to use stdio instead of iostream.

### Avoid using C++17's std::filesystem

```std::filesystem```is not thread-safe due to its locale-aware nature. Moreover, its design is outdated and does not meet the criteria of POSIX 2008. The lack of available ```at()``` methods for ```std::filesystem``` means you will always face potential [TOCTOU](https://en.wikipedia.org/wiki/Time-of-check_to_time-of-use) security vulnerabilities. The extensive use of locales also results in significant bloat, adding up to 1MB in some cases.

In addition, as pointed out in Herb Sutter's P0709, ```std::filesystem``` creates dual error reporting issues.

```cpp
//https://en.cppreference.com/w/cpp/filesystem/copy_file
bool copy_file( const std::filesystem::path& from,
                const std::filesystem::path& to );// Return bool but this api always returns true
bool copy_file( const std::filesystem::path& from,
                const std::filesystem::path& to,
                std::error_code& ec ); // NOTICE!!! NOT NOEXCEPT
```

Overall, C++17's std::filesystem is not a good API to use and should be avoided.

### Do not use ```<charconv>```

### Do not use ```<format>```

### Avoid any feature that uses locale internally, especially ```<cctype>``` header.

It's highly recommended to avoid using any feature that uses locale internally, especially the ```<cctype>``` header.

The ```<cctype>``` header is notoriously problematic. It's slow, not thread-safe, and can create undefined behavior at random. Some C++ books suggest using functions like ```isupper(3)``` to check whether a character is uppercase, but this is a terrible recommendation for several reasons:

1. The ```isupper(ch)``` function assumes ch is in the range [0,127], which means undefined behavior can be triggered randomly if checks aren't done beforehand.
2. The function is locale-aware, and locale is not thread-safe. This can lead to issues.
3. Due to its locale usage, the function does not always provide deterministic results.
4. The function is very slow, and in many implementations, even a trivial task like this would be a DLL indirect call. This can result in a significant performance hit, especially on platforms like Windows where it can cause a 100x performance downgrade.
5. The function is neither constexpr nor noexcept, making it a poor choice.
6. The function is not generic and only works for char. It does not work for char16_t, for example.
7. Additionally, the <cctype> header is not freestanding, which means it cannot be used in certain contexts.

In summary, it's best to avoid using the <cctype> header and other locale-dependent features to ensure better performance, thread safety, and determinism in your code.

```cpp
char ch{};
if(isupper(ch))//BAD
	puts("Do something\n");
```

```cpp
template<std::integral char_type>
inline constexpr bool myisupper(char_type ch) noexcept
{
	return u8'A'<=ch&&ch<=u8'Z';//simple and naive implementation
// Ok for ascii based execution charset.
// Need to do more work for ebcdic and big endian wchar_t.
// We ignore those cases here if you do not use them.
// They should not matter for 99.999999% of code.
}

char ch{};
if(myisupper(ch))//Mostly ok
	puts("Do something\n");
```

The ```fast_io``` library offers functions that work with wchar_t with big endian, or even with execution charsets like EBCDIC. Moreover, the library is freestanding, making it a great choice for use in a wide variety of environments.
	
```cpp
char ch{};
if(fast_io::char_category::is_c_upper(ch))//ok
	print("Do something\n");
```

## Integers

### Prefer ```::std::size_t``` over ```int``` as your default integer type.

When choosing integer types, it's generally recommended to use ```::std::size_t``` as the default type rather than int. While int is a commonly used type, the C++ standard does not define how large ```sizeof(int)``` is, which can lead to issues.

On the other hand, ```::std::size_t``` is defined by the C++ standard to be an unsigned integer type that is guaranteed to be able to represent the size of any object that can be allocated in memory. Using ```::std::size_t``` as the default type can help ensure portability and prevent unexpected errors due to integer overflow.

In summary, it's best to use ```::std::size_t``` as the default integer type in your code to ensure better portability and prevent potential issues arising from undefined behavior.
	

```cpp
//HORRIBLE!!! This is undefined-behavior if vec.size() is larger than INT_MAX.
for(int i{};i!=vec.size();++i)
{
}
```

Exceptionals: APIs that use int. ```int main()``` for example.

### Avoid assuming that ```char```, ```wchar_t```, ```char8_t```, ```char16_t```, and ```char32_t``` are solely character types, as they are actually integer types.

Avoid assuming that ```char```, ```wchar_t```, ```char8_t```, ```char16_t```, and ```char32_t``` are character types. While they may seem like character types, they are actually integer types, and making assumptions otherwise can lead to undefined behavior. For example, the way C++ iostream handles these types can cause issues if you assume they are purely character types.

```cpp
//libc might define int8_t as char
int8_t i{},*p{__builtin_addressof(i)};
std::cout<<p;//DANGER!!!
std::cout<<std::format("{}\n",p);//DANGER!!!
```
### Be cautious of the endianness of ```wchar_t```

When working with wchar_t, it's important to be aware of its endianness. It's possible for the endianness of wchar_t to differ from that of your native machine. For instance, wchar_t may use the UTF32BE execution encoding, while your machine uses little endian. This means that any operations you perform on wchar_t may require swapping its endianness first. Other character types, such as char16_t and char32_t, don't have this issue, as their endianness always matches that of your machine.

### Prefer integer types in ```<cstdint>``` than basic integer types

It's recommended to use integer types in ```<cstdint>``` instead of basic integer types like ```int```. The C++ standard doesn't specify the size of ```sizeof(T)``` for ```short```, ```int```, ```long```, and ```long long```, so relying on these types can lead to unexpected behavior.

### Prefer ```::std::(u)int_leastxx_t``` over ```::std::(u)intxx_t```

It's best to use ```::std::(u)int_leastxx_t``` rather than ```::std::(u)intxx_t```, as the latter types are optional and may not exist. They are used when a single byte on certain architectures isn't 8 bits, which can save money for embedded systems. For maximum portability, always use ```::std::(u)int_leastxx_t```. It's also recommended to use the ```INTXX_C()``` and ```UINTXX_C()``` macros to define constants for ```::std::(u)int_leastxx_t```, as this makes working with these types much easier. It's worth noting that despite their names, these macros are used to define constants for ```::std::(u)int_leastxx_t```, not ```::std::(u)intxx_t```.

### Avoid ```__uint128_t``` for GCC and clang

These types only exist for 64-bit targets, and even then, they don't generate efficient code for compilers. It's better to unpack ```__uint128_t``` into two ```::std::uint_least64_t```.

### Do not use ```::std::uintmax_t``` and ```::std::intmax_t```

The reason to avoid using ```::std::uintmax_t``` and ```::std::intmax_t``` is because they can cause issues with ABI (Application Binary Interface) stability in C and C++.

ABI stability refers to the ability of a library or program to maintain compatibility with other libraries or programs that use it, even when changes are made to the implementation details of that library or program.

In the case of ```::std::uintmax_t``` and ```::std::intmax_t```, using these types can cause ABI issues because their size can vary depending on the platform and compiler being used. This can lead to compatibility issues when trying to use a library or program that was compiled with a different size for these types.

To avoid these issues and ensure compatibility, it's best to use fixed-size integer types like ```::std::(u)int_leastxx_t``` instead.

### Floating Point Arithmetic

In portable code, it is best to avoid assuming the existence of floating point types. There are several reasons for this:

1. Floating point registers might not be saved by the kernel, which could result in undefined behavior if they are used.
2. Soft floating point implementations can be very slow on hardware that lacks floating point arithmetic support.
3. The use of ```<cmath>``` APIs can lead to issues with ```math_errhandling```.
4. If you need to treat floating point types as integers, use ```std::bit_cast``` instead of pointer tricks, which can violate the strict-aliasing rule.

The strict aliasing rule is a rule in the C and C++ programming languages that states that a pointer of one type cannot be dereferenced as a pointer of a different type. In other words, it is not allowed to access the same memory location through two different pointers with different types, except for a few specific cases defined by the standard.

The strict aliasing rule is important because it allows the compiler to make optimizations based on the assumption that pointers of different types do not refer to the same memory location. Violating the strict aliasing rule can result in undefined behavior, such as crashes, incorrect results, or other unexpected behavior.	

For more information on the strict-aliasing rule, see: https://gist.github.com/shafik/848ae25ee209f698763cffee272a58f8

```c
//BAD
float Q_rsqrt( float number )
{
	long i;
	float x2, y;
	const float threehalfs = 1.5F;

	x2 = number * 0.5F;
	y  = number;
	i  = * ( long * ) &y;                       // evil floating point bit level hacking
	i  = 0x5f3759df - ( i >> 1 );               // what the fuck? 
	y  = * ( float * ) &i;
	y  = y * ( threehalfs - ( x2 * y * y ) );   // 1st iteration
//	y  = y * ( threehalfs - ( x2 * y * y ) );   // 2nd iteration, this can be removed

	return y;
}
```

```cpp
//GOOD
#include <bit>
#include <limits>
#include <cstdint>

constexpr float Q_rsqrt(float number) noexcept
{
	static_assert(std::numeric_limits<float>::is_iec559); // (enable only on IEEE 754)

	float const y = std::bit_cast<float>(
        0x5f3759df - (std::bit_cast<std::uint32_t>(number) >> 1));
	return y * (1.5f - (number * 0.5f * y * y));
}
```

### Prefer GCC and clang's vector extension over ```<immintrin.h>``` when you need SIMD

When you need to use SIMD (Single Instruction Multiple Data) in your code, it's generally better to use GCC and Clang's vector extensions instead of the ```<immintrin.h>``` library.

The vector extensions are more fundamental and more flexible, and they work on a variety of platforms, including wasm. You can find more information about the vector extensions in the GCC documentation.

It's worth noting that some types of SIMD data, such as ```_m128```, are actually implemented using vector extensions on GCC and Clang. This means that using the ```<immintrin.h>``` library to work with these types may not provide any additional benefits and may add unnecessary complexity to your code.

In general, using the vector extensions provided by GCC and Clang is a simpler and more portable way to incorporate SIMD operations into your code.

## Todo: Threads

Do not use them unless you guard against the platform you are using exactly.

### Never use spin lock.

Spin locks are a synchronization primitive used in multithreaded programming to ensure that only one thread is accessing a shared resource at a time. When one thread acquires a spin lock, other threads that try to acquire the same lock will spin in a loop, waiting for the lock to be released.

However, spin locks can cause problems in certain contexts. For example, if a thread holding a spin lock is preempted by the kernel scheduler, other threads waiting for the lock will spin indefinitely, consuming CPU resources and potentially causing the system to become unresponsive.

This issue was famously highlighted by Linus Torvalds, the creator of the Linux operating system, in a rant from 2007. In his post, Torvalds criticized the use of spin locks in the kernel and argued that they should be replaced with other synchronization primitives that were less likely to cause scheduling problems.

While spin locks can be useful in certain situations, it's important to use them judiciously and be aware of their potential downsides. In particular, in contexts where preemptive scheduling is used, it's generally better to use other synchronization primitives, such as mutexes or semaphores, that are less likely to cause scheduling problems.

See Linus Torvalds' rant.

https://www.realworldtech.com/forum/?threadid=189711&curpostid=189723

## Todo: OOP

In short, do not use OOP. Use type-erasure instead. (Herb Sutter's Metaclasses might trivialize this a lot so that there would be no reasons for OOP in any circumanstances.)

## Windows specific

### Guard Windows specific code with ```#if (defined(_WIN32) && !defined(__WINE__)) || defined(__CYGWIN__)```

In general, you should think about things like CYGWIN/MSYS2 exist. wine-gcc will define _WIN32 macros for POSIX compliant hosted GCC compilers, which are incorrect on Linux, FreeBSD, etc. That is why we should also exclude ```__WINE__```

### Do not ```#include<windows.h>``` in public headers

Reasons:
1. Many libraries like ```boost``` include the header as ```#include<Windows.h>```, which causes issues for cross compilations since UNIX filesystems are ```case-sensitive```. The GCC and clang compiler will not find that header.
2. Windows APIs are not correctly marked as noexcept.
3. It compiles very slow.
4. They introduce macros like ```min``` and ```max``` which will break C++ standard libraries.

```cpp
//BAD:
#pragma once
#include<Windows.h>
```
If you want to use win32 or nt APIs, import them by yourself. ```NEVER USE extern "C" to import APIs.```

```cpp
#pragma once
//Wrong! Here is the wrong way to import APIs.
namespace mynamespace::nt
{
struct io_status_block
{
union
{
	std::uint_least32_t Status;
	void*    Pointer;
} DUMMYUNIONNAME;
std::uintptr_t Information;
};

using pio_apc_routine = void (*)(void*,io_status_block*,std::uint_least32_t) noexcept;

#if defined(_MSC_VER) && !defined(__clang__)
__declspec(dllimport)
#elif __has_cpp_attribute(__gnu__::__dllimport__)
[[__gnu__::__dllimport__]]
#endif
extern "C" std::uint_least32_t __stdcall ZwWriteFile(void*,void*,pio_apc_routine,void*,io_status_block*,
				void const*,std::uint_least32_t,std::int_least64_t*,std::uint_least32_t*) noexcept;

}

#include<windows.h>
//BOOM!! Compiler complains.
```

```cpp
//GOOD:
#pragma once
namespace mynamespace::nt
{
struct io_status_block
{
union
{
	std::uint_least32_t Status;
	void*    Pointer;
} DUMMYUNIONNAME;
std::uintptr_t Information;
};

using pio_apc_routine = void (*)(void*,io_status_block*,std::uint_least32_t) noexcept;

#if defined(_MSC_VER) && !defined(__clang__)
__declspec(dllimport)
#elif __has_cpp_attribute(__gnu__::__dllimport__)
[[__gnu__::__dllimport__]]
#endif
extern std::uint_least32_t __stdcall ZwWriteFile(void*,void*,pio_apc_routine,void*,io_status_block*,
				void const*,std::uint_least32_t,std::int_least64_t*,std::uint_least32_t*) noexcept
#if defined(__clang__) || defined(__GNUC__)
#if SIZE_MAX<=UINT_LEAST32_MAX &&(defined(__x86__) || defined(_M_IX86) || defined(__i386__))
#if !defined(__clang__)
__asm__("ZwWriteFile@36")
#else
__asm__("_ZwWriteFile@36")
#endif
#else
__asm__("ZwWriteFile")
#endif
#endif
;

}

/*
Ignore the part that deals with name mangling for Visual C++.
See here:
https://github.com/trcrsired/fast_io/blob/master/include/fast_io_hosted/platforms/win32/msvc_linker.h
*/
```

### Call ```A``` APIS for Windows 9x kernels and ```W``` APIS for Windows NT kernels.

For Windows programming, it's important to know which API to call depending on the version of the operating system. In general, it's recommended to use the W APIs on modern versions of Windows, such as Windows NT-based systems, including Windows 10.

However, on older versions of Windows, such as Windows 95/98 and ME, only the A APIs are available. The W APIs do exist, but they don't do anything on these systems.

The problem with using the A APIs on modern versions of Windows is that they are influenced by the Windows Locale, which can cause issues. In contrast, all NT APIs are unicode APIs, so using the W APIs is generally safer and more portable.

To avoid issues with the wchar_t type and execution charset, it's recommended to import the APIs using C++'s char8_t and char16_t types instead. This can help ensure that your code is more portable and less likely to encounter issues related to character encoding and localization.

```cpp
//Use if constexpr to trivialize the API calls.

namespace mynamspace
{

enum class win32_family
{
ansi_9x,
wide_nt,
#ifdef _WIN32_WINDOWS
native = ansi_9x
#else
native = wide_nt
#endif
};

template<typename... Args>
inline void* my_create_file(Args... args) noexcept
{
	if constexpr(win32_family::native==win32_family::ansi_9x)
	{
		//Call A apis for Windows 95
		return ::mynamespace::win32::CreateFileA(args...);
	}
	else
	{
		//Call W apis for Windows NT
		return ::mynamespace::win32::CreateFileW(args...);
	}
}

}

```

This rule applies to CRT APIs, including fopen(3) _fdopen(3). You should _wfdopen for windows NT (Yes, this includes Windows 10). Better use them with win32 or NT APIs.

### Windows does provide file descriptors, and file descriptors on Windows are not win32 HANDLE

I have heard many people say windows do not provide file descriptors, which is untrue. The windows CRT implements file descriptors with win32 HANDLE.

See the API: ```_open_osfhandle```
https://docs.microsoft.com/en-us/cpp/c-runtime-library/reference/open-osfhandle?view=msvc-170

### Do not use ```std::unique_ptr``` for win32 ```HANDLE```

People love to abuse C++ smart pointers, nowadays.
```cpp
/*BAD!!!
Watch the video:
https://youtu.be/5vGWM9DLrko?t=1205
*/

std::unique_ptr<HANDLE,std::function<decltype(CloseHandle)>> pHandle(hEvent,CloseHandle);

```

Just do not be lazy. Please. Write an RAII class that wraps void*.

```cpp
//Good! Just write a class
win32_file file(u8"a.txt");
```

### Do not assume ```<pthread.h>``` exist for cpu-windows-gnu GCC and clang targets.

GCC for windows targets have three different threading ABIs. win32, posix and mcf. Only posix would provide ```<pthread.h>```. Users may use win32 or mcf. Your compilation will break. They may break C++ standard library too and libstdc++ does not provide ```<thread>``` and ```<mutex>``` headers for win32. ```LLVM libc++``` makes this mistake and that is why libc++ is not available for windows targets.


```cpp
//BAD!!
#include<pthread.h>
```
```cpp
//BAD!!
#include<thread>
```
```cpp
//BAD!!
#include<mutex>
```

```c
//BAD!!
#include<threads.h>
```

### Others

1. For cpu-windows-gnu clang triple targets, it's best to avoid using thread_local and _Thread since clang does not correctly implement GCC's ABI for Windows.

2. Be aware of the ABI differences between win32, posix, and mcf of GCC libstdc++-6.dll. Linking to the wrong C++ standard library runtime can cause iostream to break.

3. Testing and benchmarking your Windows applications on WINE, a compatibility layer that runs Windows apps on Linux, can help avoid issues like ABIs. To include C++ standard library DLLs like libstdc++-6.dll, remember to export the WINEPATH environment in $HOME/.bashrc.

## OTHER ISSUES

### The Purpose of Inline in C++: Preventing ODR Violations, Not for Hints or Expansions.

The purpose of the inline keyword in C++ is to prevent ODR (One Definition Rule) violations, not for providing hints or expansions. However, the keyword's usage is often confusing due to its different meanings compared to C. If a function with the same signature is defined in multiple translation units, the linker will fail to link them. When a function is marked as inline, the linker can discard any functions with the same signature and keep only one copy, assuming they work the same. However, if they don't work the same, it results in undefined behavior. Additionally, marking a function as inline allows GCC and clang to avoid emitting the function unless it's used, which is important for preventing dead code. For a better understanding of inline, watch the video at https://www.youtube.com/watch?v=6WjKIStrc80 and refer to this article https://devblogs.microsoft.com/oldnewthing/20200521-00/?p=103777.

### To ensure maximum portability, it is recommended to build and test your code on as many GCC cross/canadian toolchains as possible.

This will help you identify potential issues early on and allow you to develop a more robust and portable codebase. By testing on various platforms, you can ensure that your code behaves consistently across different systems and architectures, thus minimizing the risk of unexpected behavior and errors.


### Do not use ```std::unique_ptr```
