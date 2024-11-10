---
title: rust基础知识
date: 2024-03-20 16:13:42
tags: 基础知识
categories: rust
---
## 变量与可变性
在rust中通过 let 关键字来声明一个变量，并未变量绑定一个值，变量默认为不可变，即初始化之后无法对变量做出修改，可以不显示指定变量类型，编译器会自动推断。
```rust
    let a = 5;
    a = 10  // 无法修改，会出错

    let b: i32 = 5;   // 可以在变量名后跟 :变量类型  的方式来指定变量类型
```
通过 mut 关键字使得变量具有可变性。
```rust
    let mut a = 5;  // 通过 mut 关键字声明变量，则变量具有可变性
    a = 10; // 可以更改变量
```
常量在rust中与不可变变量类似，但常量在声明时需要使用 const 关键字，并且在声明时必须指定类型，赋值只能为一个常量表达式，不能为一个需要在运行时得出的值。
```rust
    const NUM: i32 = 100 * 100;
```
rust中变量具有遮蔽行为，即在一个变量定义之后，可以使用相同的变量名来遮蔽之前的变量。遮蔽与可变性不同，遮蔽可以理解为重新创建了一个新的变量，类型可与之前不同。
```rust
    let a = 5;
    ···
    let a = "hello";  // 在此之前变量 a 的值为5，之后值为 hello
```
## 函数
rust中函数的入口时 main 函数，默认情况下 main 函数没有参数和返回值。
```rust
    fn main() {

    }
```
函数的参数需要显示的指定类型，返回值类型则跟在小括号后。
```rust
    fn function(x:i32) -> i32 { // 需要一个 i32 类型的参数，返回一个 i32 的值。
        x
    }
```
语句与表达式，语句不会返回值，表达式会返回值。表达式可以赋值给另一个变量，而语句不可以。同理表达式可以作为函数的返回值，比如上面函数返回 x
```rust
    let a = 10;   // 这是一条语句，没有返回值
    a   // 这是一个表达式，会返回 a 
```
## 控制流
if 判断语句，直接在 if 后跟判断的条件即可，注意在 rust 中 if 是一个表达式，会产生值，可以赋值给另一个变量。
```rust
    if a > 5 {
        println!("a>5");
    } else {
        println!("a<=5");
    }
    // 返回两个数中的较大值
    let max = if a>b {
        a
    } else {
        b
    };
    // if 后同样可以接多个 else if
    if a > 90 {
        ···
    } else if a > 80 {
        ···
    } else if a > 70 {
        ···
    } else {
        ···
    }
```
loop 循环语句，执行循环，直到代码指定跳出，否则一直循环下去，loop 同样属于语句，可以有返回值，返回值可通过 break 返回。
```rust
    let mut a = 0;
    let b = loop {
        a = a + 1;
        if a > 100 {
            break a;    // 这里 break 指定跳出循环，并返回值给变量 b
        }
    };
```
while 条件循环，当条件为真时，执行循环，为假时跳出循环
```rust
    let mut a = 0;
    while a < 100 {
        a = a + 1;
    }
```
for 循环，相比 while 循环，for 循环在遍历集合中的元素时更加方便。并且不会出现越界的行为。
```rust
    let a = [1,2,3,4,5,6];

    for i in a {
        println!("a{i}={i}");
    }
```
## 所有权
所有权是rust特有的特性，同样也是核心功能之一，保证rust在没有gc的情况下还能够做到内存安全。

rust所有权有三个规则：
- Rust 中的每一个值都有一个被称为其 所有者（owner）的变量。
- 值在任一时刻只有一个所有者。
- 当所有者离开作用域，值将被丢弃。

首先来看作用域：
```rust
{                               // s 未进行声明，不可使用
    let s = "hello";            // 声明变量 s 变量 s 有效，可使用


}                               // 离开作用域，变量 s 无效，值被丢弃
```
接下来需要介绍一下栈内存和堆内存，在rust中，如果数据的大小是固定的且已知的，那么会被存放在栈内存中，否则将会被存放在堆内存，所有权更主要是管理堆上的内存空间。

以String类型为例：String字符串在创建是未知大小，所以会被分配到堆空间。
```rust
{
    let mut s = String::from("hello");      // 在堆空间分配内存存放值
    s.push_str("world");                    // 在字符串值后追加内容
}                                           
```
字符串的内容会被存储在堆空间中，在栈上会存放一个指向堆数据的指针，这个指针大小是固定的。

