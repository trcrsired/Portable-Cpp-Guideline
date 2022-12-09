# Portable C++ Guideline

Have you ever tried to make your C++ code maximumly portable and efficient? Here is a guideline that helps your write portable and efficient C++ code.

You may ask why I created this guideline? The reason is that guidelines like C++ Core Guideline usually teach unportable and problematic coding practices that only promote the portability trap.

Remember, there is no zero-cost / zero-overhead (Runtime) abstraction. Anyone who tries to sell you those concepts is just falsely advertising. All abstractions have runtime overhead, including Borrow Checkers ( https://blog.polybdenum.com/2021/08/09/when-zero-cost-abstractions-aren-t-zero-cost.html ) and C++ exceptions ( https://devblogs.microsoft.com/oldnewthing/20220228-00/?p=106296 ).

Current C++ standard is C++23.

## Freestanding

In general, Portable C++ code should stick to the freestanding C++ part and use platform macros to guard against the hosted environment.

https://en.cppreference.com/w/cpp/freestanding

Unfortunately, many headers you may think should be available are not available, and those headers that are available may have issues preventing you from using them.

You can have a try on this x86_64-elf-gcc compiler to ensure your code work in freestanding environment.

https://github.com/trcrsired/windows-hosted-x86_64-elf-toolchains

### Avoid ```std::addressof``` and ```operator &``` before C++23

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

Update:

C++23 finally adds ```<utility>``` as freestanding header and ```std::move```, ```std::forward``` and ```std::move_if_noexcept``` are now freestanding. However, you need to ensure you use latest compilers that support C++23.

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

Update:
std::array is NOT freestanding in C++23. It may be freestanding in future C++ standards.

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

### Avoid ```<ranges>``` and ```<iterator>``` before C++23

You may want to use concepts like ```std::random_access_range```, ```std::contiguous_range```. Unfortunately, they are not freestanding either. A suggestion is that you should keep your design simple and stick to pointers as much as possible.

Update:
```<ranges>``` and ```<iterator>``` are partial freestanding since C++23. Everything can be used besides stream iterator things.

## Heap

Yes, I know ```<new>``` is freestanding. However, C++ standard does not design it to be truly useful. They throw exceptions very chaotically.

### Avoid ```std::nothrow```

NEVER use ```std::nothrow``` and ```std::nothrow_t```. C++ standard library implements nothrow new with thrown new. Even worse, the C++ standard does not allow override the default impementations for operator new(std::nothrow_t const&) as operator new() does. There is no reason to use it.


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
1. There is simply no heap. (I know you would argue you can always provide one, but the reality is harsh, my friend.)
2. Many heaps are available. This situation usually happens within operating system kernels (and even win32 applications with the abi compatibility issue between msvcrt and universal crt) where they provide multiple heaps. Making any of the defaults may be very wrong. For example, the Windows kernel provides different heaps; one is interrupt-safe while space is limited, but the others are not but space is sufficient.
3. C++ does not provide a thread-local implementation of the heap. The default heap might be highly inefficient. (Of course, thread-local storage might not be available, but that is a separate issue.)

We can see forcing a default heap implementation never works out for a portable codebase. Just avoid new.

### Avoid handling stack or heap allocation failure.

One design failure with C and C++ is that stack exhaustion is undefined behavior while heap exhaustion is not. That creates inconsistency and a lot of issues.

#### There is no way that you can always handle heap allocation failure to avoid crashing.

In general, you should prefer just call ```std::abort``` if ```malloc(3)``` fails.

1. Destructors can allocate memory again. That creates a vicious cycle and nobody knows what could happen.
2. The implementation of GCC libsupc++ uses an emergency heap. However, implementing an emergency would still call ```std::terminate``` in many situations. So you can still get crashing even if it has an "emergency heap." I made the operator a new crash for allocation failure, removed the emergency heap from the libsupc++, and found no issues. No ABI issues. Nothing.
3. Many libraries beneath you, including Glibc, would call xmalloc, which will still crash for malloc failure. You cannot avoid the issue unless you control all your source code.
4. Operating Systems like Linux would overcommit and kill your process if you hit allocation failures. In general, programming languages like C++ cannot just handle allocation failures in any meaningful way, neither stack nor heap.
5. C++ new throws exceptions. C++ codebase will usually allocate memory on the heap with new, creating tons of invisible code-path and potential exception-safety bugs.

#### If you want to load large files, consider memory mapping API.

I know you will try to make an argument that you want to load large files like image to memory for example. I am sorry, you are usually doing the wrong thing. Instead, you should use fstat(2) or Linux's statx syscall + memory mapping.

##### Advantages
1. Memory mapping avoids the issue of file size overflow on 32 bits machine.
2. It avoids messing up the CRT heap and never triggers allocation failure from malloc or new.
3. It avoids copying the content from kernel space to user space, compared to loading the entire file to a ```std::string```
4. Memory mapping allows file sharing among different processes, saving your physical memory.
5. fseek(3) or seek(2) to load file may create more TOCTOU security vulnerabilities.
6. The overcommit is less likely if you do not write to the copy-on-write pages.
7. Memory mapping creates ```std::contiguous_range``` which is extremely useful for many workflows.
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
1. C++ exception handling ABI relies on hosted C++ features, and many need operating system support. However, you are writing an operating system; for example, you do not have exceptions.
2. Architectures like AVR, and wasm cannot natively throw exceptions. wasm32-wasi simply has no exception handling support. (EMSCRIPTEN IS NOT wasm32-wasi) Furthermore, tools like wasm2lua (https://github.com/SwadicalRag/wasm2lua) allow you to compile C++ code to Lua. However, Lua can't implement C++-style exception handling without tons of human effort and severe performance hits.
3. C++ exceptions on many platforms are extremely slow for both happy and slow paths. SJLJ exception is a typical implementation for many platforms, including 32 bits operating systems and scripting. The performance hit is enormous.
4. Not every architecture support C++ exceptions, and the implementation difficulty may be very significant.
5. C++ exceptions need heap, which we mentioned as another huge issue.
6. C++ exception runtime is enormous. It is usually 100kb, which may be unacceptable for many embedded systems.
7. C++ exceptions bloat binary size with your code size. That may be another unacceptable factor for many embedded systems.

### C++ exceptions always slow down performance. They are not "zero-overhead" abstractions.

1. Many platforms do not implement table-based exception handling models.
2. Exceptions always hurt compiler optimizations. (No matter whether it is table-based or sjlj-based.)
3. Exceptions always hurt TLB, BTB, cache and pages locality. (No matter whether it is table-based or sjlj-based.)
4. C++ Exception handling does not work very well with ASLR (Address space layout randomization) and PIC (position independent code). This is because the loaders must relocate all exception table pointers. Recently, security papers have talked about how to make that work with the PIC but will break ABIs. (No matter whether it is table-based or sjlj-based.)
5. C header files usually do not mark their functions correctly with ```noexcept```. That not only creates the issue for performance but they are technically ODR violations. Calling C apis that are not noexcept-marked are just ```undefined-behaviors```. (No matter whether it is table-based or sjlj-based.)
6. Throwing C++ exceptions is extremely costly. It can be 200x and 300x slower than a syscall instruction on ```x86_64```.
7. C++ exceptions do not work well in multithreadings systems. See ```C++ exceptions are becoming more and more problematic``` ( https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2544r0.html ).
8. Statistics shows 95% of exceptions are just programming bugs. They just create issues for exception-safety and performance issues. Those cases should use assertion or even ```std::terminate``` to deal with instead of throwing exceptions.

### In general, C++ Exceptions are just horrible. Herb Sutter's P0709R0 is our only hope.
Watch video: https://www.youtube.com/watch?v=ARYP83yNAWk

## IO

In short. Do not use iostream and stdio. If you have to use iostream and stdio, read this section to understand the pitfalls.

### stdio and iostream are NOT freestanding.

That is an issue even with toolchain vendors. LLVM libcxx cannot build without stdio, which breaks bootstrapping. Just do not use them.

### stdio and iostream do not provide the same input and output due to locale.

Locale can randomly change the behavior of stdio and iostream. That means a program that would work on one machine may not work on another machine.

### stdio and iostream are not thread-safe.

Locale affects stdio and iostream. Users might change locale at runtime, and that would cause thread-safety issues.

iostream does not have any thread awareness. It does not lock the stream. Different threads would screw up any C++ stream randomly.

### Using ```printf``` family functions incorrectly are always severe security vulnerabilities.

Usage of ```printf``` functions must be cautious.

```cpp
inline void foo(std::string const& str)
{
//DANGER! format string vulnerability
	printf(str.c_str());
//DANGER ignore the return value of printf or scanf
}
```

### Never use ```std::endl```

It is completely redundant. The tie mechanism of stream knows when to flush when deal with input and does the right thing automatically.

Even you just want to flush the stream, you still have tons of other options. One is to manually flush the buffered stream. If you want to flush out every time, you probably need unbuffered stream. Unfortunately C++ stream does not provide enough support with unbuffered stream and that is why you need ```fast_io```

### Never assume int8_t or int_least8_t or int_fast8_t to be either integers or characters.

```cpp
int8_t *p{};
std::cout<<p; // DANGER! iostream overloads treats p as char* and this will treat p as a string.
```

Unfortunately, random pitfalls like that are everywhere for iostream.

### Use memory mapping for loading large files. Not ```seek``` + ```read``` combos.

### Prefer ```fast_io``` over stdio and iostream

```fast_io``` library does not have all the issues we discussed before and provides tons of features stdio and iostream do not provide. However, it also offers deep understanding and hacking of stdio and iostream so that fast_io would expose more internal details for stdio and iostream.

More importantly, ```fast_io``` is usually 10x or even 100x faster than stdio and iostream due to little redundant work to do compared to stdio and iostream.

### Do not assume ```write(2)``` or ```read(2)``` would do "real IO"

Most of the time, the operating system kernel would cache files into memory so that users would read/write files very fast instead of blocking the process. That means syscalls like ```write(2)``` or ```read(2)``` do the job something like memcpy for most of their time. The data will flush out to disk regularly. That is why most of time, io does not take too much time but most of time would waste on overhead for the abstractions of iostream and stdio.

### Do not assume stdio and iostream have the same relative performance on different platforms.

Different operating system and toolchains would provide different implementations of iostream and stdio. The performance gap between different platforms are huge. However, none of them could match the performance of fast_io in any way.

### ```<iostream>``` is implemented with ```<stdio.h>``` for all implementations. (libstdc++, libcxx and msvc stl)

You can check that by the reading source of all of them. That is another why you should not use ```iostream``` since why would you use it when it just wraps ```stdio```?

GCC libstdc++:

[https://github.com/llvm/llvm-project/blob/main/libcxx/include/fstream#L284](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/std/fstream#L113)

LLVM libcxx:
[https://github.com/llvm/llvm-project/blob/main/libcxx/include/fstream#L284](https://github.com/llvm/llvm-project/blob/main/libcxx/include/fstream#L284)

MSVC STL:
https://github.com/microsoft/STL/blob/main/stl/inc/fstream#L781

### Prefer stdio over iostream if you cannot use a third-party library.

iostream bloats binary size due to its object-oriented design. Usually, toolchain vendors need to optimize iostream for embedded systems. A typical iostream implementation costs 1MB of binary size. Also, ```iostream``` needs to include all ```stdio``` code, so please just no iostream.

### Do not use C++17 ```std::filesystem```

```std::filesystem``` is not thread-safe since it is locale-aware. More importantly, the design of ```std::filesystem``` is very outdated. It does not even match the criteria of POSIX 2008. No ```at()``` methods are available for ```std::filesystem``` and you always suffer from [TOCTOU](https://en.wikipedia.org/wiki/Time-of-check_to_time-of-use) security vulnerabilities.

Also, it is very bloat due its locale usage. It would usually bloat 1MB.

As Herb Sutter pointed out in P0709, ```std::filesystem``` creates dual error reporting issues.

```cpp
//https://en.cppreference.com/w/cpp/filesystem/copy_file
bool copy_file( const std::filesystem::path& from,
                const std::filesystem::path& to );// Return bool but this api always returns true
bool copy_file( const std::filesystem::path& from,
                const std::filesystem::path& to,
                std::error_code& ec ); // NOTICE!!! NOT NOEXCEPT
```

C++17 filesystem are just bad apis. Never use it.

### Avoid any feature that uses locale internally. Particularly ```<cctype>```

```<cctype>``` is a very horrible header. It is slow and not thread-safe. It also creates undefined behavior randomly. Most C++ books would recommend you for using things like ```isupper(3)``` to detect whether a character is uppercase or not. This is a horrible recommendation.

1. For isupper(ch), the function assumes ch is in the range [0,127]. That means you can randomly trigger undefined behavior if you do not do the checks before.
2. the function is locale-aware. Locale is not thread-safe. You will screw it up.
3. Due to its locale usage, it does not answer deterministically.
4. It is very slow. Most of the implementation would make this call a DLL indirect call, even if you are doing a trivial task like this. I have seen on platforms like windows that using it would cause a performance downgrade of 100x.
5. It is not constexpr nor noexcept. This API is BAD.
6. It is not generic and it only works for char. It does not work for char16_t for example.
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
//Ok for ascii based execution charset. Need to do more work for ebcdic and big endian wchar_t. We ignore those cases here if you do not use them. They should not matter for 99.999999% of code.
}

char ch{};
if(myisupper(ch))//Mostly ok
	puts("Do something\n");
```

```fast_io``` library provides them and they work even for wchar_t BE and EBCDIC execution charset
```cpp
char ch{};
if(fast_io::char_category::is_c_upper(ch))//ok
	print("Do something\n");
```



## Todo: Integers

### ```::std::size_t``` should be your default integer types. Not ```int```.

C++ standard never said how large ```sizeof(int)``` is. They have caused a lot of troubles.

```cpp
//HORRIBLE!!! This is undefined-behavior if vec.size() is larger than INT_MAX.
for(int i{};i!=vec.size();++i)
{
}
```

Exceptionals: APIs that use int. ```int main()``` for example.

### Do not assume ```char```, ```wchar_t```, ```char8_t```, ```char16_t``` and ```char32_t``` are character types. They are just integer types.

That assumptions randomly create undefined behavior. One notorious example is C++ iostream.

```cpp
//libc might define int8_t as char
int8_t i{},*p{__builtin_addressof(i)};
std::cout<<p;//DANGER!!!
std::cout<<std::format("{}\n",p);//DANGER!!!
```
### Be cautious of the endianness of ```wchar_t```

It might be surprising that the endianness of ```wchar_t``` may not be the same as the endianness of your native endian.

For example, the execution encoding of ```wchar_t``` may be UTF32BE, but your machine's endianness is little. That means any options you do for ```wchar_t``` need to swap its endianness before doing anything.

Other character types, including ```char16_t``` and ```char32_t``` do not have this issue. Their endianness are always the same as your machine's endiannness.

### Prefer integer types in ```<cstdint>``` than basic integer types

They have the same issue with int. C++ standard never said how large sizeof(T) for short, int, long and long long.


### Prefer ```::std::uint_leastxx_t``` over ```::std::uintxx_t```

Types like ::std::uintxx_t are optional and may not exist. They happen when a single byte in some architectures is not 8 bits which might save a lot of money for specific embedded systems. For maximumly portable code, just use ```::std::uint_leastxx_t```

### Avoid __uint128_t for GCC and clang

Reasons:
1. They only exist for 64 bits targets.
2. Even 64 bits targets do not generate good code for compilers. Better way is to unpack ```__uint128_t``` to two ```::std::uint_least64_t```.

### Do not use ```::std::uintmax_t``` and ```::std::intmax_t```

They have caused a lot of troubles for ABI stablities issues for C and C++. Just do not use them.

## Floating Points

Short answer: Do not assume floating points exist!!! Just do not use them in portable code.

### Floating points might use extra registers which kernel is not willing to save.

Kernels usually just ban the usage for floating points.

### Soft floating points are very slow on hardwares that could not provide floating points arithmatics

### ```<cmath>``` APIs touch ```math_errhandling``` which cause a lot troubles

### Use std::bit_cast for dealing with floating point types if you want treat them as integers.

Pointer tricks would easily violate strict-aliasing rule. See
https://gist.github.com/shafik/848ae25ee209f698763cffee272a58f8

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

Vector extensions are more fundamental and more flexible. They work on a lot of platforms, including wasm.

https://gcc.gnu.org/onlinedocs/gcc/Vector-Extensions.html

Things like ```_m128``` are implemented with vector extensions on GCC and clang. As a result, there is no point to use them.

## Todo: Threads

Do not use them unless you guard against the platform you are using exactly.

### Never use spin lock.

Spin lock screws up kernel scheduling. See Linus Torvalds' rant.

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

On Windows 95/98 and ME, only A Apis are available. W apis do exist but they do not do anything.

On Windows NT-based systems, including Windows 10, you should use W APIs and avoid A APIs. The problem is that Windows Locale will influence the behavior of A APIs and cause issues. All NT apis are unicode APIs.

You can import those APIs with C++ ```char8_t``` and ```char16_t```. That will get rid of the troubles for ```wchar_t``` and execution charset.

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

### Avoid ```thread_local``` and ```_Thread``` for cpu-windows-gnu clang targets

clang does not implement GCC's ABI correct for windows. Just avoid ```thread_local``` and ```_Thread```.

### Beware ABI for win32, posix and mcf of GCC libstdc++-6.dll

They are three different ABIs, and iostream will break if you link to the wrong GCC C++ standard library runtime (libstdc++-6.dll).

### It is a good idea to run, test and benchmark your Windows Applications on WINE.

Running Windows applications on Linux with WINE has substantial benefits in avoiding issues like ABIs.

Remember to export the WINEPATH environment in $HOME/.bashrc to include C++ standard libraries DLLs like libstdc++-6.dll.

## Todo: OTHER ISSUES

### inline means to prevent ODR violation. Not for hints or expansions.

The keyword is very confusing in C++. It also has different meanings than C.

If you define a function with the same signature in multiple translation units, the linker will complain and won't link. When a function is marked as inline, the linker is free to discard any functions with the same signature and keeps only one copy, assuming these functions work the same. If they do not work the same, that is undefined behavior.

Also, when the function is defined as inline, GCC and clang won't emit functions unless used, which is very important for preventing dead code.

Watch video if you want to understand ```inline```.
https://www.youtube.com/watch?v=6WjKIStrc80
See also:

https://devblogs.microsoft.com/oldnewthing/20200521-00/?p=103777
