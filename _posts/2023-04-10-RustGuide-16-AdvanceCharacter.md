---
layout: post
title: RustGuide-16-AdvanceCharacter
date: 2023-04-10 16:45:30.000000000 +09:00
categories: [Rust, Rust Getting Start]
tags: [Rust, Rust Getting Start]
---

# 19 高级特性

## Unsafe Rust（不安全 Rust）

### 匹配命名变量
* 在 Rust 中隐藏着第二个语音，它没有强制内存安全保证，它就是 `Unsafe Rust`
* `Unsafe Rust` 与普通的 Rust 一样，但提供了一些 “超能力”
* `Unsafe Rust` 存在是因为以下几个原因
1. 静态分析是保守的（即出于安全考虑会错误判断一些情况为不合法），而使用 `Unsafe Rust`，即告诉编译器我们知道自己在做什么，并承担相应风险
2. 计算机硬件本身就是不安全的，Rust 需要能够进行底层的系统编程，如果不允许 `Unsafe Rust` 那么这些工作就无法完成了

### Unsafe 超能力
* 可以使用 `unsafe` 关键字来切换到 `Unsafe Rust` 模式，它会后边跟着一个代码块，在这个代码块里写的代码就是 `unsafe` 代码
* `Unsafe Rust` 里可执行 4 个动作（超能力）
1. 解引用原始指针
2. 调用 `unsafe` 函数或方法
3. 访问或修改可变的静态变量
4. 实现 `unsafe trait`

> `unsafe` 并没有关闭借用检查或停用其他的安全检查措施，如果你在 `unsafe` 里使用引用，那么这个引用依然会被检查，`unsafe` 仅仅是让你可以访问上述 4 个特性，所以即便是 `unsafe` 中你依然会获得一定的安全性，通过把上边 4 种不安全操作约束在 `unsafe` 代码块中，你就要知道任何内存安全相关的错误必须留在 `unsafe` 块里，尽可能隔离 `unsafe` 代码，最后将其封装在安全的抽象里，提供安全的 API。实际上某些标准库就使用了 `unsafe` 代码，但他们在这之上又提供了安全的抽象接口，因为使用安全的抽象是安全的。
{: .prompt-info }


### 1 解引用原始指针
* 在 `Unsafe Rust` 里有两种类似于引用的新型指针，它们叫原始指针（或叫裸指针英文是 Raw Pointer）
1. 与引用类似，它要么是可变的，要么是不可变的
2. 可变的，语法是 `*mut T`
3. 不可变的，语法是 `*const T`，意味着指针在解引用之后不能直接对其进行赋值

> 这里 `*` 不是解引用符号，它是类型名的一部分，表示这是一个指针
{: .prompt-info }


#### 原始指针与引用的区别
1. 原始指针可忽略借用规则，即允许通过同时具有不可变和可变指针或多个指向同一位置的可变指针
2. 原始指针无法保证能指向合理的内存，而引用总是指向合理的内存
3. 原始指针允许为 `null`
4. 原始指针不实现任何自动清理

* 在放弃这些安全的保证后，就可以换取更好的性能以及与其他语言或硬件接口的能力了

```rust
fn main() {
    let mut num = 5;

    /*
        在 unsafe 代码块之外创建原始指针
        但只能在 unsafe 代码块中对原始指针解引用
    */
    // 将不可变引用转为不可变原始指针，这是一个有效的原始指针
    let r1 = &num as *const i32;

    // 将可变引用转为可变原始指针，这是一个有效的原始指针
    let r2 = &mut num as *mut i32;

    // 对原始指针解引用
    unsafe {
        println!("r1: {}", *r1);
        println!("r2: {}", *r2);   

        // 可以用 r2 修改其值，这需要多加小心
        *r2 = 3; 
        println!("r2: change to {}", *r2);   
    }

    /*
        创建一个无法知道其有效性的原始指针
        原始指针不一定一直有效

        这个内存地址可能有数据，也可能没有数据，这样依然可以创建一个原始指针
        address 可能是无效的
     */
    let address = 0x1245usize;
    let r = address as *const i32;
    // 对原始指针解引用
    unsafe {
        println!("r: {}", *r);
    }

}
```

#### 使用原始指针的原因
1. 与 C 语言进行接口交互
2. 构建借用检查器无法理解的安全抽象


### 2 调用 unsafe 函数或方法
* `unsafe` 函数或方法，即在它们前边加上了 `unsafe` 关键字的函数或方法
* 调用它们前需要满足一些条件（主要靠看文档），因为 Rust 无法对这些条件进行验证
* 调用 `unsafe` 函数或方法，必须在 `unsafe` 块里调用
* `unsafe` 函数或方法也属于 `unsafe` 块

