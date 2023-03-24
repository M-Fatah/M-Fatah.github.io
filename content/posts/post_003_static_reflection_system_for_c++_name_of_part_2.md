---
title: "Static reflection system for C++ - name_of - part 2"
url: /posts/static_reflection_system_for_c++_name_of_part_2
date: 2023-03-20T03:05:07+02:00
draft: false
---

### Fixing the second issue:
To fix the second issue, the way I did it, it needed to be runtime and also I used some features from `C++20` like `std::string_view::starts_with(...)`. I am sure there are ways to keep it compile time and also compatible with `C++17`, but for me it wasn't a big deal, since I already use `C++20` in my projects and it can always be visited later for improvements.

We can check the type passed as a template paramter and check if it is a primitive type then we can return a string literal with the name directly. If it is not a primitive type, we call a function called `get_type_name(...);` which defines a static `char` buffer that will hold the generated name, then in turns calls another function called `_reflect_append_name(...);`,

```C++
inline static constexpr size_t REFLECT_MAX_NAME_LENGTH = 128;

inline static constexpr void
_reflect_append_name(char *name, u64 &count, std::string_view type_name)
{
    constexpr auto string_append = [](char *string, const char *to_append, u64 &count) {
        while(*to_append != '\0' && count < REFLECT_MAX_NAME_LENGTH - 1)
            string[count++] = *to_append++;
    };

    constexpr auto append_type_name_prettified = [string_append](char *name, std::string_view type_name, u64 &count) {
        if (type_name.starts_with(' '))
            type_name.remove_prefix(1);

        if (type_name.starts_with("const "))
        {
            string_append(name, "const ", count);
            type_name.remove_prefix(6);
        }

        #if defined(_MSC_VER)
        if (type_name.starts_with("enum "))
            type_name.remove_prefix(5);
        else if (type_name.starts_with("class "))
            type_name.remove_prefix(6);
        else if (type_name.starts_with("struct "))
            type_name.remove_prefix(7);
        #endif

        if (type_name.starts_with("signed char"))
        {
            string_append(name, "i8", count);
            type_name.remove_prefix(11);
        }
        else if (type_name.starts_with("short int"))
        {
            string_append(name, "i16", count);
            type_name.remove_prefix(9);
        }
        else if (type_name.starts_with("short") && !type_name.starts_with("short unsigned int"))
        {
            string_append(name, "i16", count);
            type_name.remove_prefix(5);
        }
        else if (type_name.starts_with("int"))
        {
            string_append(name, "i32", count);
            type_name.remove_prefix(3);
        }
        else if (type_name.starts_with("__int64"))
        {
            string_append(name, "i64", count);
            type_name.remove_prefix(7);
        }
        else if (type_name.starts_with("long int"))
        {
            string_append(name, "i64", count);
            type_name.remove_prefix(8);
        }
        else if (type_name.starts_with("unsigned char"))
        {
            string_append(name, "u8", count);
            type_name.remove_prefix(13);
        }
        else if (type_name.starts_with("unsigned short"))
        {
            string_append(name, "u16", count);
            type_name.remove_prefix(14);
        }
        else if (type_name.starts_with("short unsigned int"))
        {
            string_append(name, "u16", count);
            type_name.remove_prefix(18);
        }
        else if (type_name.starts_with("unsigned int"))
        {
            string_append(name, "u32", count);
            type_name.remove_prefix(12);
        }
        else if (type_name.starts_with("unsigned __int64"))
        {
            string_append(name, "u64", count);
            type_name.remove_prefix(16);
        }
        else if (type_name.starts_with("long unsigned int"))
        {
            string_append(name, "u64", count);
            type_name.remove_prefix(17);
        }
        else if (type_name.starts_with("float"))
        {
            string_append(name, "f32", count);
            type_name.remove_prefix(5);
        }
        else if (type_name.starts_with("double"))
        {
            string_append(name, "f64", count);
            type_name.remove_prefix(6);
        }

        for (char c : type_name)
            if (c != ' ')
                name[count++] = c;
    };

    bool add_pointer = false;
    bool add_const   = false;
    if (type_name.ends_with("* const"))
    {
        type_name.remove_suffix(7);
        add_pointer = true;
        add_const = true;
    }

    if (type_name.ends_with(" const "))
    {
        string_append(name, "const ", count);
        type_name.remove_suffix(7);
    }
    else if (type_name.ends_with(" const *"))
    {
        string_append(name, "const ", count);
        type_name.remove_suffix(8);
        add_pointer = true;
    }
    else if (type_name.ends_with('*'))
    {
        type_name.remove_suffix(1);
        add_pointer = true;
    }

    if (type_name.ends_with(' '))
        type_name.remove_suffix(1);

    if (type_name.ends_with('>'))
    {
        u64 open_angle_bracket_pos = type_name.find('<');
        append_type_name_prettified(name, type_name.substr(0, open_angle_bracket_pos), count);
        type_name.remove_prefix(open_angle_bracket_pos + 1);

        name[count++] = '<';
        u64 prev = 0;
        u64 match = 1;
        for (u64 c = 0; c < type_name.length(); ++c)
        {
            if (type_name.at(c) == '<')
            {
                ++match;
            }

            if (type_name.at(c) == '>')
            {
                --match;
                if (match <= 0)
                {
                    _reflect_append_name(name, count, type_name.substr(prev, c - prev));
                    name[count++] = '>';
                    prev = c + 1;
                }
            }

            if (type_name.at(c) == ',')
            {
                _reflect_append_name(name, count, type_name.substr(prev, c - prev));
                name[count++] = ',';
                prev = c + 1;
            }
        }
    }
    else
    {
        append_type_name_prettified(name, type_name, count);
    }

    if (add_pointer)
        name[count++] = '*';
    if (add_const)
        string_append(name, " const", count);
}

template <typename T>
inline static constexpr const char *
name_of()
{
         if constexpr (std::is_same_v<T, i8>)   return "i8";
    else if constexpr (std::is_same_v<T, i16>)  return "i16";
    else if constexpr (std::is_same_v<T, i32>)  return "i32";
    else if constexpr (std::is_same_v<T, i64>)  return "i64";
    else if constexpr (std::is_same_v<T, u8>)   return "u8";
    else if constexpr (std::is_same_v<T, u16>)  return "u16";
    else if constexpr (std::is_same_v<T, u32>)  return "u32";
    else if constexpr (std::is_same_v<T, u64>)  return "u64";
    else if constexpr (std::is_same_v<T, f32>)  return "f32";
    else if constexpr (std::is_same_v<T, f64>)  return "f64";
    else if constexpr (std::is_same_v<T, bool>) return "bool";
    else if constexpr (std::is_same_v<T, char>) return "char";
    else if constexpr (std::is_same_v<T, void>) return "void";
    else
    {
        constexpr auto get_type_name = [](std::string_view type_name) -> const char * {
            static char name[REFLECT_MAX_NAME_LENGTH] = {};
            size_t count = 0;
            _reflect_append_name(name, count, type_name);
            return name;
        };

        #if defined(_MSC_VER)
        constexpr auto type_function_name = std::string_view{__FUNCSIG__};
        constexpr auto type_name_prefix_length = type_function_name.find("name_of<") + 8;
        constexpr auto type_name_length = type_function_name.rfind(">") - type_name_prefix_length;
        #elif defined(__GNUC__)
        constexpr auto type_function_name = std::string_view{__PRETTY_FUNCTION__};
        constexpr auto type_name_prefix_length = type_function_name.find("= ") + 2;
        constexpr auto type_name_length = type_function_name.rfind("]") - type_name_prefix_length;
        #else
        #error "[REFLECT]: Unsupported compiler."
        #endif
        static const char *name = get_type_name({type_function_name.data() + type_name_prefix_length, type_name_length});
        return name;
    }
}
```

