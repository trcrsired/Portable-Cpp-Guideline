# Portable-C++-Guideline

Have you ever tried to make your C++ code maximumly portable and efficient? Here is a guideline that helps your write portable and efficient C++ code.

You may ask why I created this guideline? The reason is that guidelines like C++ Core Guideline usually teach unportable and problematic coding practices that only promote the portability trap.

Remember, there is no zero-cost / zero-overhead (Runtime) abstraction. Anyone who tries to sell you those concepts is just falsely advertising. All abstractions have runtime overhead, including Borrow Checkers and C++ exceptions.

Current C++ standard is C++23.

## Freestanding

In general, C++ code should stick to the freestanding C++ part and use platform macros to guard against the hosted environment.

https://en.cppreference.com/w/cpp/freestanding

Unfortunately, many headers you may think should be available are not available, and those headers that are available may have issues preventing you from using them.

You can have a try on this x86_64-elf-gcc compiler to ensure your code work in freestanding environment.

https://bitbucket.org/ejsvifq_mabmip/windows-hosted-x86_64-elf-toolchains

### Avoid ```std::addressof``` and ```operator &```

C++ allows overloading operator &, which is a historical mistake. ISO C++ 11 standard introduces std::addressof that would fix the issue. C++17 allows std::addressof to be constexpr Unfortunately, ```std::addressof``` is in ```<memory>``` header, which is not freestanding. The std::addressof is impossible to implement in C++ without compiler magic.

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
}

/*
D:\Desktop>x86_64-elf-g++ -c builtin_addressof.cc -O3 -std=c++23 -s -flto

Compilation success
*/
```

### Avoid ```std::move```, ```std::forward``` and ```std::move_if_noexcept```

```std::move```, ```std::forward``` and ```std::move_if_noexcept``` are in ```<utility>``` header, which are not freestanding.

The safest thing is to write them by yourself. Unfortunately, recently clang 15 added a patch that would treat ```std::move```, ```std::forward```, ```std::addressof```, ```std::__addressof``` and ```std::move_if_noexcept``` as compiler magics which means we cannot get 100% efficiency by writing by ourselves.

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

### Avoid ```std::array```

The same issue as why you should avoid ```std::addressof```, ```std::move```, and etc. ```<array>``` header is not freestanding.


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


### Avoid all C++ containers, iterators, algorithms, allocators

None of the C++ Containers, Iterators, Algorithms, and Allocators are freestanding, and they won't be freestanding soon due to deeply complex designs interacting with heap and exception handling. (We will explain why in the next few chapters.)

By C++ Core Guideline or "modern" C++ books, ```std::vector``` should be the default container. However,        ```<vector>``` is not freestanding. Including them creates a portability issue. Allocators are not freestanding either, and even ```std::allocator``` is freestanding (which it cannot due to interacting with exceptions), the vector itself will throw ```std::logic_error``` in methods like ```push_back()```. Disabling Exceptions handling will not remove the dependencies to exceptions which still cause either linkage errors or binary bloat issues.

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
