---
layout: post
title: Rust Match Syntax
tags: 
- note
- reading
- rust
---

match 控制流运算符，通过模式匹配完成程序流控制。小到变量声明绑定，大到结构体解构到处都是match结构的身影。

模式由一些常量组成：解构数组，枚举，结构体或者是元组，变量，通配符，占位符。描述了要处理的额数据的形状。

match是穷尽的

# 语法
```rust
// 创建一个枚举变量
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quater(UsState),
}

// 通过模式匹配返回各个硬币类型代表的数值
fn value_in_cents(coin: coin) -> u32 {
    match Coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter(state) => {   // 值绑定模式，内部绑定的值会被绑定到state上，在之后的代码块中就可以使用了
            println!("State quarter from {:?}!", state);
            25
        },
    }
}

// 匹配Option<T>
match x {
    Some(i) => i,
    None => None,
}

// 匹配字面量
match x {
    1 => {
        // some code 
    },
    2 => 33,
}

// 匹配命名变量
let x = Some(5);
let y = 10;

match x {
    Some(50) => 50,
    Some(y) => println!("Matched, y = {:?}", y),  // 此时y不再是外部作用于的y了，绑定之后是模式匹配后提取出来的值
}

// 匹配多种模式
match x {
    1 | 2 => println!("one or two"),
}

// 匹配范围
match x {
    1 ... 3 => println!("one through 3"),
    _ => println!("ignore ..."),
}

match y {
    'a' ... 'z' => println!('y: {}', a);
}

// 解构并提取值
struct Porint {
    x: i32, 
    y: i32,
}

let p1 = Point{x: 32, y: 64};

let Point{a, b} = p1;
assert_eq!(32, a);
assert_eq!(64, b);

let points = vec![
    Point { x: 0, y: 0 },
    Point { x: 1, y: 5 },
    Point { x: 10, y: -3 },
];
let sum_of_squares: i32 = points
    .iter()
    .map(|&Point {x, y}| x * x + y * y)
    .sum();
    
let ((feet, inches), Point {x, y}) = ((3, 10), Point { x: 3, y: -10 });

fn main() {
    let p = Point { x: 0, y: 7 };

    match p {
        Point { x, y: 0 } => println!("On the x axis at {}", x),
        Point { x: 0, y } => println!("On the y axis at {}", y),
        Point { x, y } => println!("On neither axis: ({}, {})", x, y),
    }
}

// 忽略模式中的值
// `_`通配符用于不想列举出所有可能值的场景。`_ => (),`;
// `_x` 用于需要忽略的变量。和`_`的区别是`_`不会绑定变量，而`_x`会（所有权转移不一样)，用///于一些生命了但是没用到的变量;
// `..`用来忽略多个值，但是要是明确的使用方式，`first, ..`, `first, .. last`, `.., last`, `first, ..`;

match x {
    1 => {},
    2 => {},
    .. => (),
}

match x {
    Some(_) => println!("got a Some and I don't care what's inside"),
    None => (),
}

let numbers = (2, 4, 8, 16, 32);

match numbers {
    (first, _, third, _, fifth) => {
        println!("Some numbers: {}, {}, {}", first, third, fifth)
    },
}

fn main() {
    let _x = 5;
    let y = 10;
}

struct Point {
    x: i32,
    y: i32,
    z: i32,
}

let origin = Point { x: 0, y: 0, z: 0 };

match origin {
    Point { x, .. } => println!("x is {}", x),
}

// 在模式中创建引用(ref, ref mut)
let robot_name = Some(String::from("Bors"));

match robot_name {
    Some(ref name) => println!("Found a name: {}", name),
    None => (),
}

println!("robot_name is: {:?}", robot_name);

// 在模式中使用匹配守卫
let num = Some(4);

match num {
    Some(x) if x < 5 => println!("less than five: {}", x),
    Some(x) => println!("{}", x),
    None => (),
}

let x = 4;
let y = false;

match x {
    4 | 5 | 6 if y => println!("yes"),  // == (4 | 5 | 6) if y
    _ => println!("no"),
}

// 测试值的同时绑定一个值到对应的变量 @ 绑定
enum Message {
    Hello { id: i32 },
}

let msg = Message::Hello { id: 5 };

match msg {
    Message::Hello { id: id @ 3...7 } => {
        println!("Found an id in range: {}", id)
    },
    Message::Hello { id: 10...12 } => {
        println!("Found an id in another range")
    },
    Message::Hello { id } => {
        println!("Found some other id: {}", id)
    },
}
```

## refutable/irrefutable（可反驳的和不可反驳的）
匹配模式有两种形式: refutable(可反驳)和irrefutable(不可反驳).对任意可能的值进行匹配都不会失效的模式被称为是irrefutable(不可反驳)的,而对某些可能的值进行匹配会失效的模式被称为是refutale(可反驳)的. let语句、 函数参数和for循环被约束为只接受irrefutable模式,因为如果模式匹配失效程序就不会正确运行. if let和while let表达式被约束为只接受refutable模式,因为它们需要处理可能存在的匹配失效的情况,并且如果模式匹配永不失效, 那它们就派不上用场了.

## let
解构右边的内容并绑定到变量
```rust
let PATTERN = EXPRESSION;   // 变量名本质上是一个特别朴素的模式
let (x, y, z) = (1, 2, 3);  // 复杂一些的let 模式解构，元组被分别绑定到不同的变量上
let x: i32 = 5;             // i32
```

## if-let
当你只关心其中的一个模式的时候，可以直接使用`if let`
```rust
if let Some(x) = test_option {
    println!("get x: {}", x);
} else if let Ok(age) = age {
    if age > 50 {
        50
    }
}
```

## while-let
循环绑定模式匹配中的值，只要匹配就一直循环下去
```rust
let mut stack = vec![1, 2, 3];

while let Some(top) = stack.pop() {
    println!{"{}", top};
}
```

## for let
在for循环中解构模式
```rust
let v = vec![1, 2, 3];
for (index, value) in v.iter().enumerate() {   // index, value 就是一个模式
    println!("{} is at index {}", value, index);
}
```

## fn 参数匹配
函数参数中的模式匹配
```rust
fn print_x_y(&(x, y): &(i32, i32)) {
    // x, y可以使用了
}
```