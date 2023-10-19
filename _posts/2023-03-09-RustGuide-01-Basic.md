---
layout: post
title: RustGuide-01-Basic
date: 2023-03-09 16:45:30.000000000 +09:00
categories: [Rust, Rust Getting Start]
tags: [Rust, Rust Getting Start]
---

# 相关命令

* 查看版本，例如 rustc 1.68.0 (2c8cc3432 2023-03-06)
1. 2c8cc3432 commit hash
2. 1.68.0 版本

```
$ rustc --version
```

* 更新
```
$ rustup update
```

* 卸载
```
$ rustup self uninstall
```

* 本地文档
```
$ rustup doc
```

# Hello World

* rust 文件后缀名称 .rs
* 命名规范：单词用下划线分隔，例如 hello_word.rs
```rust
// main 函数式 Rust 程序入口
// Rust 缩进是 4 个空格，而不是 Tab
fn main() {
    // println! 是一个宏，函数没有 !
    println!("hello world!");
}
```

* 运行 Rust 程序前必须先编译，使用 rustc 命令编译
```
$ rustc main.rs
```
* 编译成功后生成二进制文件，rustc 只适合简单程序

# Cargo
* Cargo 是 Rust 的构建系统和包管理工具，用于构建代码，下载依赖库
* 查看 Cargo 版本
```
$ cargo --version
```
* 使用 Cargo 创建新项目，cargo new `项目名`
```
$ cargo new hello_world
```
* 生成新项目目录结构
1. Cargo.toml 配置文件
2. src 文件夹，用于放源代码
3. 顶层目录可以放 README、许可信息、配置文件、其他跟源代码无关的文件
4. 如果没有使用 Cargo 创建项目，也可手动变成 Cargo 形式，即把源代码放到 src 目录下，并添加 Cargo.toml 配置文件

* Toml（Tome‘s Obvious，Minimal Language 格式）文件



```
[package] // 表示区域标题，下方内容是来配置 package 的
name = "hello_world"  // 项目名
version = "0.1.0"     // 项目版本
edition = "2021"      // 使用的 Rust 版本

[dependencies] // 项目的依赖项
```

> 在 Rust 里，代码的包称为 crate
{: .prompt-info }


## Build Cargo 项目
```
$ cargo build
```
生成文件路径：/target/debug/{项目名}

> 第一次 cargo build 会在顶层目录生成 cargo.lock，cargo.lock 负责追踪项目依赖的精确版本（类似 Podfile.lock）
{: .prompt-info }

## Run Cargo 项目
编译 + 运行项目
```
$ cargo run
```
* 如果之前编译成功过，并且源代码没有改动，那就直接运行程序

## Check Cargo 项目
* 用于检查代码，确保编译能通过，但不产生任何可执行文件
```
$ cargo check
```

> cargo check 比 cargo build 快的多，所以开发时可以用 cargo check，到生成二进制文件时候再用 cargo build
{: .prompt-info }

## 为发布构建
* 编译时会进行优化，代码会运行更快，但是编译时间更长
* 可执行文件生成在 target/release 下
```
$ cargo build --release
```



# 猜数游戏


## 变量

* rust 默认变量都是 immutable 不可变的
* 在变量前加 mut 表示变量是可变的 mutable

```rust
let foo = 1;
// foo = 2; // error can't assign twice to immutable variable 

let mut _bar = foo; // rust 默认变量都是 immutable 不可变的
_bar = 3;  // 在变量前加 mut 表示变量是可变的 mutable
```



```rust
// 引用标准库 std 的 io 库 可以使用 io
/*
rust 默认把 prelude（预导入） 导入到程序的每个作用域中,如果你使用的类型 prelude 里没有
这时你需要自行导入, 就是使用 use 导入库
 */
use std::io;

// fn 即声明一个函数
fn main() {
    println!("猜数!");
    println!("猜一个数");

    // rust 中 String 使用 utf-8 编码
    // :: 两个冒号表示后边的是一个静态方法
    // 这里 new 函数创建一个空白的字符串
    let mut guess = String::new();

    // 1 读取一行东西，并放到 guess 字符串引用里
    // 2 expect 为抛出异常时候输出的错误信息
    // 3 & 表示参数是一个引用
    // 4 read_line 参数是需要一个引用
    // 5 引用默认也是不可变的比如 &guess, 所以 &mut 就是可变引用
    // 
    /* 
        6   
       read_line 返回 Result 类型，Result 就是枚举类型
       io::Result 有两个值 OK 和 Err
       如果是 OK 表示操作成功，并且里边有结果
       如果是 Err 表示操作失败，Err 里有失败 reason
       expect 是 Result 的方法，如果返回 Err 则 expect 会中断程序，将传入字符串显示
       如果返回 OK，expect 就将用户输入的值返回
     */
    io::stdin().read_line(&mut guess).expect("无法读取行");

    // {} 是一个占位，输出时候会被替换成 guess 变量的值
    // 一个 {} 对应一个变量的值
    println!("你猜测的数是：{}", guess);

}

```

