# 模块 std::pin
钉住数据在内存中的位置类型

有时候，保证对象不在内存中移动是有用的，在这些场合里，他们在内存中的位置不变化并且值得依赖。这种应用场景的一个主要的例子就是创建自引用结构，当移动了一个有指向自身的指针字段时，（指针字段将会悬挂），这将会导致未定义的行为结果。
**注意** 本文所说的移动可能是指内存中的数据变了（无论是移动了位置，被重新写入数据，或是被释放了； 也可能是指数据所有权被移动（尤其是结构中的字段，从而导致数据可能被修改或是无法被释放）

`Pin<P>`保证任何任何指向P的指针在内存在有一个固定的位置，这意味着它可以被移动到任何的位置而他的内存直到在被Drop之前将不会被释放，我们称这个指针是被“钉定”的

默认Rust中所有的类型都是可移动的，Rust允许通过值传递所有的类型，而且通用的智能指针类型如：`Box<T>`和`&mut T`允许替换和移动`T`的值：你可以移出一个`Box<T>`或者使用`mem::swap`，`Pin<P>`包装了指针P，因此`Pin<Box<T>>`运行与通常的`Box<T>`非常相似:当`Pin<Box<T>>`被丢弃，它的内容也被丢弃，内存被释放。类似地，`Pin<&mut T>`与`&mut T`非常相像。然后，`Pin<P>` 不允许客户代码获取`Box<T>` 或 `&mut T`来访问被“钉住”的数据，这暗示着你将不能进行类似`mem::swap`的操作:

```rust
use std::pin::Pin;
fn swap_pins<T>(x: Pin<&mut T>, y: Pin<&mut T>) {
    // `mem::swap` 需 `&mut T`, 但是我们无法得到它.
    // 伤心了。。。。我们无法交换这些引用的内容
    // 我们仍旧可以使用`Pin::get_unchecked_mut`, 但是因为以下原因，这个是不安全的:
    // 不允许使用从Pin中移出的东西。
}
```

值得重申的是，`Pin<P>` 并不改变`Rust`编译器把所有的类型都考虑成可移动的这个事实。`mem::swap`仍旧对于任何类型都是可调用的。 相反`Pin<P>`防止了一些(由 `Pin<P>`指针所指定)特定值被需要`&mut T`为参数的函数调用所移动(就像`mem::swap`)。 ** 理解说明，`Pin<P>`隐藏了`&mut T` **

`Pin<P>` 可以用来包装任何指针`P`,因此它影响了`Deref`和`DerefMut`。对于`Pin<P>`,`P: Deref`应该被看成一个指向`P::Target`的"P-类型指针" -- 因此, `Pin<Box<T>>`是一个具有所有权的指向钉住的`T`的指针,而`Pin<Rc<T>>`是一个指向钉住的`T`的引用计数。 为正确起见：`Pin<P>`依赖于`Deref` 和 `DerefMut`的函数实现并不会移出他们的self参数，并且在他们被调用时，只返回钉住的数据。

## Unpin --解钉
很多类型由于并不依赖需要固定的内存地址，因此即使已经被钉住时，也是可以自由移动的。 这些类型包括所有的基础类型 (例如 bool, i32, and 引用)以及仅仅由这些类型组成的类型。 不关心钉住的类型实现了Unpin（解钉）这一自动特征, 丢弃了Pin<P>的功能。 对于解钉的类型`T: Unpin`, `Pin<Box<T>> `和 `Box<T>`的功能是完全一致的,`Pin<&mut T>`和`&mut T`也是一样。

注意，钉定和解钉仅影响指向`P::Target`的类型，并不是被`Pin<P>`包含的`P`自身。例如，无论`Box<T>`是否是unpin都不会影响`Pin<Box<T>>`的行为（这里`T`是被指向的类型）。 即Pin<Box<T>>影响的是T，而不是Box<T>