```rust

unsafe fn dangerous() {
    println!("I am dangerous !");
}

fn main() {

    // 调用 Unsafe 函数需要再 Unsafe 块里
    unsafe {
        dangerous();
    }

}
```

#### 创建 unsafe 代码的安全抽象
* 函数包含 `unsafe` 代码并不意味着需要将整个函数标记为 `unsafe`
* 将 `unsafe` 代码包裹在安全函数中是一种常见的抽象

下边代码让我们看看标准库提供 `split_at_mut` 函数是如何实现的

```rust
/*
    简单定义 split_at_mut 看看其实现的原理
 */
// fn split_at_mut_demo(slice: &mut [i32], mid: usize) 
// -> (&mut [i32], &mut [i32]) {
//     let len = slice.len();

//     assert!(mid <= len);
//     // 这里报错，因为违反了借用规则, 但这里其实是切片的两个部分没有交集，其实是安全的
//     // 修正需要使用 unsafe code
//     (&mut slice[..mid], &mut slice[mid..])
// }


/*
    自己实现 split_at_mut 
    
    使用 unsafe 代码修正 split_at_mut_demo
    split_at_mut 里使用了 unsafe 代码，但是其函数
    本身没有标记为 unsafe 所以 split_at_mut 就是一个
    unsafe 安全抽象，我们就可以从安全的代码中对其进行调用
 */
fn split_at_mut(slice: &mut [i32], mid: usize) 
-> (&mut [i32], &mut [i32]) {
    let len = slice.len();
    assert!(mid <= len);
    /*
        这里返回一个原始指针，它的类型是 *mut i32
        它指向这个切片
     */
    let ptr = slice.as_mut_ptr();

    unsafe {
        /*
            from_raw_parts_mut 接收一个原始指针和一个长度来创建切片
            ptr.add(mid) 获得以 mid 为偏移量的原始指针
         */
        (slice::from_raw_parts_mut(ptr, mid),
        slice::from_raw_parts_mut(ptr.add(mid), len - mid)
        )
    }

}

fn main() {
    let mut v = vec![1, 2, 3, 4, 5, 6];
    // r 是 v 的完整的 slice
    let r = &mut v[..];
    /*
        标准库提供 split_at_mut 函数，
        将参数 index 为界限，
        分隔成两个切片
        split_at_mut 实际是使用了不安全的代码
     */
    let (a, b) = split_at_mut(r, 3);
    //let (a, b) = r.split_at_mut(3);

    assert_eq!(a, &mut [1, 2, 3]);
    assert_eq!(b, &mut [4, 5, 6]); 
}
```

* 这样可能 crash 因为无法保证拥有 10000 元素的切片是有效的

```rust
fn main() {
    println!("test_address");
    let address = 0x12345usize;
    let r = address as *mut i32;
    // 这样可能 crash 因为无法保证拥有 10000 元素的切片是有效的
    let slice: &[i32] = unsafe {
        slice::from_raw_parts_mut(r, 10000)
    }; 
}
```

### 使用 extern 函数调用外部代码
* `extern` 关键字可以简化创建和使用外部函数接口（FFI）的过程
* 外部函数接口（FFI，Foreign Function Interface），它允许一种编程语言定义函数，并让其他的编程语言能调用这些函数

> 任何在 `extern` 块里声明的函数都是不安全的，因为其他语言并不会遵守 Rust 的规则，而 Rust 又无法对他们进行检查
{: .prompt-info }

所以在调用外部函数时的安全问题落在了开发者身上


```rust
/*
    extern "C" 块内的函数签名就是我们想调用的外部函数的签名
    而 "C" 指明了外部函数使用的是应用程序二进制接口（ABI），
    ABI 是用于定义在汇编层面的调用方式的
 */ 
extern "C" {
    fn abs(input:i32) -> i32;
}

fn main() {
    unsafe {
        println!("Absolute value of -3 according to C: {}", abs(-3));
    }
}
```

* ABI 是用于定义在汇编层面的调用方式的
* "C" ABI 是最常见的，它遵循 C 语言的 ABI

### 从其他语言来调用 Rust 函数
* 可以使用 `extern` 关键字来创建接口，其他语言通过它们可以调用 Rust 函数
* 在 `fn` 前边添加 `extern` 关键字，并指定对应的 ABI。并且还需要添加 `#[no_mangle]` 注解，避免 Rust 在编译时改变它的名称
1. `mangle` 实际对应编译的一个阶段，在这个阶段编译器会修改函数的名称从而让其包含更多可用于后续编译的信息，这些改变后的名称通常是难以阅读的

```rust
// 这个函数在编译链接后就可以被 C 语言访问了
// 这类的 extern 功能不需要使用 unsafe

#[no_mangle]
pub extern "C" fn call_from_c() {
    println!("Just call a rust function from C!");
}
```

