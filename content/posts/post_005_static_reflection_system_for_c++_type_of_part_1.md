---
title: "Static reflection system for C++ - type_of - part 1"
url: /posts/static_reflection_system_for_c++_type_of_part_1
date: 2024-01-07T02:50:07+02:00
draft: false
---

Now we are done with `name_of<T>()` and `kind_of<T>()`. Let's start implementing `type_of<T>()`.

## Type

First we need to define what `Type` is, we will start simple and store only the `name`, `kind`, `size` and `align` of the reflected type for now.

```C++
struct Type
{
    const char *name;
    TYPE_KIND kind;
    size_t size;
    size_t align;
};
```

Later on we can expand it to store more information such as:
* pointee's type info in case of `pointers`.
* element's type info in case of `arrays`.
* enum values (name, index) in case of `enums`.
* member fields' type info in case of `structs`.

## type_of\<T>() and type_of(T{})

We want to be able to write `type_of<float>()` and `type_of(2.0f)` to get the exact same type info, to do so we need to define an overload for each type we want to reflect on.

## Fundamental types:

Starting by the fundamental types (`int`, `float`, etc.), we can write a templated version of `type_of(const T)` and use `C++`'s `constraints` to constraint it to those types only, except for `void`, as `sizeof(T)` and `alignof(T)` does not work with `void` and will result in a compilation error.

```C++
template <typename T>
requires (std::is_fundamental_v<T> && !std::is_void_v<T>)
inline static constexpr const Type *
type_of(const T)
{
    static const Type self = {
        .name = name_of<T>(),
        .kind = kind_of<T>(),
        .size = sizeof(T),
        .align = alignof(T)
    };
    return &self;
}
```

and for `void`, we can just overload the function like this:

```C++
template <typename T>
requires (std::is_void_v<T>)
inline static constexpr const Type *
type_of()
{
    static const Type self = {
        .name = name_of<T>(),
        .kind = kind_of<T>(),
        .size = 0,
        .align = 0
    };
    return &self;
}
```

This solves it for `type_of(2.0f)`, `type_of(false)`, etc. To be able to call `type_of<float>()` or `type_of<bool>()` for example, we need to define `type_of<T>()` as follows:

```C++
template <typename T>
inline static constexpr const Type *
type_of()
{
    return type_of(T{});
}
```

## Conclusion

We started by the very basics, but we have setup ourselves for success, as pretty much what comes next follows the same pattern as this, of course with a little bit of added complexity but nothing major in particular.

You can view and edit the source code on compiler explorer [here](https://godbolt.org/z/dT9bdEW6T).

In the next [article](https://M-Fatah.github.io/posts/static_reflection_system_for_C++_type_of_part_2) we will discuss how to implement `type_of<T>()` for `pointers`.