而对于大小确定的类型，则会分配在栈内存中，比如i32。
```rust
{
    let n:i32 = 6;
}
```
而对于分配在栈上的数据，在移动时会发生 copy 行为，简单理解就是复制了一份同样的数据放在了栈中，原本的数据还存在，也就是说有两个可用的数据，每个数据分别属于各自的变量。这是因为栈上的数据 copy 速度很快，直接 copy 没有什么问题。
```rust
{
    let a:i32 = 6;
    let b = a;     // 此时发生了 copy 行为，栈中会存在两份数据，a 和 b 均可用
}
```
而对于堆上的数据则不是这样，在移动时不会去复制堆上的数据(除非你使用 clone 去指定)，一般是再创建一个在栈上的指针，指向堆上的数据，那么现在就存在问题了，一份堆上的数据，有两个指针都指向了它，这和上面所说的值在任一时刻只有一个所有者违背，并且当离开作用域时，两个所有者都会对这个空间进行释放，这会出现问题，所以在 rust 中，当一个堆上的数据被移动时会发生 move 行为，即值移动到了新的所有者，旧的所有者不再有作用，如果访问会发生错误，这是在离开作用域时也不会被释放两次。
```rust
{
    let s = String::from("hello");
    let m = s;   //  此时 s 的值被移动到了 m 中，s 不在有效，无法使用，在离开作用域是 m 负责释放堆上对应的空间。
}
```
还有一种数据传递方式就是克隆，如果使用 clone 方法，那么数据会被完整的 clone 出一份，传递给新的所有者，旧的数据依然属于原来的所有者。
```rust
{
    let s = String::from("hello");
    let m = s.clone()  // 此时调用了clone函数，所以 s 值的所有权没有发生移动，s 依然可以使用。
}
```
在函数参数以及返回值中也同样要遵循所有权规则，按照参数的类型，以及存放的位置，会发生对应的 copy 或者 move 行为。
## 引用和借用
引用，顾名思义，需要引用其他的值，同时不会拿走值的所有权。

rust 程序语言设计中的例子：
```rust
fn main() {
    let s1 = String::from("hello");    // 此处创建了一个字符串变量

    let len = calculate_length(&s1);   // 此处对变量进行了引用，使用 & 符号，没有拿走值的所有权

    println!("The length of '{}' is {}.", s1, len);  // 因此在函数调用完成后，s1 变量的值依然存在未被丢弃
}

fn calculate_length(s: &String) -> usize {  // 此处参数声明的 & 符号即是引用符号
    s.len()
}

```

可以看出通过 & 符号与一个值可以创建一个值所对应的引用，可以使用值的同时不转移所有权，引用使用后被丢弃，但是原本的值不收影响。这种引用的行为被称为借用。

这里只通过 & 符号创建的引用是不可变引用，即不可以修改。
```rust
fn main() {
    let s = String::from("hello");

    change(&s);
}

fn change(some_string: &String) {
    some_string.push_str(", world");  // 尝试修改不可变引用会发生错误
}

```
与之相对应的则是可变引用，通过 &mut 来进行创建。
```rust
fn main() {
    let mut s = String::from("hello");

    change(&mut s);  // 这里创建了一个 s 的可变引用。注意，此处需要 s 本身也是可变的才可以。
}

fn change(some_string: &mut String) { // 此处函数参数在进行声明时使用了可变引用
    some_string.push_str(", world");  // 此处可以通过这个可变引用来修改原本的值。
}

```
可变引用在同一时间只能存在一个，不可变引用可以同时存在多个，这主要是为了避免数据竞争。同时在不可变引用存在期间，不可以有可变引用。
```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &mut s;
    let r2 = &mut s;   // 此处同时创建了两个可变引用，并且两个可变引用生命周期出现了重叠，这是不允许的

    println!("{}, {}", r1, r2); // 两个引用在此同时使用
}

```
下面这种情况下，两个可变引用生命周期未发生重叠，属于合法的。
```rust
fn main() {
    let mut s = String::from("hello");

    {
        let r1 = &mut s;
    } // r1 在这里离开了作用域，所以我们完全可以创建一个新的引用

    let r2 = &mut s;
}

```
在不可变引用存在期间，不允许创建可变引用。
```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s; // 没问题
    let r2 = &s; // 没问题
    let r3 = &mut s; // 大问题

    println!("{}, {}, and {}", r1, r2, r3);
}

```
这里引用的生命周期结束，即指引用最后一次使用的地方，在此之后，这个引用被视为生命周期结束。
```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s; // 没问题
    let r2 = &s; // 没问题
    println!("{} and {}", r1, r2);
    // 此位置之后 r1 和 r2 不再使用

    let r3 = &mut s; // 没问题
    println!("{}", r3);
}

```
同时引用也有着限制，引用的生命周期不能比原本值的生命周期更长，这会出现悬垂引用的问题，出现这种情况是编译器一般会报错。
## 切片
切片允许你引用集合中的一段连续的元素，而切片同样也不会拿走所有权。
```rust
    let s = String::from("hello world");
    let s1 = &s[0..5];
    let s2 = &s[6..11];
```
可以看到切片的用法 `[start..end]` 通过指定起始坐标以及结束坐标来获取指定的切片，这里会包含 start 而不包含 end。

