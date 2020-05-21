# 特性 libp2p_core ::transport ::Transport
[ + ]  显示声明
```rust
pub trait Transport {
    type Output;
    type Error: Error;
    type Listener: Stream<Item = Result<ListenerEvent<Self::ListenerUpgrade, Self::Error>, Self::Error>>;
    type ListenerUpgrade: Future<Output = Result<Self::Output, Self::Error>>;
    type Dial: Future<Output = Result<Self::Output, Self::Error>>;
    fn listen_on(
        self, 
        addr: Multiaddr
    ) -> Result<Self::Listener, TransportError<Self::Error>>
    where
        Self: Sized;
    fn dial(
        self, 
        addr: Multiaddr
    ) -> Result<Self::Dial, TransportError<Self::Error>>
    where
        Self: Sized;

    fn boxed(self) -> Boxed<Self::Output, Self::Error>
    where
        Self: Sized + Clone + Send + Sync + 'static,
        Self::Dial: Send + 'static,
        Self::Listener: Send + 'static,
        Self::ListenerUpgrade: Send + 'static,
    { ... }
    fn map<F, O>(self, f: F) -> Map<Self, F>
    where
        Self: Sized,
        F: FnOnce(Self::Output, ConnectedPoint) -> O + Clone,
    { ... }
    fn map_err<F, E>(self, f: F) -> MapErr<Self, F>
    where
        Self: Sized,
        F: FnOnce(Self::Error) -> E + Clone,
    { ... }
    fn or_transport<U>(self, other: U) -> OrTransport<Self, U>
    where
        Self: Sized,
        U: Transport,
        <U as Transport>::Error: 'static,
    { ... }
    fn and_then<C, F, O>(self, f: C) -> AndThen<Self, C>
    where
        Self: Sized,
        C: FnOnce(Self::Output, ConnectedPoint) -> F + Clone,
        F: TryFuture<Ok = O>,
        <F as TryFuture>::Error: Error + 'static,
    { ... }
    fn timeout(self, timeout: Duration) -> TransportTimeout<Self>
    where
        Self: Sized,
    { ... }
    fn outbound_timeout(self, timeout: Duration) -> TransportTimeout<Self>
    where
        Self: Sized,
    { ... }
    fn inbound_timeout(self, timeout: Duration) -> TransportTimeout<Self>
    where
        Self: Sized,
    { ... }
    fn upgrade(self, version: Version) -> Builder<Self>
    where
        Self: Sized,
        Self::Error: 'static,
    { ... }
}
```
[ - ]
Transport实现面向连接的在两个对等点之间传输有序的数据流。

连接通过监听或在Transport上拨号建立。通过侦听获得连接的对等方通常称为侦听器，而通过拨号启动连接的对等方称为拨号器，对应传统的服务器和客户端的传统角色。

大多数传输在已建立的连接上提供可靠的数据传输保证，但是这些保证的精确语义取决于特定的传输实现方案。

此特性是针对具体的面向连接的传输协议（如TCP或Unix域套接字）实现的，但也适用于为拨号或侦听过程添加其他功能（例如，通过DNS进行名称解析）的包装器。

更多的协议可以分层建立在由upgrade初始化的具有升级机制Transport上。

    注意：使用此特征的方法self不使用&self或&mut self。换句话说，收听或拨号会消耗传输对象。这样做的目的是使您可以在&Foo或&mut Foo上，而不是直接在Foo上实现此特征。

## 关联类型
type Output
[ - ]
连接设置过程的结果，包括协议升级。

通常，输出至少包含数据流的句柄（即连接或连接顶部的子流多路复用器），该句柄提供用于通过连接发送和接收数据的API。

type Error: Error
[ - ]
连接建立期间发生的错误。

type Listener: Stream<Item = Result<ListenerEvent<Self::ListenerUpgrade, Self::Error>, Self::Error>>
[ - ]
Output入站连接的s 流。

只要在运输堆栈的最低层收到连接，就应生产一个物品。该项目必须是在应用所有协议升级ListenerUpgrade后能够解析为Output值的未来。

如果此流产生错误，则认为它是致命的，并且侦听器被杀死。通过产生，可以报告非致命错误ListenerEvent::Error。

type ListenerUpgrade: Future<Output = Result<Self::Output, Self::Error>>
[ - ]
Output从Listener流获得的入站连接的挂起。

