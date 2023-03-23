---
title: "Static reflection system for C++ - kind_of"
url: /posts/static_reflection_system_for_c++_kind_of
date: 2023-03-20T04:05:07+02:00
draft: false
---

We need a way to determine if a type is a struct or enum or an array or a primitive type, etc...
So first let's define an enum called `TYPE_KIND` as following:

```C++
enum TYPE_KIND
{
    TYPE_KIND_I8,
    TYPE_KIND_I16,
    TYPE_KIND_I32,
    TYPE_KIND_I64,
    TYPE_KIND_U8,
    TYPE_KIND_U16,
    TYPE_KIND_U32,
    TYPE_KIND_U64,
    TYPE_KIND_F32,
    TYPE_KIND_F64,
    TYPE_KIND_BOOL,
    TYPE_KIND_CHAR,
    TYPE_KIND_VOID,
    TYPE_KIND_POINTER,
    TYPE_KIND_ARRAY,
    TYPE_KIND_ENUM,
    TYPE_KIND_STRUCT
};
```

Then we can take advantage of  `C++`'s `type_traits` to implement `kind_of<T>()` as following:

```C++
template <typename T>
inline static constexpr TYPE_KIND
kind_of()
{
    if constexpr (std::is_same_v<T, i8>)
        return TYPE_KIND_I8;
    else if constexpr (std::is_same_v<T, i16>)
        return TYPE_KIND_I16;
    else if constexpr (std::is_same_v<T, i32>)
        return TYPE_KIND_I32;
    else if constexpr (std::is_same_v<T, i64>)
        return TYPE_KIND_I64;
    else if constexpr (std::is_same_v<T, u8>)
        return TYPE_KIND_U8;
    else if constexpr (std::is_same_v<T, u16>)
        return TYPE_KIND_U16;
    else if constexpr (std::is_same_v<T, u32>)
        return TYPE_KIND_U32;
    else if constexpr (std::is_same_v<T, u64>)
        return TYPE_KIND_U64;
    else if constexpr (std::is_same_v<T, f32>)
        return TYPE_KIND_F32;
    else if constexpr (std::is_same_v<T, f64>)
        return TYPE_KIND_F64;
    else if constexpr (std::is_same_v<T, bool>)
        return TYPE_KIND_BOOL;
    else if constexpr (std::is_same_v<T, char>)
        return TYPE_KIND_CHAR;
    else if constexpr (std::is_same_v<T, void>)
        return TYPE_KIND_VOID;
    else if constexpr (std::is_pointer_v<T>)
        return TYPE_KIND_POINTER;
    else if constexpr (std::is_array_v<T>)
        return TYPE_KIND_ARRAY;
    else if constexpr (std::is_enum_v<T>)
        return TYPE_KIND_ENUM;
    else if constexpr (std::is_compound_v<T>)
        return TYPE_KIND_STRUCT;
}
```

Let's also define a version that works with rvalues; to support literals `kind_of(2);` and object instances `kind_of(T{});`.

```C++
template <typename T>
inline static constexpr TYPE_KIND
kind_of(T&&)
{
    return kind_of<T>();
}
```

In the next article we will discuss how to implement `type_of<T>()`. [Static reflection system for C++ - type_of](https://M-Fatah.github.io/posts/static-reflection-system-for-C++-type_of)