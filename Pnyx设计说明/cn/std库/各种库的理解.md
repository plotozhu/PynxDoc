# Rust的核心理解目标
* 借用
* 可变不可变借用
* 线程间数据共享
* 返回值（Result/Option)
# impl 原语
```rust
  impl<T: ?Sized + Send + Sync> Sync for RwLock<T>
```
只有T先实现了Sized/Send/Sync后，才可以为`RwLock<T>`实现 Sync
# Send和Sync 
## 说明：
https://hexilee.me/2019/05/05/how-to-understand-sync-and-send-in-rust/#%E5%A6%82%E4%BD%95%E7%90%86%E8%A7%A3-syncsend  

核心理解：
* 实现了 Send 的类型，可以安全地在线程间传递所有权。也就是说， 可以跨线程移动。
* 实现了 Sync 的类型， 可以安全地在线程间传递不可变借用。也就是说，可以跨线程共享。

## Sized
https://zhuanlan.zhihu.com/p/21820917

* DST 是 Dynamic Sized Type 的缩写，意思是动态大小类型，表示在编译阶段无法确定大小的类型。
* Rust中有一个重要的 trait Sized，可以用于区分一个类型是不是 DST。所有的 DST 类型都不满足 Sized 约束。我们可以在泛型约束中使用 Sized、!Sized、?Sized 三种写法。其中 T:Sized 代表类型必须是编译期确定大小的，T:!Sized 代表类型必须是编译期不确定大小的，T:?Sized 代表以上两种情况都可以。在泛型代码中，泛型类型参数默认携带了 Sized 约束，因为这是最普遍最常见的情况。如果我们希望这个泛型参数也可以支持 DST 类型，那么就应该为它专门加上 ?Sized 约束，示例如下：
```rust
use std::fmt::Debug;

fn call<T>(p : &T) where T:Debug
{
    println!("{}", p);
}

fn main() {
    let x : &[i32] = &[1,2,3,4];
    call(x);
}
```
以上写法，等同于默认有一个 T:Sized 约束。当参数是 &[i32] 类型的时候，编译器推理出来泛型参数是 [i32]，不符合 Sized 约束，就会报错。修复方案是，加上 T: ?Sized 约束：
```rust
use std::fmt::Debug;

fn call<T : ?Sized>(p : &T) where T: Debug
{
    println!("{:?}", p);
}

fn main() {
    let x : &[i32] = &[1,2,3,4];
    call(x);
}
```