传输接受连接后，可能需要进行异步后处理（即协议升级协商）。这样的后处理不应阻止Listener产生下一个连接，因此进一步的连接设置将异步进行。一旦ListenerUpgrade将来解决它产生的Output 连接建立过程。

type Dial: Future<Output = Result<Self::Output, Self::Error>>
[ - ]
Output从拨出获得的待定的出站连接。

所需方法
fn listen_on(
    self,
    addr: Multiaddr
) -> Result<Self::Listener, TransportError<Self::Error>>
where
    Self: Sized, 
[ - ]
Multiaddr侦听给定的，产生未决的入站连接流，并处理此传输正在侦听的地址（参见参考资料ListenerEvent）。

从流中返回错误被认为是致命的。侦听器还可以通过生成来报告非致命错误ListenerEvent::Error。

fn dial(
    self,
    addr: Multiaddr
) -> Result<Self::Dial, TransportError<Self::Error>>
where
    Self: Sized, 
[ - ]
拨打给定的Multiaddr，返回将来的未决出站连接。

如果TransportError::MultiaddrNotSupported返回，则可能希望尝试其他方法Transport（如果有）。

提供的方法
fn boxed(self) -> Boxed<Self::Output, Self::Error>
where
    Self: Sized + Clone + Send + Sync + 'static,
    Self::Dial: Send + 'static,
    Self::Listener: Send + 'static,
    Self::ListenerUpgrade: Send + 'static, 
[ - ]
将传输转换为抽象的盒装（即堆分配）传输。

fn map<F, O>(self, f: F) -> Map<Self, F>
where
    Self: Sized,
    F: FnOnce(Self::Output, ConnectedPoint) -> O + Clone, 
[ - ]
在传输创建的连接上应用功能。

fn map_err<F, E>(self, f: F) -> MapErr<Self, F>
where
    Self: Sized,
    F: FnOnce(Self::Error) -> E + Clone, 
[ - ]
将函数应用于由运输期货产生的错误。

fn or_transport<U>(self, other: U) -> OrTransport<Self, U>
where
    Self: Sized,
    U: Transport,
    <U as Transport>::Error: 'static, 
[ - ]
添加后备传输，该备用传输在建立入站或出站连接时遇到错误时使用。

返回的传输将类似于self，但如果listen_on或dial 返回错误other则将尝试。

fn and_then<C, F, O>(self, f: C) -> AndThen<Self, C>
where
    Self: Sized,
    C: FnOnce(Self::Output, ConnectedPoint) -> F + Clone,
    F: TryFuture<Ok = O>,
    <F as TryFuture>::Error: Error + 'static, 
[ - ]
将产生异步结果的函数应用于由此传输创建的每个连接。

此功能可用于临时协议升级或用于以下配置的处理或调整输出。

有关高级传输升级过程，请参阅Transport::upgrade。

fn timeout(self, timeout: Duration) -> TransportTimeout<Self>
where
    Self: Sized, 
[ - ]
将通过传输建立的所有入站和出站连接的超时添加到连接设置（包括升级）中。

fn outbound_timeout(self, timeout: Duration) -> TransportTimeout<Self>
where
    Self: Sized, 
[ - ]
对于通过传输建立的所有出站连接，将超时添加到连接设置（包括升级）中。

fn inbound_timeout(self, timeout: Duration) -> TransportTimeout<Self>
where
    Self: Sized, 
[ - ]
对于通过传输建立的所有入站连接，将超时添加到连接设置（包括升级）中。

fn upgrade(self, version: Version) -> Builder<Self>
where
    Self: Sized,
    Self::Error: 'static, 
[ - ]
开始通过进行一系列协议升级 upgrade::Builder。

实施者
impl Transport for MemoryTransport
[src]
[ - ]
[ + ]显示隐藏的未记录项目
type Output = Channel<Vec<u8>>
type Error = MemoryTransportError
type Listener = Listener
type ListenerUpgrade = Ready<Result<Self::Output, Self::Error>>
type Dial = DialFuture
impl<A, B> Transport for EitherTransport<A, B>
where
    B: Transport,
    A: Transport, 
[src]
[ - ]
[ + ]显示隐藏的未记录项目
type Output = EitherOutput<A::Output, B::Output>
type Error = EitherError<A::Error, B::Error>
type Listener = EitherListenStream<A::Listener, B::Listener>
type ListenerUpgrade = EitherFuture<A::ListenerUpgrade, B::ListenerUpgrade>
type Dial = EitherFuture<A::Dial, B::Dial>
impl<A, B> Transport for OrTransport<A, B>
where
    B: Transport,
    A: Transport, 
