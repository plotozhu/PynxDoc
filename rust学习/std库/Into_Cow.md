# 介绍Into
标准库提供了Intotrait可以用来解决我们new方法的问题，trait的定义如下：
```
pub trait Into<T> {
  fn into(self) -> T;
}
```
into方法的定义非常直接，它接受self，然后将其转化为T类型。其使用方式如下：
```
impl Token {
  pub fn new<S>(raw: S) -> Token 
      where S: Into<String>
  {
    Token {raw: raw:into()}
  }
}
// &str
let token = Token::new("abc123");
// String
let token = Token::new(secret_from_vault("api.example.io"));
```
由于标准库中已经为&str实现了Into<String>所以这里可以直接编译通过，但是这样又有另外一个问题，每次使用&str的时候都要进行堆分配，创建String


# 使用Cow解决问题
标准库也提供了一个std::borrow::Cow的类型，使用它既可以保持API像Into<String>一样简捷，同时允许使用借用值例如&str。
**我的理解** ：使用Cow后，如果生命周期符合要求，就直接变成Owned使用，如果生命周期不符合要求，就复制一个，然后变成Owned。因此既可以是String,也可以是&str借用。

Cow的定义如下：
```
pub enum Cow<'a, B> where B: 'a + ToOwned + ?Sized {
  Borrowed(&'a B),
  Owned(B::Owned),
}
```
现在我们分析一下上面代码的意义：

* Cow<'a B>有两个泛型参数: 生命周期参数'a，和类型B
* B的泛型限制为：'a + ToOwned + ?Sized
  * 'a 表示 B 的生命周期不能短于 'a
  * ToOwned 表示B必须实现ToOwned
  * ?Sized 表示B的大小在编译期间未知。
* 两个变量
  * Borrowed(&'a B) 类型为B的对象的引用。
  * Owned(B::Owned) B类型有一个关联类型Owned，这个变量保持这个类型的值  
  
我们需要的是一个Cow<'a str>的类型，
```rust
enum Cow<'a, str>. {
  Borrowed(&'a str),
  Owned(String),
}
```   
简单来讲，Cow<'a, str>要么返回一个&'a str要么返回一个String。所以使用Cow进行改造new方法如下：  
```rust
struct Token<'a> {
  raw: Cow<'a, str>,
}

impl <'a> Token<'a> {
  pub fn new(raw: Cow<'a, str>) -> Token<'a> {
    Token {raw: raw}
  }
}

let token = Token::new(Cow::Borrow("abc123"));
let secret: String = secret_from_vault("api.example.io");
let token = Token::new(Cow::Owned(secret));
```   
现在Token既可以接受String也可以使用&str，但是这样会导致API不那么易用，所以可以使用Into的trait来解决这个问题  

```rust
struct Token<'a> {
  raw: Cow<'a, str>
}

impl <'a> Token<'a> {
  pub fn new<S> (raw: S) -> Token<'a> where S: Into<Cow<'a, str>> 
  {
    Token {raw: raw.into()}
  }
}

let token = Token::new("abc123");
let token = Token::new(secret_from_vault("api.example.io"));
```
现在Token既可以接受一个String也可以接受一个&str作为参数了。