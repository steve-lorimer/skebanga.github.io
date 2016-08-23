---
layout: post
title: C++17 If statement with initializer
---

Introduced under proposal [P00305r0](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0305r0.html), **If statement with initializer**
give us the ability to initialize a variable within an `if` statement, and then, once initialized, perform the actual conditional check.

## If statements

#### Pre C++17

When inserting into a map, if you want to check whether the element was inserted into the map or not, you need to perform the insertion,
capture the return value, and then check for successful insertion

*Example:*

```cpp
#include <iostream>
#include <map>

int main()
{
    std::map<std::string, int> map;
    map["hello"] = 1;
    map["world"] = 2;

    // first we have to perform the operation and capture the return value
    auto ret = map.insert({ "hello", 3 });

    // only then can we check the return value
    if (!ret.second)
        std::cout << "hello already exists with value " << ret.first->second << "\n";

    // ret has "leaked" into this scope, so we need to pick a different name, ret2
    auto ret2 = map.insert({ "foo", 4 });

    // now we can check the value of ret2
    if (!ret2.second)
        std::cout << "foo already exists with value " << ret2.first->second << "\n";

    return 0;
}
```

*Build and run:*

    $ clang++ -std=c++14 main.cpp
    $ ./a.out
    hello already exists with value 1

There are 2 annoyances with this code (albeit fairly minor).

1. It is more verbose; we have to create the `ret` variable which we want to check for successful insertion, and then, only once the
variable has been created, can we then perform the actual check.

2. Slightly more insidious, the `ret` variable "leaks" into the surrounding scope. Note for the 2nd insertion we have to call the
result variable `ret2`, because `ret` already exists.

It is quite easy to fix the 2nd annoyance by putting each conditional check into its own scope. This, however, comes at the cost of making
the code even more verbose, and results in these floating braces with no preceding statement.

```cpp
#include <iostream>
#include <map>

int main()
{
    std::map<std::string, int> map;
    map["hello"] = 1;
    map["world"] = 2;

    // we create a scope to enclose ret, preventing it leaking into the surrounding scope
    {
        auto ret = map.insert({ "hello", 3 });
        if (!ret.second)
            std::cout << "hello already exists with value " << ret.first->second << "\n";
    }
    // we create another scope to enclose ret, again preventing it from leaking out
    {
        auto ret = map.insert({ "foo", 4 });
        if (!ret.second)
            std::cout << "foo already exists with value " << ret.first->second << "\n";
    }

    return 0;
}
```

*Build and run:*

    $ clang++ -std=c++14 main.cpp
    $ ./a.out
    hello already exists with value 1

#### C++17

With the introduction of *if statement with initializer*, we can now create the variable *inside* the if statement.

```cpp
if (init; condition)
```

This makes the code more succint *and* doesn't leak the variable into the surrounding scope.

*Example:*

```cpp
#include <iostream>
#include <map>

int main()
{
    std::map<std::string, int> map;
    map["hello"] = 1;
    map["world"] = 2;

    // intitialize the condition we want to check from within the if statement
    if (auto ret = map.insert({ "hello", 3 }); !ret.second)
        std::cout << "hello already exists with value " << ret.first->second << "\n";

    // ret has not leaked, so we can use that for this conditional check too
    if (auto ret = map.insert({ "foo", 4 }); !ret.second)
        std::cout << "foo already exists with value " << ret.first->second << "\n";

    return 0;
}
```

*Build and run:*

    $ clang++-4.0 -std=c++1z main.cpp
    $ ./a.out
    hello already exists with value 1

Following comments on [reddit](https://www.reddit.com/r/cpp/comments/4z155b/c17_if_statement_with_initializer/d6s9tnz) a perhaps more
illustrative example would be the situation where you attempt to obtain a lock, but need to do something else if it is not currently
available.

Here we attempt to lock a `std::mutex` with a `std::unique_lock`, specifying we should only `try_to_lock`.

We can initialize the `std::unique_lock` from within the if statement that then checks whether we were able to lock the mutex or not.

*Example:*

```cpp
#include <iostream>
#include <mutex>

int main()
{
    std::mutex mtx;
    if (std::unique_lock<std::mutex> l(mtx, std::try_to_lock); l.owns_lock())
    {
        std::cout << "successfully locked the resource\n";

        //...
    }
    else
    {
        std::cout << "resource not currently available\n";
    }
    return 0;
}
```
*Build and run:*

    $ clang++-4.0 -std=c++1z main.cpp
    $ ./a.out
    successfully locked the resource

### Combining with [structured bindings]({% post_url 2016-8-19-structured-bindings %})

The usefulness of *if with initializer* becomes more apparent when combined with *structured bindings*.

In the following example we use structured bindings to unwrap the `std::pair` return value of `std::map::insert` into two
separate variables, `it` (the iterator) and `inserted` (a boolean indicating whether the insert succeeded).

We can then check whether the insert succeeded in the if statement by checking `inserted`.

*Example:*

```cpp
#include <iostream>
#include <map>

int main()
{
    std::map<std::string, int> map;
    map["hello"] = 1;
    map["world"] = 2;

    // intitialize the condition we want to check from within the if statement
    if (auto [it, inserted] = map.insert({ "hello", 3 }); !inserted)
        std::cout << "hello already exists with value " << it->second << "\n";

    // ret has not leaked, so we can use that for this conditional check too
    if (auto [it, inserted] = map.insert({ "foo", 4 }); !inserted)
        std::cout << "foo already exists with value " << it->second << "\n";

    return 0;
}
```

*Build and run:*

    $ clang++-4.0 -std=c++1z main.cpp
    $ ./a.out
    hello already exists with value 1

## Switch statements

#### Pre C++17

Similar annoyances exist with switch statements. The code is more verbose as the variable has to be initialized first, and then only
can the switch occur. Similarly, the variable leaks into the surrounding scope.

*Example:*

```cpp
#include <iostream>

enum Result
{
    SUCCESS,
    DEVICE_FULL,
    ABORTED
};

std::pair<size_t /* bytes */, Result> writePacket()
{
    return { 100, SUCCESS };
}

int main()
{
    // initialize the value we want to switch on (res ends up in surrounding scope)
    auto res = writePacket();

    // then switch on that value
    switch (res.second)
    {
        case SUCCESS:
            std::cout << "successfully wrote " << res.first << " bytes\n";
            break;
        case DEVICE_FULL:
            std::cout << "insufficient space on device\n";
            break;
        case ABORTED:
            std::cout << "operation aborted before completion\n";
            break;
    }
    return 0;
}
```

*Build and run:*

    $ clang++ -std=c++14 main.cpp
    $ ./a.out
    successfully wrote 100 bytes

#### C++17

As with if statements, we can now create the variable *inside* the switch statement.

```cpp
switch (init; var)
```

*Example:*

```cpp
#include <iostream>

enum Result
{
    SUCCESS,
    DEVICE_FULL,
    ABORTED
};

std::pair<size_t /* bytes */, Result> writePacket()
{
    return { 100, SUCCESS };
}

int main()
{
    switch (auto res = writePacket(); res.second)
    {
        case SUCCESS:
            std::cout << "successfully wrote " << res.first << " bytes\n";
            break;
        case DEVICE_FULL:
            std::cout << "insufficient space on device\n";
            break;
        case ABORTED:
            std::cout << "operation aborted before completion\n";
            break;
    }
    return 0;
}
```

Note now that we can initialize `res` *inside* the `switch` statement, and then perform the switch.

    $ clang++-4.0 -std=c++1z main.cpp
    $ ./a.out
    successfully wrote 100 bytes