### 3. 访问或修改一个可变的静态变量
* Rust 支持全局变量，但因为所有权机制可能产生某些问题，例如数据竞争
* 在 Rust 中全局变量叫静态（static）变量


```rust
/*
    命名规范大写，声明时需要标识类型
    静态变量只能存储拥有 'static 这个生命周期的引用
    这就意味着 Rust 可以自己推断出其生命周期
    所以就无需手动标注了

 */
static HELLO_WORLD: &str = "Hello, world!";
fn main() {
    println!("name is {}", HELLO_WORLD);
}
```

#### 静态变量
* 静态变量与常量类似
* 命名规范是大写的 SCREAMING_SNAKE_CASE
* 声明时候必须标注类型
* 静态变量只能存储 `'static` 生命周期的引用，无需显示标注，因为可以推断出来
* 访问不可变的静态变量是安全的

#### 常量和不可变静态变量的区别
* 静态变量：有固定的内存地址，使用它的值总会访问同样的数据
* 常量：允许使用它们的时候对数据进行复制
* 静态变量：是可以改变的，`访问`和`修改`可变的静态变量`是不安全（unsafe）的操作`

```rust
static mut COUNTER: u32 = 0;

fn add_to_count(inc: u32) {
    // 访问和修改可变的静态变量是不安全（unsafe）的操作
    // 所以这里放到 unsafe 块里进行操作
    unsafe {
        COUNTER += inc;
    }
}

fn main() {
    add_to_count(3);
    
    // 这里是单线程，但多线程其实无法保证数据竞争不发生
    unsafe {
        println!("COUNTER: {}", COUNTER);
    }
}
```

### 4. 实现 unsafe trait
* 当某个 `trait` 中存在至少一个方法拥有编译器无法校验的不安全因素时，就称这个 `trait` 是不安全的
* 声明不安全的 trait：即在 `trait` 前加 `unsafe` 关键字
1. 该 `trait` 只能在 `unsafe` 代码块中实现

```rust
unsafe trait Foo {
    // methods go here
}

// 在 unsafe 代码块中实现
unsafe impl Foo for i32 {
    // method implementations go here
}
```

### 何时使用 `unsafe` 代码
* 编译器无法保证内存安全，让开发者保证 `unsafe` 代码正确并不是一个简单活儿
* 但又充足理由使用 `unsafe` 代码时，就可以使用
* 通过显示标记的 `unsafe`，可以在出现问题时轻松定位


## 高级 Trait

### 在 `trait` 的定义中使用关联类型来指定占位类型
* 关联类型（associated type）是 `trait` 中类型的占位符，它可以用在 `trait` 的方法签名中
1. 即可以定义出包含某些类型的 `trait`，而在实现前无需知道这些类型具体是什么
2. swift 也有这个特性 

* 标准库的 Iterator 就使用了 associated type
```rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```

### 关联类型与泛型的区别
* 泛型
1. 每次实现 `trait` 时必须标注具体的类型
2. 可以为一个类型多次实现某个 `trait`（即使用不同的泛型参数）

```rust
pub trait Iterator2<T> {
    fn next(&mut self) -> Option<T>;
}

struct Counter {

}

//多次实现某个 Iterator2 trait for String
impl Iterator2<String> for Counter {
    fn next(&mut self) -> Option<String> {
        None
    }
}

//多次实现某个 Iterator2 trait for u32
impl Iterator2<u32> for Counter {
    fn next(&mut self) -> Option<u32> {
        None
    }
}
```

* 关联类型
1. 在使用 `trait` 时，无需标注类型，但需要在里边指明关联的类型
2. 无法为单个类型多次实现某个 `trait`


```rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}

struct Counter {

}

impl Iterator for Counter {
    type Item = u32;
    fn next(&mut self) -> Option<Self::Item> {
        None
    }
}
```

> 这个 `Iterator trait` 只能为 `Counter` 类型实现一次，多次会报错
{: .prompt-info }


### 默认泛型参数和运算符重载
* 可以在使用泛型参数时为泛型指定一个默认的具体类型
* 语法：`<PlaceholderType=ConcreteType>`，ConcreteType 就是默认的类型
* 这种技术常用于运算符重载（operator overloading）

> 虽然 Rust 不允许创建自己的运算符以及重载任意的运算符。但可以通过实现 `std::ops` 中列出的那些 trait 来重载一部分相应的运算符
{: .prompt-info }


这里重载了 `+` 运算符
```rust
use std::ops::Add;

#[derive(Debug, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

impl Add for Point {
    type Output = Point;

    fn add(self, rhs: Self) -> Self::Output {
        Point {
            x: self.x + rhs.x,
            y: self.y + rhs.y,
        }
    }
}

fn main() {

    assert_eq!(Point { x: 1, y: 0} + Point { x: 2, y: 3},
               Point {x: 3,y: 3});
    println!("point add");
}
```