## 例子：自引用的结构类型
```rust
use std::pin::Pin;
use std::marker::PhantomPinned;
use std::ptr::NonNull;

// 这是一个自引用的类型，因为slice字段指向了data字段，
// 由于这个模式无法通过普通的借用规则描述，我们无法通过通常的引用来通知编译器
// 相反，我们使用了一个原始的指针，通它我们知道 1：不是空的； 2：指向了一个字符串
struct Unmovable {
    data: String,
    slice: NonNull<String>,
    _pin: PhantomPinned,
}

impl Unmovable {
    //为了保证在函数返回时，数据不会被移动，我们把它放在堆里，这样生命周期与对象的一致，唯一的访问方式是通过一个指向它的指针：
    fn new(data: String) -> Pin<Box<Self>> {
        let res = Unmovable {
            data,
            // 我们仅仅在数据已经在的时候才能创建指针，否则可以在刚开始的时候，指针就已经变了
            slice: NonNull::dangling(),
            _pin: PhantomPinned,
        };
        //pin数据
        let mut boxed = Box::pin(res);
        //取数据指针
        let slice = NonNull::from(&boxed.data);
        // 我们知道这里是安全的，因此修改一个字段并不会导致结构被移动
        unsafe {
            let mut_ref: Pin<&mut Self> = Pin::as_mut(&mut boxed);
            Pin::get_unchecked_mut(mut_ref).slice = slice;
        }
        boxed
    }
}

let unmoved = Unmovable::new("hello".to_string());
// The pointer should point to the correct location,
// so long as the struct hasn't moved.
// Meanwhile, we are free to move the pointer around.
// 只要结构还没有移动，指针都应该指向正确的位置，与此同时，我们可以自由地移动包裹了（这个结构的）指针。
let mut still_unmoved = unmoved;
assert_eq!(still_unmoved.slice, NonNull::from(&still_unmoved.data));

// 因为我们的类型没有实现Unpin，这个将会在编译的时候导致失败:
// let mut new_unmoved = Unmovable::new("world".to_string());
// std::mem::swap(&mut *still_unmoved, &mut *new_unmoved);
```
## 例子，侵入性的双向链表
在一个侵入性的双向链表中，集合并不为实际的元素自身分配内存。 内存分配由客户代码控制，元素可能在栈中并且比集合自身存活的短。

为了使这个集合工作，每个元素都有一个指向它的前线和后续元素的指针。 元素仅仅在它们已经被钉住的情况下才可以加入，因为移动元素将会导致指针失效。更进一步地，列表元素的丢弃特性的实现将会导致该元素的前一个和后一个节点把该元素从列表中删除。

关键是，我们必须依赖被调用的drop，如果元素内存可以被除了drop之外的函数释放，元素邻节点中指向该元素的指针将会失效，这将会导致数据结构被破坏。

因此，pinning必须和drop相关的保证实现一起提供


## Drop 保证

pinning的目标是允许依赖某些数据在内存中的位置。为了让这个工作，不仅是要约束数据的移动，还需要约束数据的释放、使用目的改变或者是其他的使存储数据的内存失效的操作。具体来说，对于钉住的数据，直到它被drop之前，你必须维护它的内存不变性，不会变得无效、使用目的不会改变。 内存可能会因释放而失效，也可以被使用`None`的`Some(v)`调用替换，或者调用`Vec::set_len`来清除一个数组中的一些元素； 也有可能通过在结构被析构之前使用`ptr::write`覆盖原有的数据来改变使用目的。
 
这正是上一节所述的侵入式链表能够正常工作所需要保证的类型。

注意，这些保证并不意味着内存不会泄漏！ 从来不在一个钉住的元素上调用drop是完全可以的（比如，您仍旧可以在`Pin<Box<T>>`上调用`mem::forget`）。在双向链中的例子中，元素将保留在列表中，然而不调用drop，您将无法释放或是重用存储空间


## <span id="dropimplemnt">Drop 实现</span>
如果您使用了钉住（如上面两个例子）类型，你必须在实现Drop时候非常小心。drop函数使用`&mut self`，但它有可能在类型已经被钉住后被调用！就象在编译器自动调用`Pin::get_unchecked_mut`就是这样`Pin<&mut Self>`。

由于为类型实现钉住需要使用非安全代码，这个在安全代码里将永远不会成为问题，但是要注意如果使用了钉住自己的内容(例如：`Pin<&Self>`或`Pin<&mut Self>`)将会导致你自己的Drop实现后果：如果你钉住了一个类型，你必须隐式地认为Drop的实现使用了：
This can never cause a problem in safe code because implementing a type that relies on pinning requires unsafe code, but be aware that deciding to make use of pinning in your type (for example by implementing some operation on Pin<&Self> or Pin<&mut Self>) has consequences for your Drop implementation as well: if an element of your type could have been pinned, you must treat Drop as implicitly taking Pin<&mut Self>.

