---
title: "Static reflection system for C++ - name_of - part 2"
url: /posts/static_reflection_system_for_c++_name_of_part_2
date: 2023-03-20T03:05:07+02:00
draft: false
---

To fix the second issue, we will parse the type name string at runtime and generate a prettified and consistent one across major compilers.

We will start by checking the template parameter type, if it is a primitive type, we can just return a string literal with the correct name, otherwise we parse it.

```C++
inline static constexpr size_t REFLECT_MAX_NAME_LENGTH = 128;

template <typename T>
inline static constexpr const char *
name_of()
{
         if constexpr (std::is_same_v<T, int8_t>)   return "i8";
    else if constexpr (std::is_same_v<T, int16_t>)  return "i16";
    else if constexpr (std::is_same_v<T, int32_t>)  return "i32";
    else if constexpr (std::is_same_v<T, int64_t>)  return "i64";
    else if constexpr (std::is_same_v<T, uint8_t>)  return "u8";
    else if constexpr (std::is_same_v<T, uint16_t>) return "u16";
    else if constexpr (std::is_same_v<T, uint32_t>) return "u32";
    else if constexpr (std::is_same_v<T, uint64_t>) return "u64";
    else if constexpr (std::is_same_v<T, float>)    return "f32";
    else if constexpr (std::is_same_v<T, double>)   return "f64";
    else if constexpr (std::is_same_v<T, bool>)     return "bool";
    else if constexpr (std::is_same_v<T, char>)     return "char";
    else if constexpr (std::is_same_v<T, void>)     return "void";
    else
    {
        constexpr auto _name_of = [](std::string_view type_name) -> const char * {
            // a static buffer to hold the prettified type name, it lasts for the duration of the application life time.
            static char name[REFLECT_MAX_NAME_LENGTH] = {};
            size_t count = 0;
            _name_of_parse_and_append(name, count, type_name);
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

        // Generates the name the first time then caches it for subsequent calls.
        static const char *name = _name_of(type_function_name.substr(type_name_prefix_length, type_name_length));
        return name;
    }
}
```

To parse the type name; first we check if the type itself is a template type or not. If it is not a template type we parse and append the result to the buffer, If it is a template type we divide it into two parts, the type and template parameter type, for example if we have type `Foo<int64_t>` we divide it into `Foo` and `int64_t` then parse each part and append the result to the buffer. We do this recursively to handle nested template types.

Example:
```
// MSVC:
1. struct Foo<struct Foo<__int64>> -> `struct Foo` and `struct Foo<__int64>`.
2. struct Foo                      -> `Foo`                                   // Output: `Foo<`
3. struct Foo<__int64>             -> `struct Foo` and `__int64`.
4. struct Foo                      -> `Foo`                                   // Output: `Foo<Foo<`
5. __int64                         -> `i64`                                   // Output: `Foo<Foo<i64>>`

// GCC:
1. Foo<Foo<long int>>              -> `Foo` and `Foo<long int>`.
2. Foo                             -> `Foo`                                   // Output: `Foo<`
3. Foo<long int>                   -> `Foo` and `long int`.
4. Foo                             -> `Foo`                                   // Output: `Foo<Foo<`
5. long int                        -> `i64`                                   // Output: `Foo<Foo<i64>>`

// Clang:
1. Foo<Foo<long>>                  -> `Foo` and `Foo<long>`.
2. Foo                             -> `Foo`                                   // Output: `Foo<`
3. Foo<long>                       -> `Foo` and `long`.
4. Foo                             -> `Foo`                                   // Output: `Foo<Foo<`
5. long                            -> `i64`                                   // Output: `Foo<Foo<i64>>`
```

The parsing function does other things to keep name output consistent; it handles the placement of `const` keyword as well as the pointer `*` and reference `&` operators.

```C++
inline static constexpr void
_name_of_parse_and_append(char *name, size_t &count, std::string_view type_name)
{
    constexpr auto string_append = [](char *string, const char *to_append, size_t &count) {
        while (*to_append != '\0' && count < REFLECT_MAX_NAME_LENGTH - 1)
            string[count++] = *to_append++;
    };

    constexpr auto append_type_name_prettified = [string_append](char *name, std::string_view type_name, size_t &count) {
        if (type_name.starts_with(' '))
            type_name.remove_prefix(1);

        bool add_pointer = false;
        if (type_name.starts_with("const "))
        {
            string_append(name, "const ", count);
            type_name.remove_prefix(6);
        }
        else if (type_name.ends_with(" const *"))
        {
            string_append(name, "const ", count);
            type_name.remove_suffix(8);
            add_pointer = true;
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

        if (add_pointer)
            name[count++] = '*';
    };

    bool add_const     = false;
    bool add_pointer   = false;
    bool add_reference = false;
    if (type_name.ends_with("* const"))
    {
        type_name.remove_suffix(7);
        add_const = true;
        add_pointer = true;
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
    else if (type_name.ends_with("const &"))
    {
        add_const = true;
        add_reference = true;
        type_name.remove_suffix(7);
    }
    else if (type_name.ends_with(" const&"))
    {
        add_const = true;
        add_reference = true;
        type_name.remove_suffix(7);
    }
    else if (type_name.ends_with('*'))
    {
        type_name.remove_suffix(1);
        add_pointer = true;
    }
    else if (type_name.ends_with('&'))
    {
        type_name.remove_suffix(1);
        add_reference = true;
    }

    if (type_name.ends_with(' '))
        type_name.remove_suffix(1);

    if (type_name.ends_with('>'))
    {
        size_t open_angle_bracket_pos = type_name.find('<');
        append_type_name_prettified(name, type_name.substr(0, open_angle_bracket_pos), count);
        type_name.remove_prefix(open_angle_bracket_pos + 1);

        name[count++] = '<';
        size_t prev = 0;
        size_t match = 1;
        for (size_t c = 0; c < type_name.length(); ++c)
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
                    _name_of_parse_and_append(name, count, type_name.substr(prev, c - prev));
                    name[count++] = '>';
                    prev = c + 1;
                }
            }

            if (type_name.at(c) == ',')
            {
                _name_of_parse_and_append(name, count, type_name.substr(prev, c - prev));
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
    if (add_reference)
        name[count++] = '&';
}
```

You can view and edit the source code on compiler explorer [here](https://godbolt.org/z/zaPh74M1W).

In the next [article](https://M-Fatah.github.io/posts/static_reflection_system_for_c++_kind_of) we will discuss how to implement `kind_of<T>()`.