如果切片的起始位置是0，则0可以省略。
```rust
    let s = String::from("hello world");
    let s1 = &s[0..5];
    let s2 = &s[..5];  // 这两种写法获取的切片是相同的。
```
同样，如果切片的结尾包含了集合的最后元素，那么 end 也是可以省略的。
```rust
    let s = String::from("hello world");
    let len = s.len();
    let s1 = &s[3..len];
    let s2 = &s[3..];  // 这两种写法获取的切片是相同的。
```
同样通过切片的方式创建的值是不可变的。
## 结构体
结构体允许你将不同类型的元素组合在一起，并给每个元素赋予一个名字，以此来组成一个特殊的具有意义的值。而结构体在进行访问时可以直接通过元素名字来访问。

定义结构体需要使用 `struct` 关键字 + 结构体名字来定义，后面使用大括号将结构体元素包含。
```rust
struct User {      // 定义了结构体类型名字 User 
    active: bool,    //  结构体中的各个元素，以及元素对应的类型
    username: String,
    email: String,
    sign_in_count: u64,
}

fn main() {}

```
结构体在进行使用时，需要实例化一个结构体，在实例化过程中需要为每个元素赋予对应类型的值，赋值可以使用 `元素名: 值` 的方式。
```rust
struct User {
    active: bool,
    username: String,
    email: String,
    sign_in_count: u64,
}

fn main() {
    let user1 = User {     //  通过结构体类型名来创建一个结构体
        email: String::from("someone@example.com"),   //  为结构体的每个元素赋值，顺序可以与定义顺序不同
        username: String::from("someusername123"),
        active: true,
        sign_in_count: 1,
    };

    let email = user1.email   // 使用结构体中的值可以使用 . 符号
}

```
如果结构体是可变的，那么可以通过 `结构体变量名.元素名 = ` 的方式改变结构体元素值。**注意必须整个实例是可变的，rust 不支持只将某个元素标记为可变**
```rust
struct User {
    active: bool,
    username: String,
    email: String,
    sign_in_count: u64,
}

fn main() {
    let mut user1 = User {   //  实例可变
        email: String::from("someone@example.com"),
        username: String::from("someusername123"),
        active: true,
        sign_in_count: 1,
    };

    user1.email = String::from("anotheremail@example.com");  // 给实例的元素修改值
}

```
在通过变量为结构体元素赋值时，如果变量名和结构体元素名相同，则可以简写为结构体元素名即可。`元素名: 变量名` -> `元素名`
```rust
struct User {
    active: bool,
    username: String,
    email: String,
    sign_in_count: u64,
}

fn build_user(email: String, username: String) -> User {
    User {
        email,     // 变量名与元素名相同，简写即可
        username,
        active: true,
        sign_in_count: 1,
    }
}

fn main() {
    let user1 = build_user(
        String::from("someone@example.com"),
        String::from("someusername123"),
    );
}

```
通过结构体实例来创建新结构体时，可以使用结构体更新语法来创建结构体，只需更新不同的元素的值即可，相同的元素值可以使用 `..实例名` 来省略。
```rust
struct User {
    active: bool,
    username: String,
    email: String,
    sign_in_count: u64,
}

fn main() {
    // --snip--

    let user1 = User {
        email: String::from("someone@example.com"),
        username: String::from("someusername123"),
        active: true,
        sign_in_count: 1,
    };

    let user2 = User {
        email: String::from("another@example.com"),  // 更新与 user1 不同的元素
        ..user1  // 相同元素可以省略
    };
}

```
结构体同样也会涉及到所有权问题，包括结构体中的元素，如果通过结构体 user1 来创建了 user2 ，那么 user1 将不能再使用，如果将结构体中的元素值赋予了其他变量，那么元素的所有权也将转移。

比较特殊的还有单元结构体，其中没有包含任何元素，如果仅仅想在结构体上实现某些 trait 而不需要携带数据时，可以使用单元结构体
```rust
struct AlwaysEqual;

fn main() {
    let subject = AlwaysEqual;
}

```