* `pub trait Add<Rhs = Self>`，Add 这里就使用了默认的泛型参数类型，即如果在使用 trait 时没有指定默认泛型参数类型则使用 Self 作为默认类型，
所以上边代码中 Self 就是 Point


#### 自己指定默认泛型参数类型

* 为 Millimeters 实现 Add trait，让毫米与米可以相加

```rust
use std::ops::Add;

struct Millimeters(u32);
struct Meters(u32);

// 这里指定默认的泛型参数为 Meters
impl Add<Meters> for Millimeters {
    type Output = Millimeters;

    fn add(self, rhs: Meters) -> Self::Output {
        Millimeters(self.0 + (rhs.0 * 1000)) 
    }
}
```

#### 默认泛型参数的应用场景
* 扩展一个类型而不破坏现有的代码
* 允许在大部分用户都不需要的特定场景下进行自定义
1. 上边 Millimeters 和 Meters 的例子就属于自定义


### 完全限定语法（Fully Qualified Syntax）如何调用同名方法

```rust
trait Pilot {
    fn fly(&self);
}

trait Wizard {
    fn fly(&self);
}

struct Human;

impl Pilot for Human {
    fn fly(&self) {
        println!("This is your captain speaking.");
    }
}

impl Wizard for Human {
    fn fly(&self) {
        println!("Up!");
    }
}

impl Human {
    fn fly(&self) {
        println!("Human's fly");
    }
}

fn main() {
    let person = Human;
    person.fly(); // 这样是调用 Human 本身的 fly 方法，即打印 Human's fly

    // 调用 Human 的 Pilot 实现
    Pilot::fly(&person);
    
    Wizard::fly(&person);
}
```

* 上边代码比较简单调用没有问题

```rust
trait Animal {
    fn baby_name() -> String;
}

struct Dog;

impl Dog {
    fn baby_name() -> String {
        String::from("Spot")
    }
}

impl Animal for Dog {
    fn baby_name() -> String {
        String::from("puppy")
    }
}

fn test_dog() {
    println!("A baby dog is called a {}", Dog::baby_name());

    // 这样就会报错，因为无法知道使用哪个类型的 Animal 实现
    println!("A baby dog is called a {}", Animal::baby_name());
}
```

* 上边代码调用 `Animal::baby_name()` 就会报错，因为无法知道使用哪个类型的 `Animal` 实现
* 这时可以使用完全限定语法了

#### 完全限定语法
* 格式：`<Type as Trait>::function(receiver_if_method, netx_arg, ...);`
1. `receiver_if_method` 是接收者
2. `netx_arg` 是函数参数

* 这种语法可以在任何调用函数或方法的地方使用
* 并且它允许忽略那些从其他上下文能推导出来的部分

> 只有当 Rust 无法区分你期望调用哪个具体实现的时候，才需要使用这种语法。因为这种语法写起来比较麻烦 😅
{: .prompt-info }


* 改成 `<Dog as Animal>::baby_name()` 就可以了

```rust
println!("A baby dog is called a {}", <Dog as Animal>::baby_name());
```

### 使用 `supertrait` 来要求 `trait` 附带其他 `trait` 的功能
* 就是一个 `trait` 继承另外一个 `trait`
* 有时需要在一个 `trait` 中使用其他 `trait` 的功能
1. 即需要那个间接被依赖的 `trait` 也被实现

> 那个被间接依赖的 `trait` 就是当前 `trait` 的 `supertrait`
{: .prompt-info }

```rust
use std::fmt;

// 类似于继承，即 OutlinePrint 依赖于 fmt:: Display
trait OutlinePrint: fmt:: Display {
    fn outline_print(&self) {
        let output = self.to_string();
        let len = output.len();
        println!("{}", "*".repeat(len + 4));
        println!("*{}*", " ".repeat(len + 2));
        println!("* {} *", output);
        println!("*{}*", " ".repeat(len + 2));
        println!("{}", "*".repeat(len + 4));
    }
}

struct Point {
    x: i32,
    y: i32,
}

// 因为 OutlinePrint 有默认实现，所以这里我们只实现 fmt::Display 就可以了
impl fmt::Display for Point  {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}
```

### 使用 `newtype` 模式在外部类型上实现外部 trait
* 孤儿规则：只有当 trait 或类型定义在本地的包内时，才能为该类型实现这个 trait
* 但是我们可以通过 `newtype` 模式来绕过孤儿规则
1. 即利用 tuple struct（元组结构体）创建一个新的类型


```rust
use std::fmt;
/*
    我们想为 Vec 实现 Display 这个 trait
    而 Vec 和 Display 都定义在外部的包中，
    所以我们无法直接为 Vec 实现 Display 这个 trait

    那么怎么做呢？
    我们可以把 Vec 包裹在 struct Wrapper 中
 */
// 包裹 Vec 以便实现 Display trait
struct Wrapper(Vec<String>);

impl fmt::Display for Wrapper {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        /*
            因为 Wrapper 是一个 tuple struct 可以通过索引将 Vec 取出
         */
        write!(f, "[{}]", self.0.join(", "))
    }
}
```

