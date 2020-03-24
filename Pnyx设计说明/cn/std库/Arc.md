# 结构std :: sync :: Arc
[ + ]  显示声明
[ - ]
线程安全的引用计数指针。" Arc"代表"原子引用计数"。

该类型$Arc<T>$提供T在堆中分配的类型值的共享所有权。调用Arc的clone将产生一个新Arc实例，该实例指向堆上与源Arc相同的分配，同时增加引用计数。当指向所分配的最后一个Arc指针被销毁时，存储在该分配中的值（通常称为"内部值"）也会被删除。

默认情况下，Rust中的共享引用不允许更改，Arc也不例外：您通常无法获得对内的某些东西的可变引用Arc。如果对Arc的值变化，请使用 Mutex，RwLock或者Atomic 类型。

## 线程安全
与${Rc<T>}$不同，$Arc<T>$使用原子操作进行引用计数。这意味着它是线程安全的。缺点是原子操作比普通的内存访问更昂贵。如果您不共享线程之间的引用计数分配，请考虑使用$Rc<T>$以降低开销。$Rc<T>$是安全的默认设置，因为编译器将捕获$Rc<T>$在线程之间发送的任何尝试。但是，库实现可能会选择$Arc<T>$为了给库使用者更多的灵活性。

只要T实现了 `Send`和`Sync`，$Arc<T>$将同样实现`Send`和`Sync`。为什么你不能把一个非线程安全类T放到Arc<T>，使其成为线程安全的？起初这可能有点违反直觉：毕竟`Arc<T>`线程安全不是重点吗？关键在于：`Arc<T>`使具有相同数据的多个所有权成为线程安全的，但不会为其数据增加线程安全。考虑一下`Arc<RefCell<T>>` 。RefCell<T>不是`Sync`，如果`Arc<T>`一直是`Send`类型 ，`Arc<RefCell<T>>`也会一样（线程安全）。然而我们会遇到一个问题： RefCell<T>不是线程安全的，它使用非原子操作跟踪借入计数。

最终，这意味着你可能需要配对Arc<T>使用某种 std::sync类型，通常Mutex<T>。

打破周期 Weak
该downgrade方法可用于创建非所有者 Weak指针。Weak可以将upgraded 作为指向的指针Arc，但是None如果已删除分配中存储的值，则该指针将返回。换句话说，Weak指针不会使分配内部的值保持活动状态。但是，它们的确使分配（值的后备存储）保持活动状态。

Arc指针之间的循环永远不会被释放。因此， Weak用于中断周期。例如，一棵树可以具有Arc从父节点到子节点的强指针，以及从子节点到其父节点的Weak 指针。

克隆参考
使用Clone为Arc<T>和实现的特征从现有引用计数指针创建新引用 Weak<T>。

使用 std :: sync :: Arc ;
让 FOO  =  弧 :: 新（VEC ！ [ 1.0，2.0，3.0 ]）;
//以下两种语法是等效的。
让 a  =  foo。克隆（）;
让 b  =  Arc :: clone（＆foo）;
// A，B，和Foo都是圆弧该点到同一存储器位置运行
Deref 行为
Arc<T>自动取消引用T（通过Deref特征），因此您可以T在type值上调用的方法Arc<T>。为了避免与T的方法发生名称冲突，其Arc<T>自身的方法是关联的函数，使用类似于函数的语法进行调用：

使用 std :: sync :: Arc ;
让 my_arc  =  Arc :: new（（））;

Arc :: 降级（＆my_arc）; 跑
Weak<T>不会自动取消引用T，因为内部值可能已被删除。

例子
在线程之间共享一些不可变的数据：

使用 std :: sync :: Arc ;
使用 std :: thread ;

令 5  =  Arc :: new（5）;

for  _  in  0 .. 10 {
     让 5  =  Arc :: clone（＆五）;

    线程 :: 生成（move  | | {
         println ！（" {：？}"，五）;
    }）;
} 运行
共享可变的AtomicUsize：

使用 std :: sync :: Arc ;
使用 std :: sync :: atomic :: { AtomicUsize，Ordering };
使用 std :: thread ;

let  val  =  Arc :: new（AtomicUsize :: new（5））;

for  _  in  0 .. 10 {
     let  val  =  Arc :: clone（＆val）;

    螺纹 :: 重生（移动 | | {
         让 v  =  VAL。fetch_add（1，订货 :: SeqCst）;
         的println ！（"{：}？" ，v）;
    }）;
} 运行
请参阅rc文档以获取更多一般的引用计数示例。