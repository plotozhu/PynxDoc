# SCALE编解码器

Rust的SCALE（简单级联聚合Little-Endian）数据格式的实现。是用于Parity 的Substrate框架中使用的类型。

SCALE是一种轻量级格式，允许进行编码（和解码），从而使其高度适用于资源受限的执行环境，例如区块链运行时和低功耗，低内存设备。

着重要注意的是，需要在编码和解码端都了解编码的上下文信息（有关类型和数据结构的信息）。编码数据中不包括此上下文信息。**说明**：即编码过程中会丢失类型和数据结构信息，比如说字段名称、结构等，使用不匹配的数据结构对内容进行解码将会导致错误，需要编解码的调用保证解码结果的正确性。（即使用正确的数据结构进行decode)。

为了更好地了解不同类型的编码方式，查看
[位于Substrate docs站点上的基本数据格式概述页面](https://substrate.dev/docs/en/overview/low-level-data-format).

## 实现

编解码器需要实现以下的特性：

###  编码

"编码"特性用于将数据编码为SCALE格式。"编码"特性包含以下功能：
* `size_hint（＆self）-> usize`：获取编码数据所需的容量（以字节为单位）。这是为了避免编码所需的内存两次分配。它可以是估计值，而不必是确切数字。如果大小未知，甚至没有很好的最大值，那么我们可以从trait实现中跳过此函数。这是用来使操作更廉价，因此不应涉及迭代等。
*  `encode_to <T：Output>（＆self，dest：＆mut T）`：对值进行编码并将其附加到目标缓冲区。
* `encode（＆self）-> Vec <u8>`：对类型数据进行编码并返回一个切片。
* `using_encoded <R，F：FnOnce（＆[u8]）-> R>（＆self，f：F）-> R`：对类型数据进行编码，并通过闭包对编码后值进行操作。返回操作结果。

**注意：** 对于值类型，实现应该重载"using_encoded"；对于分配类型应重载"encode_to"。应该尽可能对所有类型实现`size_hint`。类型的包装器应重载所有方法。

### 解码

"解码"特性用于将编码数据反序列化/解码为相应的类型。

* `fn decode <I：Input>（value：＆mut I）-> Result <Self，Error>`：尝试将值从SCALE格式解码为调用者的类型。如果解码失败，则返回`Err`。

### CompactAs

"CompactAs"特性用于将自定义类型/结构包装为紧凑类型，这使其具有更高的空间/内存效率。紧凑的编码在[此处](https://substrate.dev/docs/en/overview/low-level-data-format#compactgeneral-integers)中进行了描述。

* `encode_as（＆self）->＆Self :: As`：将类型（self）编码为紧凑类型。类型"As"是在相同特性中定义的，其实现应为可紧凑编码的。
* `decode_from（_：Self :: As）-> Self`：从紧凑的可编码类型中解码类型（self）。

### HasCompact

如果实现"HasCompact"特性，则表明相应的类型是紧凑型可编码类型。

### EncodeLike

需要为每种类型手动实现`EncodeLike`特性。使用继承时(#Derive)可以自动完成。基本上，该特性使您可以接受多种类型都编码为相同表示形式。

## 使用示例

以下是一些示例来演示编解码器的用法。

### 简单类型


```rust
//如果没有使用派生，就导入宏
# #[cfg[not(feature="derive")]]
# parity_scale_codec_derive::{Encode, Decode};

use parity_scale_codec::{Encode, Decode};
#[derive(Debug, PartialEq, Encode, Decode)]
enum EnumType {
    ＃[codec（index =" 15"）] //类型从15开始编码
    A，
    B（u32，u64），
    C {
        a：u32，
        b：u64，
    }，
}

let a = EnumType :: A;
let b = EnumType :: B（1，2）;
let c = EnumType :: C {a：1，b：2};

a.using_encoded（| ref slice | {
    assert_eq！（slice，＆b"\x0f"）;
}）;

b.using_encoded（| ref slice | {
    slice,&b"\x01\x01\0\0\0\x02\0\0\0\0\0\0\0"）；
}）;

c.using_encoded（| ref slice | {
    assert_eq！（slice，&b"\x02\x01\0\0\0\x02\0\0\0\0\0\0\0"）；
}）;

let mut da：＆[u8] = b"\x0f";
assert_eq！（EnumType :: decode（＆mut da）.ok（），Some（a））;

let mut db：＆[u8] = b"\x01\x01\0\0\0\x02\0\0\0\0\0\0\0";
assert_eq！（EnumType :: decode（＆mut db）.ok（），Some（b））;

let mut dc：＆[u8] = b"\x02\x01\0\0\0\x02\0\0\0\0\0\0\0";
assert_eq！（EnumType :: decode（＆mut dc）.ok（），Some（c））;

let mut dz：＆[u8] =＆[0];
assert_eq！（EnumType :: decode（＆mut dz）.ok（），无）;

```

 ### 具有HasCompact的压缩类型

 ```rust
 # // Import macros if derive feature is not used.
 # #[cfg(not(feature="derive"))]
 # use parity_scale_codec_derive::{Encode, Decode};

 use parity_scale_codec::{Encode, Decode, Compact, HasCompact};

 #[derive(Debug, PartialEq, Encode, Decode)]
 struct Test1CompactHasCompact<T: HasCompact> {
     #[codec(compact)]
     bar: T,
 }

 #[derive(Debug, PartialEq, Encode, Decode)]
 struct Test1HasCompact<T: HasCompact> {
     #[codec(encoded_as = "<T as HasCompact>::Type")]
     bar: T,
 }

 let test_val: (u64, usize) = (0u64, 1usize);

 let encoded = Test1HasCompact { bar: test_val.0 }.encode();
 assert_eq!(encoded.len(), test_val.1);
 assert_eq!(<Test1CompactHasCompact<u64>>::decode(&mut &encoded[..]).unwrap().bar, test_val.0);

 # fn main() { }
 ```

### 使用CompactAs的类型
```rust
 # // Import macros if derive feature is not used.
 # #[cfg(not(feature="derive"))]
 # use parity_scale_codec_derive::{Encode, Decode};

 use serde_derive::{Serialize, Deserialize};
 use parity_scale_codec::{Encode, Decode, Compact, HasCompact, CompactAs};

 #[cfg_attr(feature = "std", derive(Serialize, Deserialize, Debug))]
 #[derive(PartialEq, Eq, Clone)]
 struct StructHasCompact(u32);

 impl CompactAs for StructHasCompact {
     type As = u32;

     fn encode_as(&self) -> &Self::As {
         &12
     }

     fn decode_from(_: Self::As) -> Self {
         StructHasCompact(12)
     }
 }

 impl From<Compact<StructHasCompact>> for StructHasCompact {
     fn from(_: Compact<StructHasCompact>) -> Self {
         StructHasCompact(12)
     }
 }

 #[derive(Debug, PartialEq, Encode, Decode)]
 enum TestGenericHasCompact<T> {
     A {
         #[codec(compact)] a: T
     },
 }

 let a = TestGenericHasCompact::A::<StructHasCompact> {
     a: StructHasCompact(12325678),
 };

 let encoded = a.encode();
 assert_eq!(encoded.len(), 2);

 # fn main() { }
 ```

## 派生属性

派生实现可以支持以下属性：
- `codec（dumb_trait_bound）`：该属性需要放置在需要实现的类型上方，这将使算法让附加的特性范围收收缩到仅使用类类型参数。这个方案可以在算法的公共接口里使用私有类型。使用此属性后，你将不会再收到错误/警告提示。  

- `codec（skip）`：需要放在字段上方，这将在在编码/解码时跳过该字段。  
- `codec（compact）`：需要放在字段上方，这将使字段使用紧凑编码。（需要该类型支持紧凑编码。）  
- `codec（encoded_as（OtherType））`：需要放置在字段上方，这将使该字段使用" OtherType"进行编码。  
- `codec（index（" 0"））`：需要放在枚举变量上方，以这将使该变量在编码时使用的使用给定的索引值。默认情况下，第一个变量的索引是通过从" 0"开始算起。

<hr/>
-------- 以下是substrate网站上对SCALE 编解码的说明 -------------
<hr/>

# SCALE Codec
SCALE（Simple Concatenated Aggregate Little-Endian）编解码器是一种轻量级，高效的二进制序列化和反序列化编解码器。

它被设计用在资源受限的执行上下文，例如 Substrate runtime，能够对数据进行高性能、无拷贝的编码和解码。 它不能进行自我描述，并假定解码上下文具有有关编码数据的所有类型信息。

## Substrate 中的 SCALE
Substrate 使用 parity-scale-codec，这是 SCALE 编解码器的 Rust 实现。 该库和 SCALE 编解码器对 Substrate 和区块链系统是有益的，因为：

它更轻量，通用序列化框架如 serde 增加了样板编码，使二机制文件过大。
它不使用 Rust STD，因此可以将其编译为 Wasm，用在 Substrate runtime 之中。
它对 Rust 的支持友好，能够为新的数据类型派生出编解码逻辑。
```rust
#[derive(Encode, Decode)]
```
substrate重新定义而不使用现有的Rust既有的编码器非常重要，因此这个编解码器需要在其他不同的平台和语言上实现，这样才能实现互操作性。

## 编解码器定义
此处您将会看到SCALE编解码器如何实现不同的类型

### 固定长度整型
基本整型使用固定宽度的小端（LE）模式编码

#### 例
* 有符号的 8-bit 整数 69: 0x45  
* 无符号的 16-bit 整数 42: 0x2a00
* 无符号的 32-bit 整数 16777215: 0xffffff00
### 压缩/通用整型
一个"压缩"或是通用整型编码足以编码大整数（最高到 2<sup>536</sup>），并且比固定宽度编码更有高效，（尽管对于单字节值，固定编码并不低效）。  
这个编码方案通过两个最低位来表示数据模式：

* 0b00: 单字节模式 高6位是小端模式的值的编码（仅支持0-63）.
* 0b01: 双字节模式：高6位和下一字节是值的小端模式的编码：可以表示的值64-(2<sup>14</sup>-1)).
* 0b10: 四字节模式，高6位和后续三个字节是值的小端模式的编码：可以表示  (2<sup>14</sup>-1)-(2<sup>30</sup>-1)).
*  0b11: 大整数模式:高6位是后续字节数的长度减去4。值以小端模式存储在后续的字节中。最高字节必须不为0，值的范围为2<sup>30</sup>-1到2<sup>536</sup>-1。**说明**，最高字节是0的话，表示字节的长度数完全可以减去1，然后最高字节数就不为0了。
#### 例子
* 无符号整型0 0: 0x00
* 无符号整型 1: 0x04
* 无符号整型 42: 0xa8
* 无符号整型 69: 0x1501
#### 错误例子:
0x0100: 在模式1里进行0编码（0被编码成了模式1）。

### 布尔型
布尔型值使用一个字节中的LSB进行编码

#### 例子
布尔 false: 0x00
布尔 true: 0x01

### Options
1或是0表示特写的类型，按如下方式编码
* 0x00 内容是 None ("empty" or "null").
* 0x01 内容是 Some，后续的字节是值的编码.

这里有个特例，如果Option中的类型是布尔型，使用1个字节编码：
* 0x00 如果 None ("empty" or "null").
* 0x01 如果值是 false .
* 0x02 如果值是 true .

### 数组向量 (列表, 序列, 集合)
相同值的集合按如如下的方式编码：首先以压缩方式编码元素的个数，然后依次串联着每个元素的编码结果。

#### 例子
无符号16-bit整型：:
```
[4, 8, 15, 16, 23, 42]
```
SCALE 编码后的字节：
```
0x18040008000f00100017002a00
```

### 元组
固定大小的一组值，每一个值可能有未预定义的但是固定类型的值。这个可以简单地将每个编码后的结果串联。

#### 例子
紧凑的无符号整型和布尔型元组：
```
(3, false): 0x0c00
```

### 结构类型 
对于结构，将对值进行命名，但这与编码无关(名称将被忽略-仅顺序重要)。所有容器都连续存储元素。元素的顺序不是固定的，它取决于容器，并且不能在解码时依赖。
这隐含地意味着将某些字节数组解码为具有特定强制顺序结构，然后对其进行重新编码可能会导致与解码后的原始字节数组不同的字节数组。

#### Example
想象一个SortedVecAsc<u8>结构，它总是具有按升序排列的字节元素，而您拥有[3, 5, 2, 8]，其中第一个元素是其后的字节数（即如果仅是[3, 5, 2]无效）。

SortedVecAsc::from([3, 5, 2, 8])会解码为[3, 2, 5, 8]，这与原始编码不匹配。

### 枚举 (带标记的联合)
固定数量的变量（项），每个变量互斥，并且潜在地暗示着另一个值或一系列值

编码后的第一个字节表示该值所对应变量的索引。其余的字节数组是实际值的编码结果。 因此最多支持256个变量（项）。

#### 例子
```rust
enum IntOrBool {
  Int(u8),
  Bool(bool),
}
```

* Int(42): 0x002a
* Bool(true): 0x0101

# 实现版本
目前，Parity SCALE 编解码器的实现版本有：

* Rust: paritytech/parity-scale-codec
* Python: polkascan/py-scale-codec
* Golang: ChainSafe/gossamer
* C++: soramitsu/scale
*  JavaScript: polkadot-js/api
