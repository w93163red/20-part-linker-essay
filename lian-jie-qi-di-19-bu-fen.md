# 链接器第19部分

## \_\_start 和 \_\_stop 符号

关于另一个GNU链接器扩展的快速说明。如果链接器在输出文件中看到一个可以成为C变量名一部分的节——名称只包含字母数字字符或下划线——链接器将自动定义标记该节开始和结束的符号。注意，这对大多数节名不适用，因为按惯例大多数节名以句点开头。但节的名称可以是任何字符串；它不必以句点开头。当这种情况发生时，对于节NAME，GNU链接器将定义符号 \_\_start\_NAME 和 \_\_stop\_NAME 分别指向节的开始和结束地址。

这对于在几个不同的目标文件中收集一些信息，然后在代码中引用它很方便。例如，GNU C库使用这个来保存可能被调用以释放内存的函数列表。\_\_start 和 \_\_stop 符号用于遍历列表。

在C代码中，这些符号应该被声明为类似 `extern char __start_NAME[]` 的形式。对于外部数组，符号的值和变量的值是相同的。

## 字节交换

我正在开发的新链接器gold是用C++编写的。其中一个吸引人的地方是使用模板特化来进行高效的字节交换。任何可以用于交叉编译器的链接器都需要能够在写出时交换字节，以便在小端系统上运行时生成大端系统的代码，反之亦然。GNU链接器总是一次一个字节地将数据存储到内存中，这对于本机链接器来说是不必要的。几年前的测量显示，这占用了链接器约5%的CPU时间。由于本机链接器是最常见的情况，值得避免这种惩罚。

在C++中，这可以使用模板和模板特化来完成。思路是为写出数据编写一个模板。然后提供两个模板的特化，一个用于相同字节序的链接器，一个用于相反字节序的链接器。然后在编译时选择要使用的那个。代码看起来像这样；为简单起见，我只展示16位的情况。

```cpp
// Endian 只是表示主机是否为大端序。
struct Endian
{
public:
  // 用于模板特化。
  static const bool host_big_endian = __BYTE_ORDER == __BIG_ENDIAN;
};

// Valtype_base 是一个基于大小（8、16、32、64）的模板，
// 它将类型 Valtype 定义为指定大小的无符号整数。
template<int size>
struct Valtype_base;

template<>
struct Valtype_base<16>
{
  typedef uint16_t Valtype;
};

// Convert_endian 是一个基于大小和主机与目标是否具有相同字节序的模板。
// 它定义类型 Valtype，就像 Valtype_base 一样，还定义了一个 convert_host 函数，
// 该函数接受 Valtype 类型的参数并返回相同的值，但如果主机和目标具有不同的字节序，则进行交换。
template<int size, bool same_endian>
struct Convert_endian;

template<int size>
struct Convert_endian<size, true>
{
  typedef typename Valtype_base<size>::Valtype Valtype;
  static inline Valtype
  convert_host(Valtype v)
  { return v; }
};

template<>
struct Convert_endian<16, false>
{
  typedef Valtype_base<16>::Valtype Valtype;
  static inline Valtype
  convert_host(Valtype v)
  { return bswap_16(v); }
};

// Convert 是一个基于大小和目标是否为大端序的模板。
// 它定义 Valtype 和 convert_host，就像 Convert_endian 一样。
// 它与 Convert_endian 的唯一区别在于第二个模板参数的含义。
template<int size, bool big_endian>
struct Convert
{
  typedef typename Valtype_base<size>::Valtype Valtype;
  static inline Valtype
  convert_host(Valtype v)
  {
    return Convert_endian<size, big_endian == Endian::host_big_endian>
      ::convert_host(v);
  }
};

// Swap 是一个基于大小和目标是否为大端序的模板。
// 它定义类型 Valtype 和函数 readval 和 writeval。
// 这些函数从缓冲区读取和写入适当大小的值，必要时进行交换。
template<int size, bool big_endian>
struct Swap
{
  typedef typename Valtype_base<size>::Valtype Valtype;
  static inline Valtype
  readval(const Valtype* wv)
  { return Convert<size, big_endian>::convert_host(*wv); }
  static inline void
  writeval(Valtype* wv, Valtype v)
  { *wv = Convert<size, big_endian>::convert_host(v); }
};
```

现在，例如，链接器使用 `Swap<16,true>::readval` 读取16位大端值。这之所以有效，是因为链接器总是知道要交换多少数据，而且它总是知道它是在读取大端还是小端数据。
