---
layout: post
title: Rust 中级教程-接口设计建议-03-Obvious
date: 2023-10-08 16:45:30.000000000 +09:00
categories: [Rust, Rustaceans]
tags: [Rust, Rustaceans]
---


# 4 显而易见（Obvious）

## 4.1 前言

* 你接口的用户通常是不了解你接口实现的细节，所以他可能不会完全理解接口的所有规则和限制

> 因此，重要的一点是，要让用户容易理解你的接口，并难以用错，通过 Rust 文档和类型系统可以基本实现这个需求
{: .prompt-info }


## 4.2 文档

### 让接口透明化的第一步就是：写出好的文档，我们来看看下面几点

* 1 清楚的记录
  * 可能出现意外的情况，或它依赖于用户执行超出类型签名要求的操作
  * 例如 `panic` 如果你的代码会发生 `panic`，需要记录在文档中，说明什么情况下会发生 `panic`
  * 例如 返回错误，也要在文档中写明什么情况会返回错误
  * 例如 `unsafe` 函数，要写明用户要满足什么条件才能安全的调用这个函数
  * 看个例子
  

```rust
/// 除法运行，返回两个数的结果
/// 
/// # Panics
/// 
/// 如果除数为零，该函数会发生 panic。
/// 
/// # 示例
/// 
/// ````
/// let result = divide(10, 2);
/// assert_eq!(result, 5);
/// ````
pub fn divide(dividend: i32, divisor: i32) -> i32 {
    // Code...
    todo!()
}
```

* 2 要写出好的文档，需要在 `crate 或 module 级别`，包括端到端的用例
  * 而不是针对特定类型或方法，这么做的好处是让用户可以了解所有内容如何组合到一起
  * 并且让用户对接口的整体结构有一个相对清晰的理解
    * 从而让开发者快速了解到各方法和类型的功能，以及在哪使用它们
  * 你提供这个用例后，用户就可以把它们直接粘贴使用了，这就相当于提供了定制化使用的起点，让其结合自己的需求进行修改


* 3 组织好文档
  * 即利用模块来将语义相关的项进行分组
  * 使用内部的文档链接将这些项相互链接起来
  * 可以考虑使用 `#[doc(hidden)]` 标记那些不打算公开，但出于遗留原因需要的接口部分，避免弄乱文档
  * 下面是例子
  
  
```rust
// 一个简单的模块，包含一些用于内部使用的函数和结构体
pub mod internal {
    /// 一个拥有内部计算辅助函数 (因为标记了 #[doc(hidden)]，所以这个句不会出现在文档中)
    #[doc(hidden)]
    pub fn internal_helper() {
        // 内部计算实现...
    }

    /// 一个仅用于内部使用的结构体
    #[doc(hidden)]
    pub struct InternalStruct {
        // 字段和方法
    }
}

// 一个公共接口函数，调用了内部辅助函数
pub fn public_function() {
    // 调用内部辅助函数
    internal::internal_helper();
}
```

* 4 尽可能的丰富你的文档
  * 有时需要解释一些概念，就可以添加到外部资源的链接
    * 例如相关规范文档（RFC）、博客、白皮书等等
  * 可使用 `#[doc(cfg(..))]` 突出显示仅在特定配置下可用的项
    * 这样用户就能快速了解到，为什么在文档中列出的某个方法不可用，因为其只能在特定的条件下使用
  * 可使用 `#[doc(aliias = "...")]` 可以让用户以其他名称搜索到类型和方法
  * 在顶层文档中，引导用户了解常用的模块、Trait、类型、方法等等
  * 下面看两个例子

