# trait std::borrow::Borrow
[ + ]  显示声明
```rust
pub trait Borrow<Borrowed> 
where
    Borrowed: ?Sized, 
{
    fn borrow(&self) -> &Borrowed;
}
```
[ - ]
*  借用数据的特征。

在Rust中，通常在不同的用例里提供同一类型类型的不同表示形式。例如，根据不同的用途，值的存储位置和管理可以被相应选择为指针类型（Box<T>）或弱类型（Rc<T>）。除了这些可以与任何类型一起使用的通用包装外，某些类型还提供了可选的高成本的部件。一个例子是String增加了将字符串扩展到基本str。这将增加原有的保持简单不变的字符串所不需要的其他信息。

这些类型通过对数据类型的引用来提供对基础数据的访问。被称作为"借来的"那种类型。例如，Box<T>可以借用为T而String 可以借用为str。

（任何）类型通过实现Borrow<T>，提供特性中的borrow方法返回一个T的引用，可以被借用成为T。一个类型可以自由被借用成多种不同的类型。如果希望借用成为可变类型-即允许修改基础数据，则可以额外实现BorrowMut<T>。

此外，在提供其他特征的实现时，需要考虑他们所代表的基础类型的是否与与对应的实际基础类型的行为结果相同。典型的，通过代码会在它依赖一些附加的特性时，使用Borrow<T>。这些特征可能会表现为附加的特性范围。

特别是Eq，Ord并且Hash对于借入和拥有的值必须等效：x.borrow() == y.borrow()应给出与相同的结果x == y。

在少数情况下，如果通用代码需要在T的所有引用类型上都能工作，使用AsRef<T>更好，因为更多类型可以安全地实现它。

## 例子
作为数据集合，HashMap<K, V>拥有键和值。即使键值的实际数据被包装在某种管理类型中，仍然应该可以使用对键值数据的引用来搜索相应的值。例如，如果键是一个字符串，则它很有可能与一个String形成HashMap，而应该可以使用&str进行搜索。因此，get需要能够使用&str，而insert可以工作在String上。

略有简化，相关部分HashMap<K, V>如下所示：
```rust 
use std::borrow::Borrow;
use std::hash::Hash;

pub struct HashMap<K, V> {
    // fields omitted
}

impl<K, V> HashMap<K, V> {
    pub fn insert(&self, key: K, value: V) -> Option<V>
    where K: Hash + Eq
    {
        // ...
    }

    pub fn get<Q>(&self, K: &Q) -> Option<&V>
    where
        K: Borrow<Q>,
        Q: Hash + Eq + ?Sized
    {
        // ...
    }
}
```


整个哈希表中键类型是通用的K。由于这些键值与哈希映射一起存储，此类型必须拥有键的数据。插入键/值对时，会给表键一个K，并且根据这个值找到正确的哈希值存储区，并根据该值检查键是否已经存在。因此这个K需要: Hash + Eq。（可哈希的，可使用=的）

但是，在哈希表中搜索值时，必须提供K的引用作为要搜索的关键字，这将需要必须要创建这样的具有所有权的值。对于字符串键值，这意味着需要要仅支持str的地方创建仅用于搜索的String。

相反，该get方法在基础键值数据类型上是通用的，在上面的函数签名中称为Q。它通过额外地要求K：Borrow<Q>来表明K可以被借用为Q，通过添加额外的Q:Hash + Eq，它指示K和Q需要满足Hash和Eq的特性（和这两个类型产生完全一致的效果）

get的实现尤其依赖于Hash调用结果的一致性，即通过对Q类型调用Hash::hash来确定密钥的哈希存储所在的桶，和通过K的值计算得到的哈希的一致性。（即把K借用成Q时，其哈希计算的一致性）

因此，如果经过K包装后的Q产生的哈希与裸的Q所产生的不同，则哈希表将会出错。例如，假设您有一个字符串包装但比较ASCII字母时忽略大小写的类型：

```rust
pub struct CaseInsensitiveString(String);

impl PartialEq for CaseInsensitiveString {
    fn eq(&self, other: &Self) -> bool {
        self.0.eq_ignore_ascii_case(&other.0)
    }
}

impl Eq for CaseInsensitiveString { }
```

因为两个相等的值需要产生相同的哈希值，所以的实现也Hash需要忽略ASCII大小写：

```rust
impl Hash for CaseInsensitiveString {
    fn hash<H: Hasher>(&self, state: &mut H) {
        for c in self.0.as_bytes() {
            c.to_ascii_lowercase().hash(state)
        }
    }
}
```

可以CaseInsensitiveString实施Borrow<str>吗？它当然可以通过其包含的拥有的字符串为字符串切片提供引用。但是由于其Hash实现方式不同，因此它的行为方式也不同str，因此，实际上，一定不能实现Borrow<str>。如果它希望允许其他人访问基础str，则可以这样做通过AsRef<str>而不会带来任何额外的要求。