[src]
[ - ]
[ + ]显示隐藏的未记录项目
type Output = EitherOutput<A::Output, B::Output>
type Error = EitherError<A::Error, B::Error>
type Listener = EitherListenStream<A::Listener, B::Listener>
type ListenerUpgrade = EitherFuture<A::ListenerUpgrade, B::ListenerUpgrade>
type Dial = EitherFuture<A::Dial, B::Dial>
impl<InnerTrans> Transport for TransportTimeout<InnerTrans>
where
    InnerTrans: Transport,
    InnerTrans::Error: 'static, 
[src]
[ - ]
[ + ]显示隐藏的未记录项目
type Output = InnerTrans::Output
type Error = TransportTimeoutError<InnerTrans::Error>
type Listener = TimeoutListener<InnerTrans::Listener>
type ListenerUpgrade = Timeout<InnerTrans::ListenerUpgrade>
type Dial = Timeout<InnerTrans::Dial>
impl<O, E> Transport for Boxed<O, E>
where
    E: Error, 
[src]
[ - ]
[ + ]显示隐藏的未记录项目
type Output = O
type Error = E
type Listener = Listener<O, E>
type ListenerUpgrade = ListenerUpgrade<O, E>
type Dial = Dial<O, E>
impl<T> Transport for OptionalTransport<T>
where
    T: Transport, 
[src]
[ - ]
[ + ]显示隐藏的未记录项目
type Output = T::Output
type Error = T::Error
type Listener = T::Listener
type ListenerUpgrade = T::ListenerUpgrade
type Dial = T::Dial
impl<T, C, D, U, I, E> Transport for Upgrade<T, U>
where
    T: Transport<Output = (I, C)>,
    T::Error: 'static,
    C: AsyncRead + AsyncWrite + Unpin,
    U: InboundUpgrade<Negotiated<C>, Output = D, Error = E>,
    U: OutboundUpgrade<Negotiated<C>, Output = D, Error = E> + Clone,
    E: Error + 'static, 
[src]
[ - ]
[ + ]显示隐藏的未记录项目
type Output = (I, D)
type Error = TransportUpgradeError<T::Error, E>
type Listener = ListenerStream<T::Listener, U>
type ListenerUpgrade = ListenerUpgradeFuture<T::ListenerUpgrade, U, I, C>
type Dial = DialUpgradeFuture<T::Dial, U, I, C>
impl<T, C, F, O> Transport for AndThen<T, C>
where
    T: Transport,
    C: FnOnce(T::Output, ConnectedPoint) -> F + Clone,
    F: TryFuture<Ok = O>,
    F::Error: Error,   
[src]  
[ - ]  
[ + ]显示隐藏的未记录项目  
type Output = O  
type Error = EitherError<T::Error, F::Error>
type Listener = AndThenStream<T::Listener, C>
type ListenerUpgrade = AndThenFuture<T::ListenerUpgrade, C, F>  
type Dial = AndThenFuture<T::Dial, C, F>  
impl<T, F, D> Transport for Map<T, F>  
where  
    T: Transport,  
    F: FnOnce(T::Output, ConnectedPoint) -> D + Clone,   
[src]  
[ - ]  
[ + ]显示隐藏的未记录项目  
type Output = D  
type Error = T::Error  
type Listener = MapStream<T::Listener, F>  
type ListenerUpgrade = MapFuture<T::ListenerUpgrade, F>
type Dial = MapFuture<T::Dial, F>
impl<T, F, TErr> Transport for MapErr<T, F>
where
    T: Transport,
    F: FnOnce(T::Error) -> TErr + Clone,
    TErr: Error, 
[src]
[ - ]
[ + ]显示隐藏的未记录项目
type Output = T::Output
type Error = TErr
type Listener = MapErrListener<T, F>
type ListenerUpgrade = MapErrListenerUpgrade<T, F>
type Dial = MapErrDial<T, F>
impl<TOut> Transport for DummyTransport<TOut>
[src]
[ - ]
[ + ]显示隐藏的未记录项目
type Output = TOut
type Error = Error
type Listener = Pending<Result<ListenerEvent<Self::ListenerUpgrade, Self::Error>, Self::Error>>
type ListenerUpgrade = Pending<Result<Self::Output, Error>>
type Dial = Pending<Result<Self::Output, Error>>