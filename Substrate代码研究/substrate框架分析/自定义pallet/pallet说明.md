
# Substrate 存储数据类型概览 ‘decl_storage’

## 存储类型
> *	在Substrate中，使用Runtime Storage可以将数据存储在区块链中。通过Substrate的存储API，runtime模块可以容易地这些数据可以从runtime逻辑中访问，并被持久化在区块中。
> *	可以使用宏decl_storage!创建新的runtime存储项。该宏会根据你的定义生成相关API，操作存储项，声明每种类型的存储项的示例：
> 
````rust
decl_storage! {
    trait Store for Module<T: Trait> as Example {
        pub SomeValue: u64;
        pub SomeMap: map u64 => u64;
        pub SomeLinkedMap: linked_map u64 => u64;
        pub SomeDoubleMap: double_map u64, blake2_256(u64) => u64;
    }
}
````

### 存储项的完整定义格式
>   $vis $name get(fn $getter) config($field_name) build($closure): $type = $default;
  
-  $vis：设置结构体的可见性。pub或不设置。
-  $name：存储项的名称，用作存储中的前缀。
-  get(fn $getter)：可选，实现Module的名称为$getter的函数,可以直接Self::$getter()获取。
-  config($field_name)：可选，定义可在GenesisConfig中包含该存储项。如果设置了get，则field_name是可选的。
-  build($closure)：可选，定义闭包，调用它可以覆盖存储项的值。
-  $type：存储项的类型。
-  $default：定义默认值，即无值时的返回值。


### 基本的存储项由名称和类型组成，常见的存储类型：
- Value，单值类型，可用来存储某种单一类型的值，如布尔，数值，枚举，结构体等。
- Map，简单映射类型，类型标识为map，可以存储键值对，通过key可以索引到value，并进行相应的修改。
- Linked Map，链接映射类型，类型标识为linked_map，和map类型类似，也是用于存储键值对，不同的是linked_map可以对所有的键值对进行遍历操作，而map目前只能对值（value）进行遍历，不能遍历所有的键（key） 。
- Double Map，双键映射类型，类型标识为double_map，顾名思义，两个key，对应一个value，主要目的是通过第一个键（key 1）快速删除任意key 2的记录，也可以遍历key 1对应的所有的值。

#### 单值类型
* 使用方法

```rust
//增：
MyUnsignedNumber::put(number);
//查：
let my_unsigned_num = MyUnsignedNumber::get();
//改：
MyUnsignedNumber::mutate(|v| v + 1);
//删：
MyUnsignedNumber::kill();
```

#### 简单映射类型

```rust
// 插入一个元素
MyMap::insert(key, value);

// 通过key获取value
MyMap::get(key);

// 删除某个key对应的元素
MyMap::remove(key);

// 覆盖或者修改某个key对应的元素
MyMap::insert(key, new_value);
MyMap::mutate(key, |old_value| old_value+1);
```

#### 链接映射类型
增删改查和简单映射一样，多了遍历方法

```rust
// 遍历键值对
let result: Vec<(u8, Vec<u8>)> = MyLinkedMap::enumerate()
	.filter(|(k, _)| k > &10)
	.collect();
```

#### 双键映射类型
* 使用方法

```rust
let sender = ensure_signed(origin)?; // 获取key1

// 插入一个元素
MyDoubleMap::<T>::insert(sender, key2, value); // key2为参数传入

// 获取某一元素
MyDoubleMap::<T>::get(sender, key2);

// 删除某一元素
MyDoubleMap::<T>::remove(sender, key2);

// 删除key对应的所有元素
MyDoubleMap::<T>::remove_prefix(sender);
```

# 自定义EVENT ‘decl_event’

## 普通定义

## 范形定义

## 调用event

```
 	Self::deposit_event(RawEvent::RecoveryCreated(who));
```

# 自定义错误 ERROR ‘decl_error’

## 定义ERROR

```
 use frame_support::{decl_error, decl_module};
 
 decl_error! {

     pub enum MyError for Module<T: Trait> {

         MyCoolErrorMessage,

         YouAreNotCoolEnough,
     }
}
```

## 使用

```
     pub struct Module<T: Trait> for enum Call where origin: T::Origin {
         type Error = MyError<T>;

        #[weight = 0]
         fn do_something(origin) -> frame_support::dispatch::DispatchResult {
             Err(MyError::<T>::YouAreNotCoolEnough.into())
        }
     }
```

# pallet 定义 ‘decl_module’

# 测试自定义pallet  及相关maroc定义说明

## 创建‘impl_outer_origin’
