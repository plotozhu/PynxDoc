# 说明
Client对应有两个FullClient和LightClient，对应有两个Executor。 从代码里看，直接使用了executor的两个函数：
* `executor.runtime_version(id)`返回运行时的版本
* `executor.contextual_call` 执行操作
  
另外使用executor作为参数生成了`call_executor`对象
* `let call_executor = LocalCallExecutor::new(backend.clone(), executor, spawn_handle);`

* 而这里的executor是实际的交易执行的对象：
```rust
	let executor = NativeExecutor::<TExecDisp>::new(
		config.wasm_method,
		config.default_heap_pages,
		config.max_runtime_instances,
    );
```
使用了config中的三个参数来创建一个NativeExecutor对象，其中：  
>    * wasm_method：         是指令的执行方式，执行方式有两种‘解释型’和‘编译型’  
>    * default_heap_pages：  是默认分配的堆的大小，是64KB为一个单位（页）  
>    * max_runtime_instances： 同时允许的runtime个数  

* `NativeExecutor`是通用的`CodeExecutor`的实现，它来确定wasm中代码对应的代理，并且分派给native代码执行，如果无法执行，使用`WasmExecutor`来执行
```rust
pub struct NativeExecutor<D> {
	/// Dummy field to avoid the compiler complaining about us not using `D`.
	_dummy: std::marker::PhantomData<D>,
	/// Native runtime version info.
	native_version: NativeVersion,
	/// Fallback wasm executor.
	wasm: WasmExecutor,
}
```
通过new生成一个NativeExecutor执行的对象，其过程如下：
> 1. 获取sp_io::SubstrateHostFunctions获取一个`HostFunctions`对象
> 2. 通过extend函数，将`D::ExtendHostFunctions::host_functions()`中获取的函数填充到host_functions中
> 3. 使用这个扩展后的`HostFunctions`对象来创建wasm_executor
> 4. 用`wasm_executor`来创建`NativeExecutor`对象，并且返回此对象

