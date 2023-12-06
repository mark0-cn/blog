### clone trait 与 copy trait 的区别 

```
copy():
+--------+       +--------+
| person1|       | person2|
+--------+       +--------+
|  name  |----|   name  |----------
+--------+    |   +--------+      |
|  age   |    |   |  age   |      |
+--------+    |   +--------+      |
              |                   |
              -----            |---
                  |            | 
堆空间：           |     --------
                  V    V
+--------+      +---------+  
|  name  |----> | "Alice" | 
+--------+      +---------+  

clone()方法复制：
+--------+       +--------+
| person1|       | person3|
+--------+       +--------+
|  name  |----   |   name  |------
+--------+    |  +--------+      |
|  age   |    |  |  age   |      |
+--------+    |  +--------+      |
              |                  |
              -----              |
                  |              | 
堆空间：           |              |     
                  V              V
+--------+      +---------+   +---------+  
|  name  |----> | "Alice" |   | "Alice" | 
+--------+      +---------+   +---------+  
                   旧空间         新空间
```

### 什么情况使用panic

+ 非预期的错误
+ 后续代码的运行会受到显著影响
+ 内存安全的问题

### panic 原理剖析

当调用 panic! 宏时，它会

1. 格式化 panic 信息，然后使用该信息作为参数，调用 std::panic::panic_any() 函数

2. panic_any 会检查应用是否使用了 panic hook，如果使用了，该 hook 函数就会被调用（hook 是一个钩子函数，是外部代码设置的，用于在 panic 触发时，执行外部代码所需的功能）

3. 当 hook 函数返回后，当前的线程就开始进行栈展开：从 panic_any 开始，如果寄存器或者栈因为某些原因信息错乱了，那很可能该展开会发生异常，最终线程会直接停止，展开也无法继续进行

4. 展开的过程是一帧一帧的去回溯整个栈，每个帧的数据都会随之被丢弃，但是在展开过程中，你可能会遇到被用户标记为 catching 的帧（通过 std::panic::catch_unwind() 函数标记），此时用户提供的 catch 函数会被调用，展开也随之停止：当然，如果 catch 选择在内部调用 std::panic::resume_unwind() 函数，则展开还会继续。

还有一种情况，在展开过程中，如果展开本身 panic 了，那展开线程会终止，展开也随之停止。

一旦线程展开被终止或者完成，最终的输出结果是取决于哪个线程 panic：对于 main 线程，操作系统提供的终止功能 core::intrinsics::abort() 会被调用，最终结束当前的 panic 进程；如果是其它子线程，那么子线程就会简单的终止，同时信息会在稍后通过 std::thread::join() 进行收集。

### Cargo.toml vs Cargo.lock

Cargo.toml 和 Cargo.lock 是 Cargo 的两个元配置文件，但是它们拥有不同的目的:

+ 前者从用户的角度出发来描述项目信息和依赖管理，因此它是由用户来编写
+ 后者包含了依赖的精确描述信息，它是由 Cargo 自行维护，因此不要去手动修改

它们的关系跟 package.json 和 package-lock.json 非常相似，从 JavaScript 过来的同学应该会比较好理解。

是否上传本地的 Cargo.lock？

当本地开发时，Cargo.lock 自然是非常重要的，但是当你要把项目上传到 Git 时，例如 GitHub，那是否上传 Cargo.lock 就成了一个问题。

关于是否上传，有如下经验准则:

+ 从实践角度出发，如果你构建的是三方库类型的服务，请把 Cargo.lock 加入到 .gitignore 中。
+ 若构建的是一个面向用户终端的产品，例如可以像命令行工具、应用程序一样执行，那就把 Cargo.lock 上传到源代码目录中。

例如 axum 是 web 开发框架，它属于三方库类型的服务，因此源码目录中不应该出现 Cargo.lock 的身影，它的归宿是 .gitignore。而 ripgrep 则恰恰相反，因为它是一个面向终端的产品，可以直接运行提供服务。

那么问题来了，为何会有这种选择？

原因是 Cargo.lock 会详尽描述上一次成功构建的各种信息：环境状态、依赖、版本等等，Cargo 可以使用它提供确定性的构建环境和流程，无论何时何地。这种特性对于终端服务是非常重要的：能确定、稳定的在用户环境中运行起来是终端服务最重要的特性之一。

而对于三方库来说，情况就有些不同。它不仅仅被库的开发者所使用，还会间接影响依赖链下游的使用者。用户引入了三方库是不会去看它的 Cargo.lock 信息的，也不应该受这个库的确定性运行条件所限制。

### UnsafeCell Cell RefCell

为了实现内部库可变性(interior mutability)

UnsafeCell -> Cell -> RefCell

UnsafeCell

提供一个get方法，可以使不可变引用&T，获取到可变引用*mut T指针，它通过强转实现，强转到裸指针是安全的，但要注意正确的对它进行解引用。

```rust
pub const fn get(&self) -> *mut T {
   self as *const UnsafeCell<T> as *const T as *mut T
}
```