## format

* 安装 format
```
$ rustup component add rustfmt
```

* 对代码进行格式化
```
$ cargo fmt
```


## 随机数
[随机数](https://crates.io/crates/rand)

一个 crate 就是一个库


### cargo update
* 会忽略 cargo.lock 中的版本，来找到 cargo.toml 里符合的版本

```
$ cargo update
```

### 添加依赖库 rand
* 在 cargo.toml 中添加 rand 库

```
[dependencies]
rand = "0.8.5"  // 指定版本号

```

之后 vscode 会下载依赖库，下载好后可执行 `cargo build`


* 生成随机数

```rust

// Rng 是一个 trait，可以看成是一个协议
use rand::Rng; // trait

    /* 
        ThreadRng 就是一个随机数生成器，它是位于本地线程空间
        并通过系统获得随机数的种子
        gen_range 在 [1, 101) 之间获取随机数
    */
    let rand_number = rand::thread_rng().gen_range(1..101);
    println!("神秘数字是 {}", rand_number);
```

> command+shift+p 查找开启 rust server
{: .prompt-info }


## 猜数游戏全部代码
```rust
// 引用标准库 std 的 io 库 可以使用 io
/*
rust 默认把 prelude（预导入） 导入到程序的每个作用域中,如果你使用的类型 prelude 里没有
这时你需要自行导入, 就是使用 use 导入库
 */
use std::io;
// Rng 是一个 trait，可以看成是一个协议
use rand::Rng; // trait

// 引入 Ordering，Ordering 是一个枚举类型它有三个值
use std::cmp::Ordering;

// fn 即声明一个函数
fn main() {
    println!("猜数!");

    /*
        ThreadRng 就是一个随机数生成器，它是位于本地线程空间
        并通过系统获得随机数的种子
        gen_range 在 [1, 101) 之间获取随机数
    */
    let rand_number = rand::thread_rng().gen_range(1..101);
    println!("神秘数字是 {}", rand_number);

    // 无限循环使用 loop 关键字
    loop {
        println!("猜一个数---- Begin");
        // rust 中 String 使用 utf-8 编码
        // :: 两个冒号表示后边的是一个静态方法
        // 这里 new 函数创建一个空白的字符串
        let mut guess = String::new();

        // 1 读取一行东西，并放到 guess 字符串引用里
        // 2 expect 为抛出异常时候输出的错误信息
        // 3 & 表示参数是一个引用
        // 4 read_line 参数是需要一个引用
        // 5 引用默认也是不可变的比如 &guess, 所以 &mut 就是可变引用
        //
        /*
           6
          read_line 返回 Result 类型，Result 就是枚举类型
          io::Result 有两个值 OK 和 Err
          如果是 OK 表示操作成功，并且里边有结果
          如果是 Err 表示操作失败，Err 里有失败 reason
          expect 是 Result 的方法，如果返回 Err 则 expect 会中断程序，将传入字符串显示
          如果返回 OK，expect 就将用户输入的值返回
        */
        io::stdin().read_line(&mut guess).expect("无法读取行");

        // 类型隐藏，rust 可以让定义同名的变量，一般用于类型转换
        // 从这行后边 guess 就是 u32 类型了
        // trim 会去掉字符串的空白
        // parse 将字符串转为 number 类型
        // shadow
        //let guess: u32 = guess.trim().parse().expect("Please type a number !");

        // 优化 match 是 rust 进行匹配的重要模式
        let guess: u32 = match guess.trim().parse() {
            Ok(num) => num,     // if 是 ok 则将值返回
            Err(_) => continue, //  if 是 error 则跳过本次进行下一次循环
        };

        // {} 是一个占位，输出时候会被替换成 guess 变量的值
        // 一个 {} 对应一个变量的值
        println!("你猜测的数是：{}", guess);

        // 拿 guess 与 rand_number 进行比较
        /*
           Ordering 是一个枚举类型它有三个值，Less、Greater、Equal


           一个 match 表达式由多个 arm 组成，
           如果 match 后边的值跟某个 arm 匹配了，那就执行那个 arm 的语句，
           match 匹配是从上到下代码进行匹配的
        */
        /*
           rust 是静态的强类型语言，有类型推断的能力
           把 match 下注释掉 rand_number 就变成 i32 类型了，
           有下边 match 代码 rand_number 就是 u32 类型，因为 guess 是 u32，比较时类型要一样
        */
        match guess.cmp(&rand_number) {
            Ordering::Less => println!("Too small"), // 这是一个 arm
            Ordering::Greater => println!("Too great"),
            Ordering::Equal => {
                println!("You win！");
                break; // 跳出 loop 循环
            }
        }
        println!("猜一个数---- End");
    }
}
```

# 基础

## 变量与可变性
* 声明变量使用 `let` 关键字
* 默认情况下，变量是不可变的（immutable）
* 在声明变量时，在变量前加上 mut，就可以使变量变为可变的

```rust
let mut x = 5;
println!("the x is {}", x);

x = 6;
println!("the x is {}", x);
```


## 常量与变量
* 常量（constant）在绑定值后也是不可变的，但是它与不可变的变量有很多区别
1. 常量不能使用 mut 关键字，因为常量永远都不可变
2. 声明常量使用 const 关键字，并且它的类型必须被标注清楚
3. 常量可以在任何作用域内进行声明，包括全局作用域
4. 常量只能绑定到常量表达式上，无法绑定到函数的调用结果或只能在运行时才能计算出的值
5. 在程序运行期间，常量在其声明的作用域内一直有效

* 常量的命名规范：Rust 里常量使用全大写字母，每个单词之间使用下划线分隔，例如 MAX_HEIGHT
* 常量声明例子
```rust
const MAX_HEIGHT: u32 = 100_000;
```


## Shadowing (隐藏)
* 在 Rust 中可以使用相同的名字声明新的变量，新的变量会 shadow（隐藏）之前声明的同名变量

```rust
let y = 5;
let y = y + 1; // shadowing ，在后续代码中这个变量 y 就代表新的变量 y
```

* shadow 与 mut 标记是不一样的
1. 使用 let 声明（shadow）的新变量，也是不可变的
2. 使用 let 声明（shadow）的新变量，它的类型可以与之前的不同

```rust
let spaces = "    ";          // 类型是 &str
let spaces = spaces.len();    // 类型是 usize
println!("spaces {}", spaces);    
```

# 数据类型
* 标量和复合类型
* rust 是静态编译语言，在编译时必须知道所有变量的类型
1. 基于使用的值，编译器通常能够推断出它的具体类型
2. 但如果可能的类型比较多（例如把 String 转为整数的 parse 方法），就必须添加类型的标注，否则编译器会报错


```rust
let guess: u32 = "42".parse().expect("Not a number");
println!("guess {}", guess);
```

## 标量类型
* 一个标量类型代表一个单个的值
* rust 有 4 个主要的标量类型
1. 整数类型
2. 浮点类型
3. 布尔类型
4. 字符类型

## 整数类型
* 例如 u32 是一个无符号，占 32 位的空间的整数
* 无符号以 u 开头，范围 0~2^n - 1
* 有符号以 i 开头，范围 -2^(n - 1)~2^(n - 1) - 1


![image](/assets/images/rust/integer.png)


* isize 和 usize 类型，arch 表示系统的架构
1. 由程序运行的计算机架构决定，如果是 64 位的那就是 64 位，如果是 32 位那就是 32 位的
2. 使用场景不多，主要场景是对某种集合进行索引操作

* 整数的字面值
1. 除了 byte 类型外，所有数值字面量都允许使用类型后缀
2. 如 57u8，即值是 57 类型是 u8

> 整数的默认类型是 i32，速度较快
{: .prompt-info }


![image](/assets/images/rust/iliteral.png)


### 整数的溢出
* 例如 u8 的范围是 0~255，如果把 u8 变量的值设置为 256
1. 在调试模式下编译：rust 会检查整数溢出，如果发生溢出，程序在运行时就会 panic
2. 在发布模式下（--release）编译：rust 不会检查可能导致 panic 的整数溢出，
3. 在发布模式下，如果发生溢出会进行环绕操作，256 变为 0, 257变为 1，不会 panic

## 浮点类型
* f32，32 位，单精度
* f64，64 位，双精度

* rust 浮点类型使用了 IEEE-754 标准来表示

> f64 是默认的浮点类型
{: .prompt-info }

## 布尔类型
* 占一个字节大小

## 字符类型
* char 类型被用来描述语言中最基础的单个字符
* 字符类型的字面量使用单引号
* 占 4 个字节
* 是 Unicode 标量值


```rust
let emoj = '🥰';
```

## 复合类型
* 两种复合类型：元组（Tuple）、数组

### Tuple 
* 可以将多个类型的多个值放到一个类型里
* 它的长度是固定的，一旦声明就无法改变
* Tuple 里个每个位置都对应一个类型，各元素类型不用相同
* 可以使用模式匹配来解构 Tuple 来获取元素的值
* 访问 Tuple 元素，使用索引


```rust
let tup: (i32, f64, u8) = (500, 6.4, 1);
let (x, y, z) = tup;
println!("{}, {}, {}, ", tup.0, tup.1, tup.2);
println!("{}, {}, {}, ", x, y, z);
```

### 数组
* 数组也可以将多个值放在一个类型里
* 数组中每个元素的类型必须相同
* 数组的长度也是固定的，一旦声明就无法改变
* 数组没有 Vector 灵活，Vector 类似数组是标准库提供的，它的长度是可变的
* 数组的类型，以 [类型;长度] 表示
* 数组是在 stack 上分配的单个块内存
* 如果访问索引越界了
1. 编译会通过
2. 运行时会报错 panic，不会允许继续访问相应的内存地址

```rust
let list = [1,2,3,4,5];
let a: [i32;5] = [1,2,3,4,5];

let a = [3; 5]; // 等于 [3,3,3,3,3]
println!("{}", a[0]);
```

# 函数
* 声明函数时使用 fn 关键字
* rust 使用 snake case 命名规范
1. 所有字母都是小写，单词之间使用下划线分隔

## 函数参数
1. parameters 形参
2. arguments 实参


> 函数签名里必须声明每个参数的类型
{: .prompt-info }

* rust 是基于表达式的语言，表达式会计算产生一个值


> 语句返回空的 Tuple， 即 ()
{: .prompt-info }


## 函数的返回值
* 返回值不能命名
* -> 后是返回值，需要声明返回值的类型
* 大多数函数都是使用函数体内最后一个表达式的值作为返回值
* 也可使用 return + 值，返回值

```rust
fn another_function(x: i32) {
    println!("another function x is {}", x);

    let y = {
        x + 3  // x + 3 是这个块的返回值
        //x + 3;   // 加了 ; 就变成语句了，而语句默认返回 空 tuple 即 ()
    };
    println!("another function y is {}", y);

    println!("6 + 5 is {}", plus_five(6));

}

fn plus_five(x: i32) -> i32 {
    x + 5
}
```

# 控制流

## if
* if 的条件必须是 bool 类型
* 条件分支叫 arm
* 如果使用了多于一个 else if，那么最好替换成 match
* if 表达式可以作为 let 的右项


```rust
let cond = true;
let number = if cond { 4 } else { 2 };
println!("number is {}", number);
```

## 循环
* rust 提供 3 种循环：loop、while、for

## loop
* 告诉 rust 反复执行一块代码，直到喊它停
* 可以在 loop 中使用 break 让它停止


```rust
let mut count = 0;
let result = loop {
    count += 1;

    if count == 10 {
        break count * 2;  // 返回 loop 块的结果给 result 变量
    }
};

println!("result is {}", result);
```

## while 循环
* 每次执行循环体之前都判断一次条件

```rust
let mut number = 3;
while number != 0 {
    println!("{}", number);

    number -= 1;
}
```

## for 循环
* 常用的循环方式


```rust
let a = [10, 20, 30, 40, 50];
// item is &i32
for item in a.iter() {
  println!("the value is {}", item);
}
```

* Range 与 for 循环
1. Range 指定一个开始数字和一个结束数字，[start, end)
2. rev 方法可以反转 Range

```rust
for num in (1..4).rev() { // (1..4) 就是 [1,4)
    println!("the num is {}", num);
}
```

