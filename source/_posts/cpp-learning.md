---
title: C++ 量化开发学习日记（一）
date: 2026-07-24 11:58:22
tags:
  - C++
  - 量化开发
  - 日记
categories:
  - 技术
---

今天开始更新 C++ 学习笔记，主要记录我个人学习 C++ 量化开发的过程。

我本身是 ACM 选手，算法基础没问题，但确实没怎么用 C++ 写过工程类项目。最近想往量化开发方向跳槽，故开始系统学习 C++ 工程实践。

---

## 1. `<cstdint>` —— 定长整数类型

这个库提供了 `uint8_t`、`uint16_t`、`int8_t` 等定长整数类型，核心价值在于**跨平台保证类型的字节宽度一致**。

比如 `uint8_t` 在所有平台上都固定占 8 位（1 字节），不会出现某个平台 `int` 是 4 字节、另一个平台是 8 字节的情况。

- `uint8_t` 是 **u**nsigned int 8-bit 的缩写，表示无符号 8 位整数（非负）
- `int8_t` 则是有符号 8 位整数

> 量化场景下对数据精度和存储大小敏感，定长类型很重要。

---

## 2. `using` 类型别名

```cpp
using OrderId = uint64_t;
```

作用是给类型起别名。后续直接用 `OrderId` 代替 `uint64_t`：

```cpp
using OrderId = uint64_t;
OrderId order_id_1;  // 等价于 uint64_t order_id_1;
```

这样做的好处：如果以后想换一种 ID 类型，只需要改 `using` 那一行，不用全局搜索替换。

---

## 3. Clang 与 GCC

两个都是 C++ 编译器，负责将 C++ 源码编译为机器码。

- **GCC**：Linux 平台默认，历史悠久
- **Clang**：macOS 默认编译器，错误提示更友好，编译速度更快

可以用 `g++ --version` 或 `clang++ --version` 查看当前用的是哪个。

---

## 4. `enum` 枚举类型

### 传统枚举

```cpp
enum Color {
    Green,  // 默认值 0
    Red,    // 递增为 1
    Blue    // 递增为 2
};

Color a = Green;  // a = 0
```

传统枚举的缺陷：不同 enum 中不能有同名成员，会产生**重复定义编译错误**：

```cpp
enum Color  { Green, Blue };
enum Color1 { Green };        // 错误！Green 重复定义
```

### `enum class`（C++11）

C++11 引入了强类型枚举，解决了作用域污染问题：

```cpp
enum class Color : uint8_t {   // 指定底层类型为 uint8_t
    Green,
    Red,
    Blue
};

Color color = Color::Green;    // 必须带作用域访问
```

优势：
- **作用域隔离**：不同 `enum class` 可以有同名成员
- **可指定底层类型**：`: uint8_t` 控制内存占用
- **类型安全**：不能隐式转换为 int

> 量化开发中订单状态、指令类型等场景非常适合用 `enum class`。

---

## 5. `#pragma once` —— 防止头文件重复包含

一个预处理指令，保证头文件在**单次编译**中只会被包含一次。

典型场景：
```
b.hpp 引用了 a.hpp
main.cpp 同时引用了 a.hpp 和 b.hpp
```

编译时 `a.hpp` 的内容会被 `main.cpp` 和 `b.hpp` 各展开一次，如果没有保护，`a.hpp` 中的定义会出现两次，导致**重复定义编译错误**。

`#pragma once` 就是最简单有效的保护手段。等价于传统的 include guard：

```cpp
// 传统写法
#ifndef FOO_HPP
#define FOO_HPP
// ... 头文件内容
#endif

// 现代写法（更简洁）
#pragma once
// ... 头文件内容
```

---

## 6. `[[nodiscard]]`（C++17）

属性标记，要求**调用者必须使用该函数的返回值**，否则编译器发出警告。

```cpp
[[nodiscard]] Quantity remaining() const {
    return quantity - filled_qty;
}
```

调用 `remaining()` 后如果不接收返回值，编译器会警告。适用于：
- 纯计算函数（返回值就是唯一目的）
- 错误码返回（避免调用者忽略错误）
- 资源获取函数

---

## 7. `namespace` —— 命名空间

用 `namespace{}` 包裹代码可以防止命名冲突：

```cpp
namespace Order1 {
    struct Order {
        // ...
    };
}

namespace Order2 {
    struct Order {
        // ...
    };
}

Order1::Order order1;  // 通过命名空间区分
Order2::Order order2;
```

两个 `Order` 结构体同名但分属不同命名空间，互不冲突。在实际项目中，不同模块用不同 namespace 是标准做法。