例如, 你可以用如下的方式实现`Drop`:
```rust
impl Drop for Type {
    fn drop(&mut self) {
        //因为这个值从来没有用过，所有`new_unchecked`是可以的
        // again after being dropped.
        inner_drop(unsafe { Pin::new_unchecked(self)});
        fn inner_drop(this: Pin<&mut Type>) {
            // 实际的丢弃动作
        }
    }
}
```
函数`inner_drop`具有drop所应该具有的类型, 因此，请确保您不会以与pinning冲突的方式意外使用self。

更进一步地，如果你的类型使用了#[repr(packed)], 编译器将自动移动相应的字段来实现drop,而这甚至有可能在字段充分对齐的情况下发生，这个导致的后果是你不能试图钉住`#[repr(packed)]`类型

## 字段投射和结构化钉定
当工作在钉住的结构上时，浮现另外一个问题：在函数中如何访问` Pin<&mut Struct>`的字段。常用的方案是写相应的辅助函数（因此称为投射）把` Pin<&mut Struct>`转化为该字段的引用，但是该引用应该是什么类型呢？是`Pin<&mut Field> `还是`&mut Field`? 同样的问题也会在访问枚举中的字段时产生，在考虑容器类型或是包裹类型，例如`Vec<T>`, `Box<T>` 或 `RefCell<T>`时也会产生同样的问题。这个问题同样适用于可变和共享的引用，我们此处使用了更常用的可变引用作说明。

实际上，完全由数据结构的作者决定把钉住的对象`Pin<&mut Struct>` 投射成`Pin<&mut Field>`还是`&mut Field`， 不过还是有一些约束，最主要的约束是一致性：任意一个字段都可以被投射成钉住的引用，或者解钉后的实际内容。如果对同一个字段实现了两个，似乎不太合理哦！

作为数据结构创建者的你决定是否需要对每一个字段的pinning进行传播，具有传播性的pinning同样被称为“结构化的”，因为它可以跟随类型的结构，在下面的章节中，我们描述了每一种选择的考虑因素。


### 字段非结构化钉定
钉定结构的某个字段可能是非钉定的，这个看起来有点反直觉，但这是个容易的选择：如果`Pin<&mut Field>`从未被创建，也没什么错！因此，如果你决定某个字段没有结构化钉定，你所需要确信的是你从来不会创建这个字段的钉定。

没有结构化钉定的字段也可以有一个投射方法，把`Pin<&mut Struct> `转化为`&mut `字段


```rust
impl Struct {
    fn pin_get_field(self: Pin<&mut Self>) -> &mut Field {
        // 考虑到`field`从来不会被钉定，这是可以的
        unsafe { &mut self.get_unchecked_mut().field }
    }
}
```
假如类型从来不会创建`Pin<&mut Field>`，因此与钉定无关，此时即使结构内的字段的类型不是Unpin类型的，也可以为这个结构实现Unpin。

### 结构化字段钉定
另外一个选项是字段的钉定是结构化的，意思是当结构钉定时同样钉定字段。

这将允许写一个创建`Pin<&mut Field>`的投射函数，说明该字段时钉定的：
```rust
impl Struct {
    fn pin_get_field(self: Pin<&mut Self>) -> Pin<&mut Field> {
        // "field"将会在"self"被钉定的时候钉定，所以这是可以的。
        unsafe { self.map_unchecked_mut(|s| &mut s.field) }
    }
}
```
然而，结构化钉定带来一些额外的要求：

默认地，结构必须在所有的字段都已经解钉时才解钉。但“解钉”是安全的特征，这样结构体的编写者有责任不要给`Struct<T>`添加类似`impl<T> Unpin `这样的实现（注意添加投射操作需要使用不安全的代码，事实上`解钉`是安全的特征，不会打破上述原则，你仅需关心这些你不安全地使用的部分）。

