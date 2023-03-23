---
title: "Static reflection system for C++ - name_of - part 1"
url: /posts/static_reflection_system_for_c++_name_of_part_1
date: 2023-03-20T02:05:07+02:00
draft: false
---

Unfortunately, C++ does not provide a way to get type names properly, at least in a consistent manner between major compilers; for example, using `typeid(T).name()` will return mangled names on `GCC` and `Clang`.

Fortunately there is a workaround to solve this problem; compilers provide a macro definition that gives you the function name along with its arguments and template information at compile time.

`MSVC` provides `__FUNCSIG__`, while `GCC` and `Clang` provide `__PRETTY_FUNCTION__`.

So, if we define the function `name_of<T>()` as following:

```C++
template <typename T>
inline static constexpr const char *
name_of()
{
#if defined(_MSC_VER)
    return __FUNCSIG__;
#elif defined(__GNUC__) || defined(__clang__)
    return __PRETTY_FUNCTION__;
#endif
}
```

and we call it with T = char; `name_of<char>();` it will return:

```C++
// MSVC:
"const char *__cdecl name_of<char>(void)"

// GCC:
"constexpr const char* name_of() [with T = char]"

// Clang:
"const char *name_of() [T = char]"
```

We can then extract the type name from the function name like this:

```C++
template <typename T>
inline static constexpr std::string_view
name_of()
{
#if defined(_MSC_VER)
    constexpr auto type_function_name = std::string_view{__FUNCSIG__};
    constexpr auto type_name_prefix_length = type_function_name.find("name_of<") + 8;
    constexpr auto type_name_length = type_function_name.rfind(">") - type_name_prefix_length;
#elif defined(__GNUC__) || defined(__clang__)
    constexpr auto type_function_name = std::string_view{__PRETTY_FUNCTION__};
    constexpr auto type_name_prefix_length = type_function_name.find("= ") + 2;
    constexpr auto type_name_length = type_function_name.rfind("]") - type_name_prefix_length;
#else
    #error "[REFLECT]: Unsupported compiler."
#endif
    return type_function_name.substr(type_name_prefix_length, type_name_length);
}
```

This works perfectly fine, however there are two issues:
1. The entire function name will be embedded into the executable, although we are just interested in only the type name.
2. Type names may differ between different compilers:
```C++
struct Foo { };
auto foo_name = name_of<Foo>();
auto i64_name = name_of<int64_t>();
// MSVC:
"struct Foo"
"__int64"

// GCC:
"Foo"
"long int"

// Clang:
"Foo"
"long"
```

### Fixing the first issue:
We can fix the first issue by just copying the type name part to a static `std::array<char>` and then return it.

```C++
template <typename T>
inline static constexpr const char *
name_of()
{
    constexpr auto _name_of = []<u64 ...indices>(std::string_view type_name, std::index_sequence<indices...>) {
        return std::array{type_name[indices]..., '\0'};
    };

    #if defined(_MSC_VER)
        constexpr auto type_function_name = std::string_view{__FUNCSIG__};
        constexpr auto type_name_prefix_length = type_function_name.find("name_of<") + 8;
        constexpr auto type_name_length = type_function_name.rfind(">") - type_name_prefix_length;
    #elif defined(__GNUC__) || defined(__clang__)
        constexpr auto type_function_name = std::string_view{__PRETTY_FUNCTION__};
        constexpr auto type_name_prefix_length = type_function_name.find("= ") + 2;
        constexpr auto type_name_length = type_function_name.rfind("]") - type_name_prefix_length;
    #else
        #error "[REFLECT]: Unsupported compiler."
    #endif

    constexpr auto type_name = type_function_name.substr(type_name_prefix_length, type_name_length);
    static constexpr auto name = _name_of(type_name, std::make_index_sequence<type_name.length()>());
    return name.data();
}
```

Now, we can print type names like this:

```C++
printf("%s\n", name_of<char>());
printf("%s\n", name_of<signed char>());
printf("%s\n", name_of<float>());
printf("%s\n", name_of<int>());
```

The pros of this approach are:
- Works during compile time.
- Stores the type name efficiently without wasting storage.

But we still did not fix the second issue; to fix the second issue, we need to break the compile time rule, and do string manipulations during runtime to produce consistent type names accross all compilers. It is still very efficient to do so; `name_of<T>();` only calculates the name the first time it gets called, then caches the result in a static variable; subsequent calls return the cached name string.

In the next article we will discuss how to fix the second issue.
[Static reflection system for C++ - name_of - part 2](https://M-Fatah.github.io/posts/static_reflection_system_for_C++_name_of_part_2)