## 高级类型

### 1 使用 `newtype` 模式实现类型安全和抽象
* `newtype` 模式的作用：
1. 可用于静态的保证各种值之间不会混淆并表明值的单位
2. 为类型的某些细节提供抽象能力
3. 可通过轻量级的封装来隐藏内部实现细节

### 2 使用类型别名来创建类型同义词
* Rust 提供了类型别名的功能
1. 可以为现有类型生产另外的名称（同义词）
2. 这个类型的别名并不是一个独立的类型

* 使用 `type` 关键字创建类型别名
* 别名的主要用途就是减少代码的字符重复

```rust
use std::fmt;

// 类型别名 1
type Kilometers = i32;


// 类型别名 2
type Thunk = Box<dyn Fn() + Send + 'static>;

fn take_long_type(f: Thunk) {

}

fn return_long_type() -> Thunk {
    Box::new(|| println!("hi"))
}

// 类型别名 3

// 由于这个声明在 use std::io::Error 中所以这里需要注释
//type Result<T> = Result<T, std::io::Error>;

// 可以再别名一次，以便直接使用 Result<T>
type Result<T> = std::io::Result<T>;

pub trait Write {
    fn write(&mut self, buf: &[u8]) -> Result<usize>;

    fn flush(&mut self) -> Result<()>;

    fn write_all(&mut self, buf: &[u8]) -> Result<()>;
    fn write_fmt(&mut self, fmt: fmt::Arguments) -> Result<()>;

}

fn main() {
    let x: i32 = 5;
    let y: Kilometers = 5;
    println!("x + y = {}", x + y);

    //-----------
    let f: Thunk = Box::new(|| println!("hi"));
}
```

### 3 Never 类型
* 有一个名为 `!` 的特殊类型，表示函数永远不会返回值
1. 它没有任何值，行话称为空类型（empty type）

* 我们倾向于叫它 `Never` 类型，因为它在不返回值的函数中充当返回类型
* 不返回值的函数也被称作叫发散函数（diverging function）

* `Never` 类型可强制被转换为其他类型进行返回，例如下边代码的 `continue`

```rust
// 这个函数表示返回 Never 类型
// 这里报错，什么不写返回的是 `()`

// mismatched types
// expected type `!`
// found unit type `()`
fn bar() -> ! {

}

//------- 例子 1
fn main() {
    let guess = "";

    loop {
        let guess: u32 = match guess.trim().parse() {
            Ok(num) => num,    
            // 这里 continue 就是 Never 类型，而 Never 类型无法产生一个可供返回的值，
            // 所以这个 Err 分支的类型就采用了第一个分支的类型      
            Err(_) => continue, 
        };
    };

}
```

* Never 的表达式可以被强制转成其他的任意类型

```rust
// 例子 2 此代码报错，只用于理解 Never
impl <T> Option<T> {
    pub fn unwrap(self) -> T {
        match self {
            // 这个分支返回 T 类型
            Some(val) => val,
            /*
                而 panic! 的返回类型就是 Never
                panic 时会终止程序，不会返回任何值
                所以 None 时，不会返回值
                所以 unwrap 的代码就是合理的
             */
            None => panic!("called `Option::unwrap()` on a `None` value"),
        }
    }
}

// 例子 3
    loop {
        /*
            这个循环永远不会结束
            它的返回类型就是 !
            当然可以使用 break 指令终止循环
         */
        print!("and ever ");
    }
```

### 动态大小的类型
* Rust 需要在编译时确定为一个特定类型的值分配多少空间
* 动态大小类型（Dynamically Sized Types，DST）的概念
1. 它可以使我们在编写代码时使用只有在运行时才能确定大小的值


> `str` 就是动态大小的类型（注意`不是 &str`）：只有运行时才能确定字符串的长度
{: .prompt-info }


* 下列代码无法正常工作，因为 Rust 无法在编译时知晓其占多少内存
1. 而同一类型所有的值必须使用等量的值

```rust
let s1: str = "Hello there!";
let s2: str = "How's it going?";
```

* 可以使用 `&str（字符串切片）`解决上边的代码
1. `&str` 里存储的是 `str 的地址`以及 `str 的长度`
2. 它就存这俩所以它的内存大小肯定是固定的

### Rust 使用动态大小类型的通用方式
* 它们总会附带一些额外的元数据来存储动态信息的大小
1. 使用动态大小类型的时总会把它的值放在某种指针后边，就像 `str 与 &str`

### 另外一种动态大小的类型：即 `Trait`