结构的析构器不允许在将结构化的字段移出其参数之外，这正是前面章节中所提出的那个问题：drop获取了`&mut self`，但是结构（以及其字段）可能在前面被"钉定"了。你必须保证你不会在你的Drop的实现中移动字段。特别地，正如前面所解释的，这意味着你的结构不允许是`#[repr(packed)]`。 要编写一个让编译器帮助你不会意味打破“钉定"的代码，请查阅[该章](#span-id%22dropimplemnt%22drop-%e5%ae%9e%e7%8e%b0span)

你必须确认你拥有"drop"保证：一旦你的结构被“钉定”，容纳该内容的内存不会被覆盖或是被非析构函数释放。这有一点棘手，如`VecDeque<T>`所示: `VecDeque<T>`的析构函数对所有的字段进行析构，如果其中某一字段的析构函数panic了，该析构函数将会失败。因此这会导致元素在没有调用其析构函数时被释放（结构的析构函数被调用了，而某些字段的析构函数没有被调用，结构被释放意味着字段被释放），违反了`Drop`保证。(`VecDeque<T> `没有字段投射函数，因此不会导致不完整）。

你必须不提供任何可能会导致被“钉住”的结构中数据被移出结构的操作。比如，如果结构包含 `Option<T>`并且有一个类似"take"的操作--`fn(Pin<&mut Struct<T>>) -> Option<T>`, 该操作将导致T被移动到钉定的结构`Struct<T>`之外 -- 这意味着不能使用结构化的钉定来保有这个数据。

一个更复杂的将数据移出被钉定的结构的例子：想象一下如果` RefCell<T>`有一个方法`fn get_pin_mut(self: Pin<&mut Self>) -> Pin<&mut T>`。我可以做如下的动作：


```rust
fn exploit_ref_cell<T>(rc: Pin<&mut RefCell<T>>) {
    { let p = rc.as_mut().get_pin_mut(); } // Here we get pinned access to the `T`.
    let rc_shr: &RefCell<T> = rc.into_ref().get_ref();
    let b = rc_shr.borrow_mut();
    let content = &mut *b; // And here we have `&mut T` to the same data.
}
```

这是灾难性的，它意味着我们可以首先钉定` RefCell<T> (using RefCell::get_pin_mut)`的内容，然后使用后续的可变引用来移动它！


## 例子
对于类似Vec<T>的类型, 所有的可能性（结构化或非结构化钉定）都有意义。具有结构化钉定的Vec<T>可以有`get_pin/get_pin_mut` 方法来获得元素的钉定后引用。然后它不允许在一个钉定的Vec<T>上调用`pop`,因为这将导致移动（钉定化结构）的元素！也不允许`push`, 这有可能会重新分配内存导致内容移动。

一个非结构化钉定的`Vec<T>`可以实现`impl<T> Unpin`，由于内容是从来不钉定的并且`Vec<T>`自身也可以移动，从这点来看钉定对这个向量没有任何作用。

在标准库中，指针类型通常没有结构化的钉定，因此他们不会提供钉定投射函数，这就是为什么`Box<T>: Unpin`保有了`T`的一切.这一切对于指针类型是有意义的，因为移动`Box<T>`并不会移动实际的`T`: 即使`T`不能，`Box<T>`仍然可以自由移动(所谓的：解钉)。实际上，即使是`Pin<Box<T>>` 和 `Pin<&mut T>`本身也总是`Unpin`特征的, 基于同样的原因： 它们的内容(the T)是钉定的，但是指针本身可以在不移动钉定数据的情况移动。对于`Box<T>` 和 `Pin<Box<T>>`, 内容是否钉定和指针是否钉定是完全独立的，意味着这是非结构化钉定。

当和`Future`组合实现时,因为你需要获得钉定的引用来调用`poll`,嵌套的futures可能需要结构化的钉定。但是如果你的组合含有任何不需要“钉定”的数据时，你可以使这些字段非结构化钉定，从而可以即合我在只有`Pin<&mut Self>`的时候，自由地使用一个可变引用来访问他们。(例如在你自己的`poll`实现里).


# 扩展阅读

### [rust async:pin 概念解析](https://zhuanlan.zhihu.com/p/67803708)

### [关于future的一个很好的说明](https://stevenbai.top/rust/futures_explained_in_200_lines_of_rust/)