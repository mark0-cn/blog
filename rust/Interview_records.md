# 面试题目

某智驾公司 面试官很好，时长一个小时，在面试中学到了很多

### pin 在 future 中的作用

### 不同类型的sizeof是多少

```rust
use std::mem::{size_of, size_of_val};

pub enum E {
    A(u8),
    B(i16),
    C(i64),
}

// assume on 64-bit platform
fn main() {
    let t = ['1', '2', '3'];
    let p = &t[..2]; // slice
    let p1 = &t[..];
    let reference = &p;

    println!("size_of::<char>(): {}", size_of::<char>()); // 4
    println!("size_of::<[char; 3]>(): {}", size_of::<[char; 3]>()); // 12
    println!("size_of_val(&t): {}", size_of_val(&t)); // 8
    println!("size_of_val(&p): {}", size_of_val(&p)); // 8
    println!("size_of_val(&p1): {}", size_of_val(&p1)); // 8
    println!("size_of_val(&reference): {}", size_of_val(&reference)); // 8

    println!("size_of::<&E>(): {}", size_of::<&E>()); // 0000 ref: xxxx
    println!("size_of::<E>(): {}", size_of::<E>()); // 8 taggd union: max + discri // xxxxx

    println!("size_of::<Vec<String>>(): {}", size_of::<Vec<String>>());
    println!(
        "size_of::<Option<Vec<String>>>(): {}",
        size_of::<Option<Vec<String>>>()
    );
    println!(
        "size_of::<Option<&Vec<String>>>(): {}",
        size_of::<Option<&Vec<String>>>()
    );
}
```

### rust 内存泄漏问题

我在交流的时候说到了c语言内存泄漏，所以面试官问我rust的内存泄漏

### Cell 和 Refcell 的底层原理

### match 的模式匹配

这段代码输出什么？
```rust
fn main() {
    match_test(3, 4); // ？
    match_test(4, 4); // ？
    match_test(5, 4); // ？
}

fn match_test(x: i32, target: i32) {
    match x {
        1..=3 => {
            println!("A")
        }
        target => {
            println!("B")
        }
        _ => {
            println!("C")
        }
    }
}
```

### rust 异步机制是像 go 一样抢占式的吗

### enum 底层存储结构原理

### rust FFI 在使用的时候需要注意哪些问题

我在简历中写到了，所以面试官问了

### 一个结构体它的成员都实现了 send 和 sync 那么这个结构体是 send 或 sync 的吗？

### 解释某个具体的函数签名

```rust
pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future
    where
        <Self::Future as Future>::Output == Result<Self::Response, Self::Error>;

    fn poll_ready(
        &mut self,
        cx: &mut Context<'_>
    ) -> Poll<Result<(), Self::Error>>;

    fn call(&mut self, req: Request) -> Self::Future;
}
```

### static send move 有什么关联， static 有什么作用

```rust
pub fn spawn<F, T>(f: F) -> JoinHandle<T> 
where
    F: FnOnce() -> T,
    F: Send + 'static,
    T: Send + 'static,
```

### 笔试题

```rust
// compiles?
fn stuff(thing: &mut String) {
    let _a: &mut String = thing;
    let _b = thing;
}

// compiles?
fn stuff2(thing: &mut String) {
    let _a = &mut *thing;
    let _b = thing;
}

// compiles?
fn stuff3(thing: &mut String) {
    let _a = thing;
    let _b = thing;
}

```
