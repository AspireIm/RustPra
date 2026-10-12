Rust 里所有类型都可以按**赋值/传参时的行为**分成两大类：

- **Copy 类型**：赋值、传参时自动复制一份，原变量还能用。
- **非 Copy 类型**：赋值、传参时发生移动，所有权转移，原变量失效。

判断标准不是“是不是 `String` 或 `i32`”，而是：

> **这个类型有没有实现 `Copy` trait。**

---

## 一、实现了 `Copy` 的类型

这些类型通常很小、完全在栈上、没有堆资源、不需要手动释放。

| 类型 | 例子 | 说明 |
|---|---|---|
| 整数 | `i8` `i16` `i32` `i64` `i128` `isize` `u8` `u16` `u32` `u64` `u128` `usize` | 都是 Copy |
| 浮点数 | `f32` `f64` | 都是 Copy |
| 布尔 | `bool` | Copy |
| 字符 | `char` | Copy |
| 单元类型 | `()` | Copy |
| 共享引用 | `&T` | Copy，可以复制多个共享引用 |
| 函数指针 | `fn(i32) -> i32` | Copy |
| 元组 | `(i32, bool)` | 如果每个元素都 Copy，整个元组就 Copy |
| 数组 | `[i32; 3]` | 如果元素 Copy，数组就 Copy |
| `Option<T>` | `Option<i32>` | 如果 `T: Copy`，则 `Option<T>` 也 Copy |
| `Result<T, E>` | `Result<i32, bool>` | 如果 `T` 和 `E` 都 Copy，则整体 Copy |
| 自定义结构体/枚举 | `#[derive(Copy, Clone)]` | 必须显式派生，且所有字段都 Copy |

```rust
let x = 5;
let y = x;
println!("{}", x); // ✅ x 还能用，因为 i32 是 Copy
```

---

## 二、没有实现 `Copy` 的类型

这些类型通常拥有堆内存、文件句柄、锁、引用计数等资源，需要析构，不能简单按位复制。

| 类型 | 例子 | 说明 |
|---|---|---|
| 堆字符串 | `String` | 拥有堆内存，移动 |
| 动态数组 | `Vec<T>` | 拥有堆内存，移动 |
| 堆分配盒子 | `Box<T>` | 拥有堆内存，移动 |
| 引用计数指针 | `Rc<T>` `Arc<T>` | 不是 Copy，但 `clone()` 会增加引用计数 |
| 资源句柄 | `File` `TcpStream` `MutexGuard` | 拥有系统资源，移动 |
| 内部可变性 | `RefCell<T>` | 不是 Copy |
| 可变引用 | `&mut T` | 不是 Copy，因为可变借用是独占的 |
| 自定义类型 | 默认不是 Copy | 除非显式 `#[derive(Copy, Clone)]` 且字段全 Copy |

```rust
let s = String::from("hello");
let t = s;
// println!("{}", s); // ❌ s 已经被移动，不能再用
```

---

## 三、`&T` 和 `&mut T` 的区别很重要

| 引用类型 | 是否 Copy | 原因 |
|---|---|---|
| `&T` 共享引用 | ✅ Copy | 多个不可变引用可以同时存在 |
| `&mut T` 可变引用 | ❌ 非 Copy | 可变引用必须独占，不能随便复制 |

```rust
let mut x = 5;
let r1 = &x;
let r2 = r1; // ✅ &T 是 Copy，r1 还能用

let m1 = &mut x;
let m2 = m1; // m1 被移动，不能再使用
```

---

## 四、元组、数组、`Option`、`Result` 看内部

它们是否 Copy，取决于内部元素是否全部 Copy：

```rust
let a = (1, true);          // ✅ Copy，因为 i32 和 bool 都 Copy
let b = a;
println!("{:?}", a);        // a 还能用

let c = (String::from("a"), 1); // ❌ 非 Copy，因为 String 非 Copy
let d = c;
// println!("{:?}", c);     // c 不能用了
```

```rust
let o1 = Some(5);           // ✅ Option<i32> 是 Copy
let o2 = o1;
println!("{:?}", o1);       // o1 还能用

let o3 = Some(String::from("a")); // ❌ Option<String> 非 Copy
let o4 = o3;
// println!("{:?}", o3);    // o3 不能用了
```

---

## 五、自定义类型默认是移动语义

你自己定义的结构体或枚举，默认**不是 Copy**，赋值/传参时移动。

```rust
struct Point {
    x: i32,
    y: i32,
}

let p1 = Point { x: 1, y: 2 };
let p2 = p1;
// println!("{}", p1.x); // ❌ 默认移动，p1 不能用了
```

如果想让它变成 Copy，需要显式派生：

```rust
#[derive(Copy, Clone)]
struct Point {
    x: i32,
    y: i32,
}

let p1 = Point { x: 1, y: 2 };
let p2 = p1;
println!("{}", p1.x); // ✅ 现在可以了
```

但前提是：**所有字段都必须是 Copy**。如果字段里有 `String`，就不能派生 `Copy`。

---

## 六、`Copy` 和 `Clone` 的区别

| | `Copy` | `Clone` |
|---|---|---|
| 是否隐式 | 是，赋值/传参自动发生 | 否，必须显式调用 `.clone()` |
| 开销 | 通常很小，按位复制 | 可能很大，比如深拷贝堆数据 |
| 原变量 | 还能用 | 还能用 |
| 例子 | `i32`、`&T` | `String`、`Vec<T>`、`Rc<T>` |

```rust
let s = String::from("hello");
let t = s.clone(); // 显式深拷贝堆上的 "hello"
println!("{}", s); // ✅ s 还能用
```

注意：`Rc<T>::clone()` 不是深拷贝，只是增加引用计数；`String::clone()` 是深拷贝堆数据。

---

## 七、总结口诀

- **标量、共享引用、函数指针** → 通常是 `Copy`，复制。
- **有堆、有资源、需要 `Drop`** → 通常非 `Copy`，移动。
- **元组、数组、`Option`、`Result`** → 看内部是否全是 `Copy`。
- **可变引用 `&mut T`** → 非 `Copy`，移动。
- **自定义类型** → 默认移动，除非显式 `#[derive(Copy, Clone)]` 且字段全 `Copy`。

一句话：

> **Copy 类型 = 可以安全按位复制的纯栈上数据；非 Copy 类型 = 拥有堆或资源，必须移动所有权，由最后一个所有者负责释放。**