> 其实每一个 `Trait` 都是一个动态大小的类型，可以通过名称对其进行引用
{: .prompt-info }

* 为了将 `trait` 用作 `trait` 对象，必须将它放置在某种指针之后
1. 例如 `dyn Trait` 或 `Box<dyn Trait>` (`Rc<dyn Trait>`) 之后

### Sized Trait
* 为了处理动态大小的类型，Rust 提供了一个 `Sized Trait` 来确定一个类型的大小在编译时是否已知
1. 在编译时可以计算出大小的类型会自动实现这一 trait
2. Rust 还会为每一个泛型函数隐式的添加 `Sized 约束`


```rust
fn generic<T>(t: T) {

}
// 上边会被隐式转化为下边这样
fn generic<T: Sized>(t: T) {

}
```

* 默认情况下，泛型函数只能被用于编译时已经知道大小的类型，可以通过特殊语法解除这一限制
1. `?Sized Trait` 约束
2. `?Sized` 表达了不确定性，就表示 `T` 可能是 `Sized` 的也可能不是 `Sized` 的

> 限制：`?Sized` 只能被用在 `Sized` 的上边，不能用于其他的 `Trait`
{: .prompt-info }


```rust
// 这时 T 会变成 &T，因为 T 可能不是 Sized 的，这时就需要放在某种指针后边
fn generic<T: ?Sized>(t: &T) {

}
```

## 高级函数和闭包

### 函数指针
* 函数可以传递给其他函数 （闭包也可以传递给函数，之前讲过）
* 在传递的过程中函数就会被强制转换成 `fn` 这个类型
* `fn` 就是所谓的函数指针（function pointer）

```rust
fn add_one(x: i32) -> i32 {
    x + 1
}

/*
    参数时 fn 即函数指针
    这个函数指针要求参数是 i32，返回值也是 i32
 */
fn do_twice(f: fn(i32) -> i32, arg: i32) -> i32 {
    f(arg) + f(arg)
}

fn test_fn() {
    let answer = do_twice(add_one, 5);
    println!("The answer is: {}", answer);
}
```

### 函数指针与闭包的区别
#### 闭包
1. 闭包就是实现了 3 种`闭包 Trait`

#### `fn` 函数指针
1. 它是一个类型，而不是一个 `Trait`
2. 我们可以直接指定 `fn` 为参数类型，而不用声明一个以 `Fn Trait` 为约束的泛型参数
3. 函数指针实现了 3 中`闭包 Trait（Fn、FnMut、FnOnce）` 的全部
* 所以总是可以把函数指针用作参数传递给一个接收闭包的函数
* 也正是这个原因，我们倾向于搭配闭包 `trait` 的泛型来编写函数：这样这个函数就可以同时接收闭包和普通函数作为它的参数了

#### 而在某些场景下，我们可能只想接收 `fn` 而不接收闭包，场景如下：
* 1 与外部不支持闭包的代码交互：C 函数

```rust
fn main() {
    let list_of_numbers = vec![1, 2, 3];
    let list_of_strings: Vec<String> = list_of_numbers
    .iter()
    /*
        这里使用闭包
        闭包的参数时 i，然后调用 i 的 to_string() 方法
     */
    .map(|i| i.to_string())
    .collect();
    println!("list_of_strings is {:#?}", list_of_strings);

    let list_of_numbers = vec![1, 2, 3];
    let list_of_strings: Vec<String> = list_of_numbers
    .iter()
    /*
        这里将 ToString::to_string 函数传入
     */
    .map(ToString::to_string)
    .collect();

    println!("list_of_strings is {:?}", list_of_strings);

    /*
        map 的参数需要实现 FnMut Trait 
        而闭包和函数指针 ToString::to_string 都实现了 FnMut Trait
     */
}
```

* 另一个列子

```rust
fn main() {
    enum Status {
        Value(u32),
        Stop,
    }
    /*
        Value(3) 与函数调用相似
        实际上这种构造器确实被实现成了函数
        这个函数会接收一个参数并返回新的实例

        所以我们可以把这种构造器也作为实现了闭包 Trait 的函数指针来使用

     */
    let v = Status::Value(3);

    let list_of_statuses: Vec<Status> = 
    (0u32..20)
    // 这里就是把构造器直接传入即可
    .map(Status::Value)
    .collect();
}
```

### 返回闭包
* 闭包使用 `Trait` 进行表达，我们无法在函数中直接返回一个闭包，但可以将一个实现了该 `Trait` 的具体类型作为返回值

```rust
// 这样想返回一个闭包时不可以的 
/*
    这样想返回一个闭包时不可以的，编译报错
    因为在编译时无法获取其已知的大小
 */
fn returns_closure() -> Fn(i32) -> i32 {
     |x| x + 1
}

// 解决方案
// 即将其放在某种指针的后边
/*
    解决方案
    即将其放在某种指针的后边
    这时它的返回类型就有固定的大小了，编译就不会出现问题了
 */
fn returns_closure() -> Box<dyn Fn(i32) -> i32> {
    Box::new(|x| x + 1)
}
```


