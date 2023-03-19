---
title: "Static reflection system for C++"
url: /posts/static-reflection-system-for-C++
date: 2023-03-20T01:05:07+02:00
draft: false
---

Recently I have been experimenting with a simple [reflection](https://github.com/M-Fatah/reflect) library to use for generic serialization and logging. In this blog post series, I will share the details on how I implemented it. Inspired by golang's [reflect](https://pkg.go.dev/reflect) package, although not feature rich like it unfortunately.

So, first things first, I decided to go with `C++20` but I think with some tweaks, you can get it to work with `C++17`.

### Features:
- Single file header only library.
- No allocations, type info is stored statically.
- Primitive types, pointers, arrays and enums are supported out of the box.
- Type info for user defined types are generated with minimal writing overhead.
- User defined custom annotations for struct fields stored statically with the type info.

### Prerequisites:
- **MSVC:** `-std:c++20 -Zc:preprocessor`. => This will tell MSVC to conform to the preprocessor standard.
- **GCC:** `-std=c++2b`.

### A brief introduction to the API:
```C++
// name_of<Foo>(); => "Foo".
auto n = name_of<T>();

// (TYPE_KIND_POINTER, TYPE_KIND_STRUCT, ..etc).
auto k = kind_of<T>(); // By type.
auto k = kind_of(T{}); // By instance.

// Get type info.
auto t = type_of<T>(); // By type.
auto t = type_of(T{}); // By instance.

// Get value (pointer to the data, accompanied by the type info).
auto v = value_of(T&&);

struct Foo
{
    f32 x;
    void *internal;
};

// Member fields can be tagged with optional custom user defined annotations.
TYPE_OF(Foo, x, (internal, "NoSerialize"))

/*
{
    .name = "Foo",
    .kind = TYPE_KIND_STRUCT,
    .size = sizeof(Foo),
    .align = alignof(Foo),
    .as_struct.fields = {
        {"x", offsetof(Foo, x), type_of<f32>(), ""},
        {"internal", offsetof(Foo, internal), type_of<void*>(), "NoSerialize"}
    },
    .as_struct.field_count = 2
}
*/
auto foo_type = type_of<Foo>();

// Works on template types also.
template <typename T>
struct Bar
{
    T t;
};

TYPE_OF(Foo, t)

/*
{
    .name = "Bar<Foo>",
    .kind = TYPE_KIND_STRUCT,
    .size = sizeof(Bar<Foo>),
    .align = alignof(Bar<Foo>),
    .as_struct.fields = {
        {"t", offsetof(Bar<Foo>, t), type_of<Foo>(), ""},
    },
    .as_struct.field_count = 1
}
*/
auto bar_type = type_of<Bar<Foo>>();
```

### Limitations and things to improve:
- Make it fully compile time.
- Functions reflection is not supported yet.
- Support C++17.
- Support Clang.

### Let's get started:
I will break down each function implementation in its own article
- [Static reflection system for C++ - name_of](https://M-Fatah.github.io/posts/static-reflection-system-for-C++-name_of)
- [Static reflection system for C++ - kind_of](https://M-Fatah.github.io/posts/static-reflection-system-for-C++-kind_of)
- [Static reflection system for C++ - type_of] (WIP)
- [Static reflection system for C++ - value_of] (WIP)