```rust
///! 这是一个用于处理图像的库
///!
///! 这个库提供了一些常用的图像处理功能，例如
///! - 读取和保存不同格式的图像文件 [`Image::load`] [`Image::save`]
///! - 调整图像的大小、旋转和裁剪 [`Image::resize`] [`Image::rotate`] [`Image::crop`]
///! - 应用不同的滤镜和效果 [`Filter`] [`Effect`]
///! 
///! 如果您想了解更多关于图像处理的原理和算法，您可以参考以下资源：
///! - [数字图像处理](https://book.xxx.com/subject/xxxx)，一本经典教科书，介绍图像处理的基本概念
///! - [Learn OpenCV](https://learnopencv.com)，一个网站，提供很多用 OpenCV 实现图像处理功能的教程和示例代码
///! - [Awsome Computer Vision](https://github.com/jbhuang0604/awesome-computer-vision)，一个仓库 

/// 一个表示图像的结构体
#[derive(Debug, Clone)]
pub struct Image {

}

impl Image {
    /// 从指定路径加载一个图像文件
    /// 
    /// 支持的格式有：PNG、JPEG、GIF、BMP 等
    /// 
    /// # 参数
    /// 
    /// - `path`: 图像文件的路径
    /// 
    /// # 返回值
    /// 
    /// 如果成功，返回一个 [`Image`] 实例；如果失败，返回一个 [`Error`]
    /// 
    /// # 示例
    /// 
    /// ```no_run
    /// use image::Image;
    /// 
    /// let img = Image::load("test.png")?;
    /// ```
    #[doc(alias = "读取")]
    #[doc(alias = "打开")]
    pub fn load<P: AsRef<Path>>(path: P) -> Result<Self, Error> {
        todo!()
    }
}
```
  
  
```rust
// 一个只在启用了 `foo` 特性时才可用的结构体
#[cfg(feature = "foo")]
#[doc(cfg(feature = "foo"))]
pub struct Foo;

impl Foo {
    // 一个只在启用了 `foo` 特性时才可用的方法
    #[cfg(feature = "foo")]
    #[doc(cfg(feature = "foo"))]
    pub fn bar(&self) {
        // ...
    }

}
```
  
  
## 4.3 类型系统
* 类型系统可以确保：
  * 1 接口显而易见
  * 2 自我描述性
  * 3 难以被误用

### 4.3.1 语义化类型
* 即有些值是有超过其表面意义的（不仅仅适用于基本类型）
  * 比如 `0 和 1` 它可能代表`男和女`


```rust
/*
    这里参数是 3 个 bool，用户可能会把其含义给记混了
*/
fn process_data(dryRun: bool, overwrite: bool, validate: bool) {
    // Code...
}

enum DryRun {
    Yes,
    No,
}

enum Overwriite {
    Yes,
    No,
}

enum Validate {
    Yes,
    No,
}
/*
    可以将 bool 定义为枚举，这样更有语义化
    用户在调用时候就不容易出错
*/
fn process_data2(dryRun: DryRun, overwrite: Overwriite, validate: Validate) {
    // Code...

}

fn main() {
    process_data2(DryRun::No, Overwriite::Yes, Validate::No);
}
```

### 4.3.2 有时可以使用 `零大小的类型` 来表示关于类型实例的特定事实
* 例子: 比如有个火箭的类型，有个发射的方法，在发射前调用没有问题，但如果发射了再调用就会有问题，那怎么解决呢？

```rust
struct Grounded;

struct Launched;

enum Color {
    White,
    Black,
}

struct Kilograms(u32);

// 这里泛型不用 T，用 Stage 表示
// 这里表示 只有 Stage 在 Grounded 时才能创建 🚀
struct Rocket<Stage = Grounded> {
    /*
        PhantomData 在没编译完时候就相当于里边的 Stage
        而编译完就没有了，就相当于一个单元类型
        它的作用就是在不同的条件下限制这个火箭类型的行为
     */
    stage: std::marker::PhantomData<Stage>,
}

impl Default for Rocket<Grounded> {
    fn default() -> Self {
        Self { 
            stage: Default::default(),
        }
    }
}

// 这里表示只有 Stage 在 Grounded 时才能发射 🚀

impl Rocket<Grounded> {
    pub fn launch(self) -> Rocket<Launched> {
        Rocket { stage: Default::default() }
    }
}

// 这里表示只有 🚀 发射后，才能调用加速、减速

impl Rocket<Launched> {
    pub fn accelerate(&mut self) {}
    pub fn decelerate(&mut self) {}
}

// 这些方法在任何阶段都可以调用
impl<Stage> Rocket<Stage> {
    
    pub fn color(&self) -> Color {
        Color::White
    }