## 宏 Macro
* 宏在 Rust 里指的是一组相关特性的集合称谓
1. 包括使用 `macro_rules!` 来构建声明宏（declarative macro）
2. 以及 3 种过程宏

### 3 种过程宏
1. 自定义的 `#[derive]` 宏，用于 `struct` 和 `enum`，可为其指定随 `derive` 属性添加的代码
2. 类似属性的宏，在任何条目上添加自定义属性
3. 类似函数的宏，看起来像函数调用，可以对其指定为参数的 token 进行操作

### 函数和宏的区别

* 宏
1. 本质上讲宏是用来编写可以生成其他代码的代码（即所谓的元编程，metaprogramming）
2. 编译器会在解释代码前把宏展开
3. 宏的定义要比函数的定义复杂的多，宏是难以阅读、理解、维护的
4. 在某个文件调用宏的时候，必须提前定义宏或将宏引入当前的作用域


* 函数
1. 函数在定义签名的时候，必须声明参数的个数和类型，而宏可以处理可变的参数
2. 而函数可以在任何位置定义并在任何位置使用

### `macro_rules!` 声明宏（可能会弃用）
* 或叫宏模版
* Rust 中最常见的宏形式：声明宏
1. 类似于 `match` 表达式的匹配模式
2. 在定义声明宏时，需要使用 `macro_rules!`


```rust
//let v: Vec<u32> = vec![1, 2, 3];

/*
    看一下 vec! 宏的简单实现

    #[macro_export] 表明这个宏在它所在的包被引入作用域后才能使用，
    缺少这个标注的宏就不能被引入作用域

    宏名称是 vec
 */
#[macro_export]
macro_rules! vec {
    /*
        这里开始就是宏的定义体
        这里类似于 match 的模式匹配（分支），而这里只有一个分支

        ( $($x:expr), *) 就是它的匹配模式
        => 后就是其对应的代码
        
        这个宏只存在一种有效的匹配模式，其他模式都会导致编译时的错误
        而某些比较复杂的宏他们就可能包含多个分支
        这与 match 表达式还是有本质区别的，match 表达式它匹配的是值
        而这里的宏匹配的是 Rust 的代码结构

        代码解释
        $x:expr 表示可以匹配任何的 Rust 表达式，然后把它命名为 $x
        , 意味着一个可能的逗号分隔符会出现在捕获代码的后边
        * 表示能够匹配 0 个或多个 * 之前的东西
        所以这个例子 ( $x:expr ), *) 就是匹配上边 1, 2, 3 这三个表达式  
     */
    ( $( $x:expr ), *) => {
        {
            let mut temp_vec = Vec::new();
            $(
                /*
                    每次匹配都会生成这样的代码
                    本例就是：
                    push(1)
                    push(2)
                    push(3)
                 */
                temp_vec.push($x);
            )*
            // 返回 Vec
            temp_vec 
        }
    };
}


这个宏最后代码类似于这样
// let mut temp_vec = Vec::new();
// temp_vec.push(1);
// temp_vec.push(2);
// temp_vec.push(3);
// temp_vec
```

> 大部分程序员只是用，一般不会编写宏
{: .prompt-info }


### 基于属性来生成代码的过程宏
* 是之后替代 `macro_rules!` 的宏

* 这种形式更像函数（某种形式的过程）
1. 接收并操作输入的 Rust 代码
2. 然后生成另外一些 Rust 代码作为结果

* 三种过程宏
1. 自定义派生
2. 属性宏
3. 函数宏

* 创建过程宏时
1. 宏定义必须单独放在它们自己的包中，并且使用特殊的包类型

```rust
use proc_macro;

// some_attribute 用来指定过程宏的占位符
#[some_attribute]
/*
    定义了过程宏的函数

    TokenStream 表示一段标记序列，在 proc_macro 中定义
    这也是过程宏的核心
    
    需要被宏处理的组成了参数 TokenStream
    结果 TokenStream 就是处理后的

    函数附带的属性就决定我们创建的是哪种过程宏
    同一个包中可以有多种不同的过程宏

 */
pub fn some_name(input: TokenStream) -> TokenStream {
    
}
```

### 1 自定义 derive（派生）宏
* 需求：创建一个 hello_macro 包，定义一个拥有关联函数 hello_macro 的 HelloMacro trait
* 我们需要提供一个能自动实现 trait 的过程宏
* 在它们的类型上标注 `#[derive(HelloMacro)]`，进而得到 hello_macro 的默认实现

```shell
$ mkdir derive-19
$ cd derive-19
$ vim Cargo.toml
$ cargo new hello_macro --lib
$ cargo new hello_macro_derive --lib
```

