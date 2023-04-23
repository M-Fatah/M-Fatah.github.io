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
#include <type_traits>

template <typename T>
inline static constexpr TYPE_KIND
kind_of()
{
	using Type = std::remove_cvref_t<T>;
	if constexpr (std::is_same_v<Type, int8_t>)
		return TYPE_KIND_I8;
	else if constexpr (std::is_same_v<Type, int16_t>)
		return TYPE_KIND_I16;
	else if constexpr (std::is_same_v<Type, int32_t>)
		return TYPE_KIND_I32;
	else if constexpr (std::is_same_v<Type, int64_t>)
		return TYPE_KIND_I64;
	else if constexpr (std::is_same_v<Type, uint8_t>)
		return TYPE_KIND_U8;
	else if constexpr (std::is_same_v<Type, uint16_t>)
		return TYPE_KIND_U16;
	else if constexpr (std::is_same_v<Type, uint32_t>)
		return TYPE_KIND_U32;
	else if constexpr (std::is_same_v<Type, uint64_t>)
		return TYPE_KIND_U64;
	else if constexpr (std::is_same_v<Type, float>)
		return TYPE_KIND_F32;
	else if constexpr (std::is_same_v<Type, double>)
		return TYPE_KIND_F64;
	else if constexpr (std::is_same_v<Type, bool>)
		return TYPE_KIND_BOOL;
	else if constexpr (std::is_same_v<Type, char>)
		return TYPE_KIND_CHAR;
	else if constexpr (std::is_same_v<Type, void>)
		return TYPE_KIND_VOID;
	else if constexpr (std::is_pointer_v<Type>)
		return TYPE_KIND_POINTER;
	else if constexpr (std::is_array_v<Type>)
		return TYPE_KIND_ARRAY;
	else if constexpr (std::is_enum_v<Type>)
		return TYPE_KIND_ENUM;
	else if constexpr (std::is_compound_v<Type>)
		return TYPE_KIND_STRUCT;
}
```

Let's also define a version that works with `rvalues`; to support literals `kind_of(2);` and object instances `kind_of(T{});`.

```C++
template <typename T>
inline static constexpr TYPE_KIND
kind_of(T &&)
{
    return kind_of<T>();
}
```

You can view and edit the source code on compiler explorer [here](https://godbolt.org/z/axdzGTGGf).

In the next [article](https://M-Fatah.github.io/posts/static_reflection_system_for_C++_type_of) we will discuss how to implement `type_of<T>()`.