    pub fn weight(&self) -> Kilograms {
        Kilograms(0)
    }

}
```

### 4.3.3 `#[must_use]` 注解（即必须使用函数的返回值）
* 将这个注解添加到类型、`Trait` 或函数后，如果用户的代码接收到该类型或 `Trait` 的元素，或调用了该函数，但没有明确的处理它，那么编译器就会发生警告

```rust
use std::error::Error;


/*
    使用 #[must_use] 注解，表示必须使用 process_data 函数的返回值
    如果没有使用其返回值，编译器就会发生警告
    这有助于提醒用户在处理潜在的错误情况时要小心，并减少可能的错误
*/
#[must_use]
fn process_data(data: Data) -> Result<(), Error> {

    Ok(())
}
```


# 5 受约束（Constrained）

## 5.1 即接口更改时要三思
* 如果你的接口要做出用户可见的更改，就一定要三思而后行
  * 要确保你做出的变化：
    * 不会破坏用户现有的代码
    * 而且这次更改变化应该保留一段时间，不应该频繁的变化
  * `频繁的`向后不兼容的更改（主版本增加），会引起用户的不满
    
## 5.2 向后不兼容的更改
* 有些向后不兼容的更改：
  * 是显而易见的（如你改变了公共的名称，或为它添加一个新的公共方法）
  * 而有些是很微妙的（这与 Rust 的工作方式相关）
  
* 这里主要介绍微妙棘手的更改，以及如何为其制定计划，这时就需要在接口的灵活性上做出权衡、妥协

### 5.2.1 对类型进行修改
* 如果你移除或重命名一个公共类型，那么几乎肯定会破坏用户的代码
  * 解决方案：尽可能利用可见性修饰符来解决
    * 例如：`pub(crate) 即对当前 crate 可见`、`pub(in path) 即对某个路径可见` ...

#### 例子1
    
```rust
pub mod outer_mod {
    pub mod inner_mod {
        // This function is visible within `outer_mod`
        // 只对 mod outer_mod 可见
        pub(in crate::outer_mod) fn outer_mod_visible_fn() {}

        // This function is visible to the entire crate
        // 整个 crate 都可见
        pub(crate) fn crate_visible_fn() {}

        // This function is visible within `outer_mod`
        // super is outer_mod
        pub(super) fn super_mod_visible_fn() {
            // This function is visible since we're in the same `mod`
            inner_mod_visible_fn();
        }

        // This function is visible only within `inner_mod`,
        // which is the same as leaving it private.
        // 即当前 inner_mod 可见，就类似于是私有的
        pub(self) fn inner_mod_visible_fn() {}
    }

    pub fn foo() {
        inner_mod::outer_mod_visible_fn();
        inner_mod::crate_visible_fn();
        inner_mod::super_mod_visible_fn();

        // Error! inner_mod_visible_fn is private
        // inner_mod::inner_mod_visible_fn();
    }
}


fn bar() {
    outer_mod::inner_mod::crate_visible_fn();

    // Error! super_mod_visible_fn is private
    outer_mod::inner_mod::super_mod_visible_fn();

    // Error! outer_mod_visible_fn is private
    outer_mod::inner_mod::outer_mod_visible_fn();

    outer_mod::foo(); 
}


fn main() {
    bar();
}
```
    
* 如果你的接口里公共类型越少，那么在更改时就越自由（即保证不会破坏现有代码）

#### 例子2：用户的代码不仅仅通过名称依赖于你的类型

```rust
// lib.rs
pub struct Unit {
    field: bool,
}



// main.rs
fn is_true(u: constrained_04::Unit) -> bool {
    matches!(u, constrained_04::Unit { field: true })
}

fn main() {
    // 引用 lib.rs 里的 Unit
    // 用户原来代码，Unit 中没有任何字段
    // Unit 添加 field 字段后，这里就报错了
    // 无论 field 是 pub 还是 private 的都会报错
    let u = constrained_04::Unit;
}
```

* 针对例子 2 的问题，Rust 提供了 `#[non_exhaustive]` 注解来缓解这些问题
  * `non_exhaustive` 表示类型或枚举在将来可能会添加更多字段或变体
    * 它可以应用于 `struct、enum、enum variants（枚举变体）`
  * 如果你在你的 `crate` 中使用 `non_exhaustive` 定义了某个类型，那么在其他 `crate` 中使用你定义的类型，编译器会禁止一些事情：
    * 禁止隐式构造：`lib::Unit { field: true }`
    * 禁止非穷尽模式的匹配（即没有尾随 `,` 和 `..` 的模式）
    * 例子 3
    