为什么可以将*const UnsafeCell<T>强转成*const T ?
UnsafeCell使用了#[repr(transparent)]属性宏，显示指定了UnsafeCell<T>内存结构必须与T相同，所以完全可以转换UnsafeCell<T>到T

Cell

```rust
impl<T> Cell<T> {
   pub fn set(&self, val: T) {
      let old = self.replace(val);
      drop(old);
   }
   pub fn replace(&self, val: T) -> T {
      mem::replace(unsafe { &mut *self.value.get() }, val)
   }
}
```

通过Unsafe block调用Unsafecell的get方法获取&mut T，之后通过mem::replace进行替换

RefCell

```rust
#[cfg_attr(not(test), rustc_diagnostic_item = "RefCell")]
#[stable(feature = "rust1", since = "1.0.0")]
pub struct RefCell<T: ?Sized> {
    borrow: Cell<BorrowFlag>,
    // Stores the location of the earliest currently active borrow.
    // This gets updated whenever we go from having zero borrows
    // to having a single borrow. When a borrow occurs, this gets included
    // in the generated `BorrowError`/`BorrowMutError`
    #[cfg(feature = "debug_refcell")]
    borrowed_at: Cell<Option<&'static crate::panic::Location<'static>>>,
    value: UnsafeCell<T>,
}
```

提供了borrow和borrow_mut方法用来获取&T和&mut T

通过对BorrowFlag值的加减操作实现对Ownship的检查

### 关联类型与泛型类型的区别

- 对于某一特性，每个类型仅应当有单一实现时，使用关联类型。
- 对于某一特性，每个类型可以有多个实现时，使用泛型类型。

https://github.com/pretzelhammer/rust-blog/blob/master/posts/translations/zh-hans/tour-of-rusts-standard-library-traits.md#%E6%B3%9B%E5%9E%8B%E7%B1%BB%E5%9E%8B%E4%B8%8E%E5%85%B3%E8%81%94%E7%B1%BB%E5%9E%8B-generic-types-vs-associated-types

### 子特性与超特性 Subtraits & Supertraits

子特性的“子”即为子集，超特性的“超”即为超集。若有下列特性声明：

```rust
trait Subtrait: Supertrait {}
```

所有实现了子特性的类型都是实现了超特性的类型的子集，也可以说，所有实现了超特性的类型都是实现了子特性的类型的超集。

以上代码等价于：

``` rust
trait Subtrait where Self: Supertrait {}
```

此外，对于特定类型如何同时实现子特性与超特性并没有规定。子、超特性之间的方法也可以相互调用。

```rust
trait Supertrait {
    fn super_method(&mut self);
}

trait Subtrait: Supertrait {
    fn sub_method(&mut self);
}

struct CallSuperFromSub;

impl Supertrait for CallSuperFromSub {
    fn super_method(&mut self) {
        println!("in super");
    }
}

impl Subtrait for CallSuperFromSub {
    fn sub_method(&mut self) {
        println!("in sub");
        self.super_method();
    }
}

struct CallSubFromSuper;

impl Supertrait for CallSubFromSuper {
    fn super_method(&mut self) {
        println!("in super");
        self.sub_method();
    }
}

impl Subtrait for CallSubFromSuper {
    fn sub_method(&mut self) {
        println!("in sub");
    }
}

struct CallEachOther(bool);

impl Supertrait for CallEachOther {
    fn super_method(&mut self) {
        println!("in super");
        if self.0 {
            self.0 = false;
            self.sub_method();
        }
    }
}

impl Subtrait for CallEachOther {
    fn sub_method(&mut self) {
        println!("in sub");
        if self.0 {
            self.0 = false;
            self.super_method();
        }
    }
}

fn main() {
    CallSuperFromSub.super_method(); // prints "in super"
    CallSuperFromSub.sub_method(); // prints "in sub", "in super"
    
    CallSubFromSuper.super_method(); // prints "in super", "in sub"
    CallSubFromSuper.sub_method(); // prints "in sub"
    
    CallEachOther(true).super_method(); // prints "in super", "in sub"
    CallEachOther(true).sub_method(); // prints "in sub", "in super"
}
```

现在，让我们看看 Copy 特性的声明：

```rust
trait Copy: Clone {}
```

以上的记号和之前我们为泛型添加特性约束的记号非常相似，但是 Copy 却完全不依赖 Clone 。早前建立的心智模型现在不适用了。在我看来，理解子特性与超特性的关系的最简单和最优雅的心智模型莫过于 —— 子特性 改良 了超特性。

“改良”一词故意地预留了一些模糊的空间，它的具体含义在不同的上下文中有所不同：

- 子特性可能比超特性的方法更加特异化、运行更快或使用更少内存等等，例如 Copy: Clone
- 子特性可能比超特性的方法具有额外的功能，例如 Eq: PartialEq ， Ord: PartialOrd 和 ExactSizeIterator: Iterator
- 子特性可能比超特性的方法更灵活和更易于调用，例如 FnMut: FnOnce 和 Fn: FnMut
- 子特性可能扩展了超特性并添加了新的方法，例如 DoubleEndedIterator: Iterator 和 ExactSizeIterator: Iterator