* workspace

```rust
[workspace]

members = [
    "hello_macro",
    "hello_macro_derive",
    "Pancakes",
]
```

* hello_macro 的 lib.rs

```rust
pub trait HelloMacro {
    fn hello_macro();
}
```

* hello_macro_derive 的 lib.rs

```rust
extern crate proc_macro;

// 
/*
    借助 proc_macro 提供的编译器接口从而在代码中读取和操作 Rust 代码
    而由于它已经被内置在 Rust 里了，所以不需要添加它到 Cargo.toml 中的依赖
 */
use crate::proc_macro::TokenStream;

/*
    将 syn 产生的数据结构重新转化为 Rust 代码
 */
use quote::quote;

/*
    syn 用于把 Rust 代码从字符串转化为可供我们操作的数据结构
 */
use syn;

/*
    当别人写 #[derive(HelloMacro)] 时，
    hello_macro_derive 函数就会自动被调用

    为什么会自动调用呢？就是因为 hello_macro_derive 上边
    标注了 #[proc_macro_derive(HelloMacro)]，我们指明了 HelloMacro trait

    而 hello_macro_derive 函数，首先会把 input 转化解释成我们可操作的数据结构


 */

#[proc_macro_derive(HelloMacro)]
pub fn hello_macro_derive(input: TokenStream) -> TokenStream {
    // 本函数负责解析 TokenStream
    // 这个函数签名对于你创建其他过程宏其实基本是一样的
    // 不同的是下边的 impl_hello_macro 

    // Construct a representation of Rust code as a syntax tree
    // that we can manipulate
    // parse 函数接受 TokenStream 并返回 DeriveInput 结构体
    // DeriveInput 代表了解析后的 Rust 代码
    let ast = syn::parse(input).unwrap();

    // Build the trait implementation
    // 负责转化语法树，即最后生成 Rust 代码的地方
    impl_hello_macro(&ast)
}

fn impl_hello_macro(ast: &syn::DeriveInput) -> TokenStream {
    // Ident 结构体，这里有被标注类型的名称
    let name = &ast.ident;
    /*
        quote! 宏，定义返回的那些 Rust 代码
        就返回 impl 的代码

        quote! 宏的结果是编译器无法直接理解的类型，
        所以需要把结果转化为 TokenStream
        通过调用 into() 方法转化为 TokenStream

        #name 会被替换为变量 name 的值
     */
    /*
        stringify! 宏，它是 Rust 内置的，
        它接收一个表达式，例如 1+2，并把其抓为字符串字面值 ("1+2")
     */
    let gen = quote! {
        impl HelloMacro for #name {
            fn hello_macro() {
                println!("Hello, Macro! My name is {}", stringify!(#name));
            }
        }
    };
    gen.into()

}
```

* Pancakes 使用 HelloMacro 宏的结构体

```rust
use hello_macro::HelloMacro;
use hello_macro_derive::HelloMacro;

/*
    效果就是用户在自己的 struct 上 derive(HelloMacro)
    让自己的类型能获得 HelloMacro 的默认实现
 */
#[derive(HelloMacro)]
struct Pancakes;

fn main() {
    Pancakes::hello_macro();
}
```

### 2 类型属性宏
* 属性宏与自定义 `derive` 宏类似
1. 允许创建新的属性
2. 但不是为 `derive` 属性生成代码


* 属性宏更加灵活
1. `derive` 宏只能用于 `struct` 和 `enum`
2. 而属性宏可以用于任意条目，例如函数

* 属性宏的工作方式与自定义 `derive` 宏几乎一样，都需要建立一个 `proc_macro_attribute` 的包，并提供相应代码的函数


```rust
/*
    route 是用于做路由的
    这里如果方法是 GET，就会走到下边的 index 函数中
    而 route 就是过程宏定义的
 */
#[route(GET, "/")]
fn index() {

}

/*
    这就是 route 宏的函数签名
    attr 就对应 route(GET, "/") 的 GET 和路径 "/"
    item 就对应函数体，这里就是 index 函数
 */
#[proc_macro_attribute]
pub fn route(attr: TokenStream, item: TokenStream) -> TokenStream {

}
```

### 3 函数宏
* 函数宏定义类似于函数调用的宏，但比普通函数更加灵活
* 函数宏可以接收 TokenStream 作为参数
* 与另外两种过程宏一样，在定义中使用 Rust 代码来操作 TokenStream
* 工作方式与自定义 derive 宏类似，也需要建立一个 proc_macro 的包，并提供相应代码的函数

```rust
/*
    定义一个解析 SQL 语句的宏
 */
let sql = sql!(SELECT * FROM posts WHERE id=1);

#[proc_macro]
pub fn sql(input: TokenStream) -> TokenStream {

}
```