```rust

// lib.rs
#[non_exhaustive]
pub struct Config {
    pub window_width: u16,
    pub window_height: u16,
}


fn some_function() {
    let config = Config {
        window_width: 640,
        window_height: 480,
    };

    // non_exhaustive struct 可以使用这种详尽的方式创建
    if let Config { 
        window_width, 
        window_height
    } = config {
        // ....
    }

}



// main.rs
use constrained_04::Config;
fn main() {
    // 在 非 non_exhaustive 定义的 crate，这样创建就不行了
    // Error! 
    let config = Config {
        window_width: 640,
        window_height: 480,
    };

    // 在 非 non_exhaustive 定义的 crate，这样创建就不行了，Error! 
    if let Config { 
        window_width, 
        window_height,
        ..       // Note：这里必须加上 `..` 表示忽略其他的字段才行，但上边的构造方式就不被允许，
                 // Note：加上 `..` 后，work
    } = config {
        // ....
    }
}
```

* 如果你的接口比较稳定的话，就应该尽量避免使用该注解


### 5.2.2 `Trait` 实现

* Rust 中的一致性规则禁止把某个 `Trait` 为某个类型进行多重实现
* 因为下边几种情况会引起破坏性的变更
  * 1 为现有 `Trait` 添加 `Blanket Implementation`，这种变更通常是破坏性变更
    * 简单的说就是那种泛型形式的实现，类似这样 `impl <T> Foo for T`
  * 2 为现有类型实现外部 `Trait` 或为外部类型实现现有 `Trait`
  * 3 移除 `Trait` 的实现
    * 但为新类型实现 `Trait` 就不是问题，就不是破坏性变更

> 为现有类型实现任何 `Trait` 都要小心，下面看几个例子
{: .prompt-info }


* case1: add impl Foo1 for Unit in this crate

```rust
// lib.rs
pub struct Unit;

pub trait Foo1 {
    fn foo(&self);
}

// case1: add impl Foo1 for Unit in this crate
impl Foo1 for Unit {
    fn foo(&self) {
        println!("foo1 is called");
    }
}


// main.rs
use constrained_04::{Foo1, Unit};

trait Foo2 {
    fn foo(&self);
}

impl Foo2 for Unit {
    fn foo(&self) {
        println!("foo2 is called");
    }
}

fn main() {
    /*
        Error! 由于 lib.rs 中实现了 Foo1 中有 foo 方法，
        而上边又对 Foo2 进行了实现，
        重名了所以这里就报错了
     */
    Unit.foo();

}
```

* case2: 

```rust
// lib.rs

pub struct Unit;

pub trait Foo1 {
    fn foo(&self);
}

// case2: add a new public trait
pub trait Bar1 {
    fn foo(&self);
}

impl Bar1 for Unit {
    fn foo(&self) {
        println!("bar1");
    }
}

 

// main.rs

// use constrained_04::Unit;

use constrained_04::*;

// case1 & case2

trait Foo2 {
    fn foo(&self);
}

impl Foo2 for Unit {
    fn foo(&self) {
        println!("foo2 is called");
    }
}

fn main() {
    /*
        case 2: 如果只 use Unit 下边代码不会报错
        如果 use constrained_04::*，则由于 lib.rs 中实现了 Bar1 中有 foo 方法，
        重名了所以也会报错
     */
    Unit.foo();

}
```

#### 大多数到现有 `Trait` 的更改也是破坏性更改
* 例如改变方法签名
* 或添加新的方法
  * 但添加新方法时，有默认实现倒数可以的，这就不算破坏性更改
  
  
#### 上边说的都是通常情况下，都算破坏性更改，因为有这样一个方式：
* 封闭 `Trait`（Sealed Trait）
  * 即这种 `Trait` 只能被其他 `crate` 使用，而不能在其他 `crate` 中实现
  * 它的作用就是防止 `Trait` 在添加新方法时，造成破坏性的变更
  * 它不是内建的功能，有多种实现方式


<!--![image](/assets/images/rust/web_server/teacher_aim.png)-->
