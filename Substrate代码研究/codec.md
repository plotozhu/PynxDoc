# SCALE编解码器

Rust的SCALE（简单级联聚合Little-Endian）数据格式的实现
用于Parity 的Substrate框架中使用的类型。

SCALE是一种轻量级格式，允许进行编码（和解码），从而使其高度适用于资源受限的执行环境，例如区块链运行时和低功耗，低内存设备。

着重要注意的是，需要在编码和解码端都了解编码的上下文信息（有关类型和数据结构的信息）。编码数据中不包括此上下文信息。

为了更好地了解不同类型的编码方式，
看看
[位于Substrate docs站点上的低级数据格式概述页面]（https://substrate.dev/docs/en/overview/low-level-data-format）。

## 实现

编解码器需要实现以下的特性：

###  编码

"编码"特征用于将数据编码为SCALE格式。"编码"特征包含以下功能：
*`size_hint（＆self）-> usize`：获取编码数据所需的容量（以字节为单位）。
这是为了避免编码所需的内存双重分配。
它可以是估计值，而不必是确切数字。
如果大小未知，甚至没有很好的最大值，那么我们可以从trait实现中跳过此函数。
这需要便宜的操作，因此不应涉及迭代等。
*`encode_to <T：Output>（＆self，dest：＆mut T）`：对值进行编码并将其附加到目标缓冲区。
*`encode（＆self）-> Vec <u8>`：对类型数据进行编码并返回一个切片。
*`using_encoded <R，F：FnOnce（＆[u8]）-> R>（＆self，f：F）-> R`：对类型数据进行编码，并对编码后的值执行闭包。
返回已执行闭包的结果。

**注意：**对于值类型，实现应覆盖" using_encoded"，对于分配类型应覆盖" encode_to"。
应该尽可能对所有类型实现`size_hint`。包装器类型应覆盖所有方法。

### 解码

"解码"特征用于将编码数据反序列化/解码为相应的类型。

*`fn encode <I：Input>（value：＆mut I）-> Result <Self，Error>`：尝试将值从SCALE格式解码为调用的类型。
如果解码失败，则返回`Err`。

### CompactAs

" CompactAs"特征用于将自定义类型/结构包装为紧凑类型，这使其具有更高的空间/内存效率。
紧凑的编码在[此处]（https://substrate.dev/docs/en/overview/low-level-data-format#compactgeneral-integers）中进行了描述。

*`encode_as（＆self）->＆Self :: As`：将类型（self）编码为紧凑类型。
类型" As"是在相同特征中定义的，其实现应为可紧凑编码的。
*`decode_from（_：Self :: As）-> Self`：从紧凑的可编码类型中解码类型（self）。

### HasCompact

如果已实现，则具有" HasCompact"特征，则表明相应的类型是紧凑型可编码类型。

### EncodeLike

需要为每种类型手动实现`EncodeLike`特性。使用导出时，它是为您自动完成。基本上，特征使您有机会接受多种类型都编码为相同表示形式的函数。

## 使用示例

以下是一些示例来演示编解码器的用法。

###简单类型

锈

使用parity_scale_codec :: {编码，解码};

＃[派生（Debug，PartialEq，Encode，Decode）]
枚举EnumType {
    ＃[codec（index =" 15"）]
    一种，
    B（u32，u64），
    C {
        答：u32，
        b：u64，
    }，
}

让a = EnumType :: A;
令b = EnumType :: B（1，2）;
令c = EnumType :: C {a：1，b：2};

a.using_encoded（| ref slice | {
    assert_eq！（slice，＆b" \ x0f"）;
}）;

b.using_encoded（| ref slice | {
    assert_eq！（切片，＆b" \ x01 \ x01 \ 0 \ 0 \ 0 \ x02 \ 0 \ 0 \ 0 \ 0 \ 0 \ 0 \ 0"）；
}）;

