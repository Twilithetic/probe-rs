# Rust 闭包入门

## 什么是闭包

**闭包 = 可以随身携带的代码块**。不需要先定义名字再调用，而是当场写当场传。

```rust
//      ┌── 参数 ──┐  ┌── 函数体 ──┐
let f = |x, y| x + y;
//      ↑ 竖线是 Rust 选的定界符，不是括号

let result = f(2, 3);  // 5
```

## 和普通函数的区别

```rust
// 普通函数
fn add(x: i32, y: i32) -> i32 { x + y }

// 闭包 —— 不用声明类型（编译器自动推断），可以当场写
let add = |x, y| x + y;

// 闭包可以用在任何需要函数值的地方
vec![1, 2, 3].iter().map(|x| x * 2)        // |x| x * 2 就是闭包
vec![5, 1, 4].sort_by(|a, b| a.cmp(b))      // 排序用闭包
Option::None.unwrap_or_else(|| 42)           // || 42 无参闭包
```

## 三种调用方式 (FnOnce / FnMut / Fn)

```rust
// FnOnce — 只能调一次，会拿走捕获的变量
let s = String::from("hello");
let consume = || { drop(s); };   // s 被移进闭包
consume();
// consume();  // ❌ 不能第二次

// FnMut — 可以多次调，可以修改捕获的变量
let mut count = 0;
let mut inc = || { count += 1; };
inc(); inc(); inc();
println!("{count}");  // 3

// Fn — 可以多次调，不修改任何东西
let x = 5;
let show = || println!("{x}");
show(); show();  // ✅ 无限次
```

## 闭包怎么捕获变量

```rust
let n = 10;
let f = |x| x + n;   // n 不是参数，但闭包里能用！
```

编译器做的事：偷偷生成一个匿名 struct，把 `n` 存进去：

```rust
// 编译器暗中生成的等价代码：
struct __GeneratedClosure {
    n: i32,
}
impl Fn<(i32,)> for __GeneratedClosure {
    fn call(&self, (x,): (i32,)) -> i32 {
        x + self.n
    }
}
let f = __GeneratedClosure { n: 10 };
```

**本质就是一个带字段的 struct + 一个 call 方法。**

## 闭包当参数传

```rust
// 接收闭包做参数的函数
fn apply_twice(x: i32, f: impl Fn(i32) -> i32) -> i32 {
    f(f(x))
}

let result = apply_twice(5, |x| x + 1);  // 7
```

## 闭包当返回值

```rust
fn make_adder(n: i32) -> impl Fn(i32) -> i32 {
    move |x| x + n
    //  ↑ 关键字: 把 n 移进闭包（而不是借用）
}

let add_five = make_adder(5);
println!("{}", add_five(3));  // 8
```

## probe-rs 里的实际例子 (main.rs:615-623)

```rust
async fn run_app<R>(
    connection_params: Option<(String, Option<String>)>,
    cb: impl AsyncFnOnce(RpcClient) -> Result<R>,
    //  ↑ "一个接受 RpcClient 返回 Result 的异步闭包"
) -> Result<R> {
    let client = RpcClient::new_local_from_wire(tx, rx);  // 创建 client
    let result = cb(client).await;  // ← 调用闭包，把 client 塞给它
    result
}

// 调用方:
run_app(connection_params, async |client| {
//                            ↑ client 是 run_app 传进来的
    cli.run(client, config, utc_offset).await
}).await;
```

### 异步闭包 (async closure)

```rust
async |client| { cli.run(client, config, utc_offset).await }
//  ↑ 关键字
//       ↑ 参数
//               ↑ 闭包体里可以 .await

// 等价于:
|client| async move { cli.run(client, config, utc_offset).await }
```

## 快速对比

| 需求 | 写法 |
|------|------|
| 普通闭包 | `\|x\| x + 1` |
| 无参闭包 | `\|\| println!("hi")` |
| 修改捕获变量 | `\|x\| { count += x }` |
| 移走捕获变量 | `\|x\| { drop(s); }` |
| 异步闭包 | `async \|client\| { ... .await }` |
| 闭包当泛型参数 | `fn foo(f: impl Fn(i32) -> i32)` |
| 闭包存 struct | `Box<dyn Fn(i32) -> i32>` |

## 记忆口诀

> **竖线里写参数，竖线后写函数体。闭包就是没有名字的函数，能当参数传、能存变量里、还能当返回值。**