What `_reflect_append_name(...);` does, is check if the type contains some keywords that we either remove like in case of `struct ` prefixes that `MSVC` put, or replace it with a more convenient name, like in the cases of primitive types; `signed char` as `i8` for example.

It does this recursively, in the cases where the type names are templated types, so we have to repeat this operation for the type name itself, and also it template argument types.

Here is an example:

```C++
template <typename T, typename R>
struct Foo
{
    T t;
    R r;
};

auto name = name_of<Foo<i8, f32>>();
```

This will get evaluated as following:

```C++
// Extract the type name from the function name;
// MSVC:
    "struct Foo<signed char, float>"
// GCC:
    "Foo<signed char, float>"
// Clang:
    "Foo<signed char, float>"

// Call _reflect_append_name(...):
    // 1. Checks if type is templated => true.
    // 2. append the type name without the template arguments
        // -> MSVC? remove "struct "
        // -> append "Foo" to the name buffer.
    // 3. Get 1st template argument type name.
        // -> primitive type? then replace it; for ex: "signed char" => "i8".
        // -> otherwise copy the name as is.
    // 4. repeat till all template arguments are evaluated.
// Return a pointer to the start of the static name buffer.
```

This only works with `C++20` as I have used methods like `std::string_view::starts_with(...)`, which were introduced in `C++20`, and unfortunately does not work during compile time anymore; I am sure there are ways to write the equivalent of this in `C++17` while keeping it compile time, but I didn't bother since I already use `C++20` in my main projects and the runtime cost of this is not high anyways (only evaluated the first time you call `name_of<T>();` then caches the resulted name).

So why did I go the extra mile for this? well, for consistency; I want type names accross all supported compilers to be exactly the same, the above implementation although lengthy, in its core it is simple; remove extra un-needed prefixes put by `MSVC`, place `const` keyword consistently across compilers and replace primitive types names like `signed char` with `i8` for example.

In the next article we will discuss how to implement `kind_of<T>()`.
[Static reflection system for C++ - kind_of](https://M-Fatah.github.io/posts/static_reflection_system_for_c++_kind_of)