**此处的D就是ServiceBuilder中的`TExecDisp`，即下面所述的[NativeExecutionDispatch](#span-id%22nativeexecutiondispatch%22nativeexecutiondispatchspan)**

* HostFunctions是一组Function对象的集合的接口，用于获取所有的Function对象：
```rust
fn host_functions() -> Vec<&'static dyn Function>;
```

* Function是一个wasm函数的抽象特征：
```rust 
pub trait Function: MaybeRefUnwindSafe + Send + Sync {
	/// 函数的名称
	fn name(&self) -> &str;
	/// 函数的签名
	fn signature(&self) -> Signature;
	/// 给定上下文和参数，函数的执行过程
	fn execute(
		&self,
		context: &mut dyn FunctionContext,
		args: &mut dyn Iterator<Item = Value>,
	) -> Result<Option<Value>>;
}
```

* 此处的FunctionContext是给函数和内存交互的抽象特征，实际上可以看成是内存分配和沙盒对象区取两个部分：
```rust
pub trait FunctionContext {
	/// 从内存的`address`读取一段数据到向量数组中
	fn read_memory(&self, address: Pointer<u8>, size: WordSize) -> Result<Vec<u8>> {
		let mut vec = Vec::with_capacity(size as usize);
		vec.resize(size as usize, 0);
		self.read_memory_into(address, &mut vec)?;
		Ok(vec)
	}
	/// 从内存的`address`读取一段数据到指定的`dest`数组中
	fn read_memory_into(&self, address: Pointer<u8>, dest: &mut [u8]) -> Result<()>;
	/// 向内存的 `address`写入一段数组内容
	fn write_memory(&mut self, address: Pointer<u8>, data: &[u8]) -> Result<()>;
	/// 申请 `size` 字节的内存
	fn allocate_memory(&mut self, size: WordSize) -> Result<Pointer<u8>>;
	/// 释放内存中的指针
	fn deallocate_memory(&mut self, ptr: Pointer<u8>) -> Result<()>;
	/// 沙盒对象获取.
	fn sandbox(&mut self) -> &mut dyn Sandbox;
}
```

* 沙盒对象可以看成是更原始的内存存取对象  
TODO 详细说明后续再描述
```rust 
/// Something that provides access to the sandbox.
pub trait Sandbox {
	/// Get sandbox memory from the `memory_id` instance at `offset` into the given buffer.
	fn memory_get(
		&mut self,
		memory_id: MemoryId,
		offset: WordSize,
		buf_ptr: Pointer<u8>,
		buf_len: WordSize,
	) -> Result<u32>;
	/// Set sandbox memory from the given value.
	fn memory_set(
		&mut self,
		memory_id: MemoryId,
		offset: WordSize,
		val_ptr: Pointer<u8>,
		val_len: WordSize,
	) -> Result<u32>;
	/// Delete a memory instance.
	fn memory_teardown(&mut self, memory_id: MemoryId) -> Result<()>;
	/// Create a new memory instance with the given `initial` size and the `maximum` size.
	/// The size is given in wasm pages.
	fn memory_new(&mut self, initial: u32, maximum: u32) -> Result<MemoryId>;
	/// Invoke an exported function by a name.
	fn invoke(
		&mut self,
		instance_id: u32,
		export_name: &str,
		args: &[u8],
		return_val: Pointer<u8>,
		return_val_len: WordSize,
		state: u32,
	) -> Result<u32>;
	/// Delete a sandbox instance.
	fn instance_teardown(&mut self, instance_id: u32) -> Result<()>;
	/// Create a new sandbox instance.
	fn instance_new(
		&mut self,
		dispatch_thunk_id: u32,
		wasm: &[u8],
		raw_env_def: &[u8],
		state: u32,
	) -> Result<u32>;

	/// Get the value from a global with the given `name`. The sandbox is determined by the
	/// given `instance_idx` instance.
	///
	/// Returns `Some(_)` when the requested global variable could be found.
	fn get_global_val(&self, instance_idx: u32, name: &str) -> Result<Option<Value>>;
}
```
### <span id="NativeExecutionDispatch">NativeExecutionDispatch</span>
`NativeExecutor`另外一个核心功能是根据提供的参数实现·`NativeExecutionDispatch`,`NativeExecutionDispatch`又是什么？是分派`CodeExecutor`调用的一个代理，分派意味着我们可以通过名字来执行`runtime`中的函数。
```rust
pub trait NativeExecutionDispatch: Send + Sync {
	/// 除了默认的Substrate中runtime的接口之外，还包括在runtime中调用的自定义的runtime的Host函数
	type ExtendHostFunctions: HostFunctions;

	/// 在runtime中分派一个方法，如果指定的方法名不存在，返回 `Err` 
	fn dispatch(ext: &mut dyn Externalities, method: &str, data: &[u8]) -> Result<Vec<u8>>;

	/// 原生代码的runtime的版本号
	fn native_version() -> NativeVersion;
}
```
上述中的`Externalities`顾名思义就是扩展操作的接口：链上(wasm)代码是在虚拟机中执行的，虚拟机除了提供内存/CPU（指令）外，还提供了相应的运行库供链上代码调用，在现有的实现中，除了默认的substrate的实现所提供的运行库之外，还提供了默认的扩展库（Http操作和localstorage数据读写），这些扩展操作都放在externalities里。  
当然，上述描述中还有一个ExtendHostFunctions，这个是用户自定义的提供给链上操作的api实现。这个详细内容查看下面的说明。

实现`NativeExecutionDispatch`是通过`native_executor_instance`宏来实现的
#### 例如：
```rust
sc_executor::native_executor_instance!(
    pub MyExecutor,
    substrate_test_runtime::api::dispatch,
    substrate_test_runtime::native_version,
);
```
#### 添加自定义的host函数
当你想要从你的runtime中使用自定义的runtime接口时，你需要让执行器了解这引起接口的host函数
* 这个是自定义的接口
```rust
# use sp_runtime_interface::runtime_interface;
#[runtime_interface]
trait MyInterface {
    fn say_hello_world(data: &str) {
        println!("Hello world from: {}", data);
    }
}
```
* 这个是声明方法
```rust 
sc_executor::native_executor_instance!(
    pub MyExecutor,
    substrate_test_runtime::api::dispatch,
    substrate_test_runtime::native_version,
    my_interface::HostFunctions,
);
```

这样我们就可以使用多个接口了，可以使用这样的方式来访问host函数。

`(my_interface::HostFunctions, my_interface2::HostFunctions)`


**备注** 反过来再看`bin/node/executor/src/lib.rs`和`bin/node-template/node/src/service.rs`中，都有这个定义：
```rust
// Our native executor instance.
native_executor_instance!(
	pub Executor,
	node_template_runtime::api::dispatch,
	node_template_runtime::native_version,
);
```
看起来是通过这个方式`node_template_runtime::api::dispatch`,把自定义的函数接口分派到`NativeExecutor`中。（推测这个::api来自于对应的impl_runtime_apis! 宏）