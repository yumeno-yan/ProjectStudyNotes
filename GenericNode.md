# GenericNode与Type

## 简介
这一章主要说明sonic-cpp中的GenericNode与Type。Type作为一个比较简单的知识不值得单独开一章，而它又与GenericNode的基础，因此合为一章。

GenericNode代表的是解析后的每个值，包括key和value。这一部分的代码位于[此处](https://github.com/bytedance/sonic-cpp/blob/master/include/sonic/dom/genericnode.h)。

## Type

Type由两个枚举组成。

```C++
enum TypeFlag
{
    // BasicType: 3 bits
    kNull = 0,    // xxxxx000
    kBool = 2,    // xxxxx010
    kNumber = 3,  // xxxxx011
    kString = 4,  // xxxxx100
    kRaw = 5,     // xxxxx101
    // Container Mask 00000110, & Mask == Mask
    kObject = 6,  // xxxxx110
    kArray = 7,   // xxxxx111

    // SubType: 2 bits
    kFalse = ((uint8_t) (0 << 3)) | kBool,   // xxx00_010, 2
    kTrue = ((uint8_t) (1 << 3)) | kBool,    // xxx01_010, 10
    kUint = ((uint8_t) (0 << 3)) | kNumber,  // xxx00_011, 3
    kSint = ((uint8_t) (1 << 3)) | kNumber,  // xxx01_011, 11
    kReal = ((uint8_t) (2 << 3)) | kNumber,  // xxx10_011, 19
    kStringCopy = kString,  // xxx00_100, 4
    kStringFree = ((uint8_t) (1 << 3)) | kString,  // xxx01_100, 12
    kStringConst = ((uint8_t) (2 << 3)) | kString,  // xxx10_100, 20

};  // 8 bits

enum TypeInfo
{
    kTotalTypeBits = 8,

    // BasicType: 3 bits
    kBasicTypeBits = 3,
    kBasicTypeMask = 0x7,

    // SubType: 5 bits (including basic 3 bits)
    kSubTypeBits = 2,
    kSubTypeMask = 0x1F,

    // Others
    kInfoBits = 8,
    kInfoMask = (1 << 8) - 1,
    kOthersBits = 56,
    kLengthMask = (0xFFFFFFFFFFFFFFFF << 8),
    kContainerMask = 0x6,  // 00000110
};
```

TypeFlag表示具体的类型。sonic-cpp用一个8位的变量来表示节点的类型，见下文的Type。其中低3位表示基础类型，也就是Number、Bool之类，低3位再结合前面的两位就可以标识子类型，也就是true、false这些。

TypeInfo用于从变量t中提取类型。例如我想要知道一个节点的子类型，只需要

```C++
static_cast<TypeFlag>(t.t & kSubTypeMask)
```

即可。

#### tips:

sonic-cpp为类型保留了8位，这样做的原因可能有两点。第一，为了以后的扩展性考虑，额外的位数可以给未来提供更多的可能性；第二，一个字节是8位，类型设置为8位有利于内存对齐，提高读取写入效率。

## GenericNode

### 关于union
union的特点是内部所有的成员变量共用同一块内存地址，改变这块内存地址上的所有内容会导致其他变量的变化。

```C++
union
{
    int a;
    char b;
}
a = 1;
b = '1'; // 此时a的值会变成'1'的ASCII码，也就是49。
```

### Type

```C++
struct Type
{
    uint8_t t;
    uint8_t _len[7];
    char _ptr[8];
};  // 16 bytes
```

这个结构体用于表示json值的类型信息。包括一个类型标记t(8位)，一个7字节的长度数组len，以及一个8字节的指针ptr。

### ContainerNext

```C++
union ContainerNext
{
    size_t ofs;
    void *children;
};  // 8 bytes
```
ContainerNext是Object类型和Array类型的基础。
其中ofs用于标识偏移量，在sonic-cpp中用于记录当前节点的父节点的位置。children是一个void*类型的指针，当解析完一个完整的Object或Array后，会申请一片新的空间存放解析完的Object或Array，然后由children指向这片区域。

### Object和Array
```C++
struct Array/Object
{
    uint64_t len;
    union ContainerNext next;
};  // 16 bytes
```
len表示子元素的长度，next是下一个元素或子元素的起始位置。

### String和Raw
```C++
struct String/Raw
{
    uint64_t len;
    const char *p;
};  // 16 bytes
```
len是字符串的长度，String是格式化后的文本，Raw是原始文本。

String赋值的函数是handler.h中的stringImpl(StringView s)。定义如下：

```C++
bool stringImpl(StringView s)
{
    st_[np_ - 1].setLength(s.size(), kStringCopy);
    st_[np_ - 1].sv.p = s.data();
    return true;
}
```



### Number
```C++
struct Number
{
    uint64_t t;
    union
    {
        int64_t i64;
        uint64_t u64;
        double f64;
    };  // lower 8 bytes
};
```
t是类型标记，i64和u64用于存储整数，而f64用于存浮点数。

### Data
```C++
struct Data
{
    uint64_t _[2];
};
```
这个结构体是一个通用的数据容器，包含两个未命名的uint64_t类型成员。它可以用于存储任何类型的数据，具体取决于上下文。

### 匿名union
在实际使用时，这个联合体可以被看作是一个单一的实体，其中的各个成员共享同一块内存。这使得可以在不改变内存占用的情况下，在不同的数据类型之间切换（也正是通过这种方式实现了通用节点）。匿名联合体的使用简化了访问联合体内成员的方式。通过联合体，解析器能够在处理不同类型的JSON数据时保持良好的性能和较低的内存消耗。

## 成员函数解析

这里列举几个重要的函数。

### setLength与setType

明白了setLength的原理，那setType也就不必多说了。

setLength一般在创建字符串、Object、Array时使用，函数原型：

```C++
void setLength(size_t len, TypeFlag flag) noexcept
{
    // 以kStringCopy为例
    // kInfoBits为8，kStringCopy为二进制下的100
    sv.len = (len << kInfoBits) | static_cast<uint64_t>(flag);
}
```

在`setLength`中，先将len左移8位，再与flag进行或操作。

不直接用sv.len = len，而是要进行移位和位运算的操作，这样做的好处是可以将类型信息包含在长度信息中。以len为6举例说明：

```shell
// 以下所有数字都是二进制形式
len = 110
sv.len = (110 << 8) | 100 = 11000000100
// 拆解后
110 | 0000 | 0100
```

这样sv.len后8位就是TypeFlag也就是类型信息，前面的数字就是字符串的长度信息。前文提到在union中所有成员共用同一片内存，注意到Type的第一个成员是一个8位的变量t，此时Type的t也就会变成sv.len的后8位，巧妙地保存了类型的信息。

setType也是类似的原理。

```C++
void setType(TypeFlag flag) noexcept
{
    sv.len = static_cast<uint64_t>(flag);
    if (IsString())
    {
        setEmptyString();
    }
}
```

区别在于setType只记录了类型信息，不记录长度信息。

