---
layout: post
title: C++17 Structured Bindings
---

Introduced under proposal [P0144R0](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0144r0.pdf), **Structured Bindings**
give us the ability to declare multiple variables initialised from a `tuple` or `struct`.

## Tuples

#### Pre C++17

When "unpacking" a tuple, for example, you had to first declare the variables, and then use `std::tie` to get the values back.

*Example:*

```cpp
#include <iostream>
#include <tuple>

int main()
{
    auto tuple = std::make_tuple(1, 'a', 2.3);

    // first we have to declare the variables
    int i;
    char c;
    double d;

    // now we can unpack the tuple into its individual components
    std::tie(i, c, d) = tuple;

    std::cout << "i=" << i << " c=" << c << " d=" << d << '\n';
    
    return 0;
}
```

*Build and run:*

    $ clang++ -std=c++14 main.cpp
    $ ./a.out
    i=1 c=a d=2.3

#### C++17

With the introduction of structured bindings, when unpacking our tuple we can declare the variables inline, at the call site, using the following syntax:

```cpp
auto [ var1, var2 ] = tuple;
```

*Example:*

```cpp
#include <iostream>
#include <tuple>

int main()
{
    auto tuple = std::make_tuple(1, 'a', 2.3);

    // unpack the tuple into individual variables declared at the call site
    auto [ i, c, d ] = tuple;

    std::cout << "i=" << i << " c=" << c << " d=" << d << '\n';

    return 0;
}
```

*Build and run:*

    $ clang++-4.0 -std=c++1z main.cpp
    $ ./a.out
    i=1 c=a d=2.3

*Note:*

I am using the llvm nightly `clang++-4.0` snapshot for the C++17 examples.

I added the [nightly package sources](http://apt.llvm.org/) to apt:

    $ grep llvm /etc/apt/sources.list
    deb http://apt.llvm.org/xenial/ llvm-toolchain-xenial main
    deb-src http://apt.llvm.org/xenial/ llvm-toolchain-xenial main

Then updated apt and installed

    $ sudo apt update
    $ sudo apt install clang-4.0

### Obtaining references to tuple members

#### Pre C++17

A major disadvantage to unpacking with `std::tie` is that it is impossible to obtain a reference to a tuple member.
This is because references can't be *reseated*. Once a reference is created, it cannot be later made to reference another object.

To illustrate, see here an attempt to create a reference and assign it to a member of the tuple using `std::tie`:

```cpp
#include <iostream>
#include <tuple>

int main()
{
    auto tuple = std::make_tuple(1, 'a', 2.3);

    // first we have to declare the variables
    int& i; // syntax error - can't declare a reference without initialising it
    char c;
    double d;
    std::tie(i, c, d) = tuple;

    std::cout << "i=" << i << " c=" << c << " d=" << d << '\n';

    // change the value of i inside the tuple
    i = 2;

    // show that the value inside the tuple has changed
    std::cout << "tuple<0>=" << std::get<0>(tuple) << '\n';

    return 0;
}
```

*Build:*

    $ clang++ -std=c++14 main.cpp
    main.cpp:8:10: error: declaration of reference variable 'i' requires an initializer
        int& i; // syntax error - can't declare a reference without initialising it
            ^
    1 error generated.

The only way in pre C++17 to obtain a reference to a tuple member was to explicitly use `std::get<index>(tuple)`:

```cpp
#include <iostream>
#include <tuple>

int main()
{
    auto tuple = std::make_tuple(1, 'a', 2.3);

    // unpack the tuple into its individual components
    auto& i = std::get<0>(tuple);
    auto& c = std::get<1>(tuple);
    auto& d = std::get<2>(tuple);

    std::cout << "i=" << i << ", c=" << c << ", d=" << d << '\n';

    // change the value of i inside the tuple
    i = 2;

    // show that the value inside the tuple has changed
    std::cout << "tuple<0>=" << std::get<0>(tuple) << '\n';

    return 0;
}
```
*Build and run:*

    $ clang++ -std=c++14 main.cpp
    $ ./a.out
    i=1, c=a, d=2.3
    tuple<0>=2

#### C++17

With the introduction of structured bindings, we can now obtain a reference to the tuple members using `auto&`:


```cpp
#include <iostream>
#include <tuple>

int main()
{
    auto tuple = std::make_tuple(1, 'a', 2.3);

    // unpack the tuple into its individual components
    auto& [ i, c, d ] = tuple;

    std::cout << "i=" << i << ", c=" << c << ", d=" << d << '\n';

    // change the value of i inside the tuple
    i = 2;

    // show that the value inside the tuple has changed
    std::cout << "tuple<0>=" << std::get<0>(tuple) << '\n';

    return 0;
}
```

*Build and run:*

    $ clang++-4.0 -std=c++1z main.cpp
    $ ./a.out
    i=1, c=a, d=2.3
    tuple<0>=2

## Structs

Structured bindings also allow us to "unpack" the members of a struct into its individual components.

```cpp
#include <iostream>
#include <tuple>

struct Foo
{
    int i;
    char c;
    double d;
};

int main()
{
    Foo f { 1, 'a', 2.3 };

    // unpack f's members into individual variables declared at the call site
    auto [ i, c, d ] = f;

    std::cout << "i=" << i << " c=" << c << " d=" << d << '\n';

    return 0;
}
```

    $ clang++-4.0 -std=c++1z main.cpp
    $ ./a.out
    i=1 c=a d=2.3

Similarly, we can obtain a reference to the members of a struct, and then change their value using our reference

```cpp
#include <iostream>
#include <tuple>

struct Foo
{
    int i;
    char c;
    double d;
};

int main()
{
    Foo f { 1, 'a', 2.3 };

    // unpack the struct into individual variables declared at the call site
    auto& [ i, c, d ] = f;

    std::cout << "i=" << i << " c=" << c << " d=" << d << '\n';

    // change the member c
    c = 'b';

    // show the value changed in f
    std::cout << "f.c=" << f.c << '\n';

    return 0;
}
```
    $ clang++-4.0 -std=c++1z main.cpp
    $ ./a.out
    i=1 c=a d=2.3
    f.c=b