c.using_encoded（| ref slice | {
    assert_eq！（切片，＆b" \ x02 \ x01 \ 0 \ 0 \ 0 \ x02 \ 0 \ 0 \ 0 \ 0 \ 0 \ 0 \ 0"）；
}）;

让mut da：＆[u8] = b" \ x0f";
assert_eq！（EnumType :: decode（＆mut da）.ok（），Some（a））;

让mut db：＆[u8] = b" \ x01 \ x01 \ 0 \ 0 \ 0 \ x02 \ 0 \ 0 \ 0 \ 0 \ 0 \ 0 \ 0";
assert_eq！（EnumType :: decode（＆mut db）.ok（），Some（b））;

让mut dc：＆[u8] = b" \ x02 \ x01 \ 0 \ 0 \ 0 \ x02 \ 0 \ 0 \ 0 \ 0 \ 0 \ 0 \ 0";
assert_eq！（EnumType :: decode（＆mut dc）.ok（），Some（c））;

让mut dz：＆[u8] =＆[0];
assert_eq！（EnumType :: decode（＆mut dz）.ok（），无）;

```

###具有HasCompact的紧凑型

锈

使用parity_scale_codec :: {编码，解码，压缩，HasCompact}；

＃[派生（Debug，PartialEq，Encode，Decode）]
struct Test1CompactHasCompact <T：HasCompact> {
    ＃[编解码器（紧凑）]
    酒吧：T，
}

＃[派生（Debug，PartialEq，Encode，Decode）]
struct Test1HasCompact <T：HasCompact> {
    ＃[codec（encoded_as =" <T为HasCompact> :: Type"）]
    酒吧：T，
}

让test_val：（u64，usize）=（0u64，1usize）;

让已编码= Test1HasCompact {bar：test_val.0} .encode（）;
assert_eq！（encoded.len（），test_val.1）;
assert_eq！（<Test1CompactHasCompact <u64 >> :: decode（＆mut＆encoded [..]）。unwrap（）。bar，test_val.0）;

```
###使用CompactAs的类型

锈

使用serde_derive :: {序列化，反序列化};
使用parity_scale_codec :: {编码，解码，压缩，HasCompact，CompactAs}；

＃[cfg_attr（feature =" std"，derive（Serialize，Deserialize，Debug））]
＃[derive（PartialEq，Eq，Clone）]
结构StructHasCompact（u32）;

为StructHasCompact启用CompactA {
    输入As = u32;

    fn encode_as（＆self）->＆Self :: As {
        ＆12
    }

    fn encode_from（__ Self :: As）-> Self {
        StructHasCompact（12）
    }
}

从<Compact <StructHasCompact >>表示为StructHasCompact {
    fn from（_：Compact <StructHasCompact>）-> Self {
        StructHasCompact（12）
    }
}

＃[派生（Debug，PartialEq，Encode，Decode）]
枚举TestGenericHasCompact <T> {
    一种 {
        ＃[codec（compact）] a：T
    }，
}

让a = TestGenericHasCompact :: A :: <StructHasCompact> {
    一个：StructHasCompact（12325678），
};

让已编码= a.encode（）;
assert_eq！（encoded.len（），2）;

```

##派生属性

派生实现支持以下属性：
-`codec（dumb_trait_bound）`：该属性需要放置在特征之一的类型上方
  应该实施。它将使确定附加特征范围的算法
  退回去只使用类型的类型参数。这在以下情况下很有用
  该算法在公共接口中包括私有类型。通过使用此属性，您应该
  无法再次收到此错误/警告。
-`codec（skip）`：需要放在字段上方，并在编码/解码时跳过该字段。
-`codec（compact）`：需要放在字段上方，并使字段使用紧凑编码。
  （该类型需要支持紧凑编码。）
-`codec（encoded_as（OtherType））`：需要放置在字段上方并使字段编码
  通过使用" OtherType"。
-`codec（index（" 0"））`：需要放在枚举变量上方，以使该变量使用给定值
  编码时的索引。默认情况下，索引是通过从" 0"开始算起
  第一个变体。