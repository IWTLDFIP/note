## 需要了解的词

- `string interning`  
    一种在内存中仅存储每个唯一字符串的一个副本的技术
    
- `unsafe.Pointer`  
    `Golang`为“大胆的”程序员提供的更直接操作内存的方式，`unsafe.Pointer`是一种特殊意义的指针（通用指针），它可以包含任意类型的地址，有点类似于C语言里的void*指针，全能型的；在`Golang`中是用于各种指针相互转换的桥梁
    

## [](https://km.woa.com/group/571/articles/show/528065#%E9%80%89%E6%8B%A9%E5%90%88%E9%80%82%E7%9A%84%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%8B%BC%E6%8E%A5%E6%96%B9%E5%BC%8F)选择合适的字符串拼接方式

首先来看看老生常谈的字符串拼接在 Golang 中的表现，这里我们主要看以下几个字符串拼接方法：`fmt.Sprintf`, `+`, `strings.Join`, `bytes.Buffer`, `strings.Builder`

实现一下单元测试：

```go
package test

import (
   "bytes"
   "fmt"
   "strings"
   "testing"
)

// fmt.Printf
func BenchmarkFmtSprintfMore(b *testing.B) {
   var s string
   for i := 0; i < b.N; i++ {
      s += fmt.Sprintf("%s%s", "hello", "world")
   }
   fmt.Errorf(s)
}

// 加号 拼接
func BenchmarkAddMore(b *testing.B) {
   var s string
   for i := 0; i < b.N; i++ {
      s += "hello" + "world"
   }
   fmt.Errorf(s)
}

// strings.Join
func BenchmarkStringsJoinMore(b *testing.B) {
   var s string
   for i := 0; i < b.N; i++ {
      s += strings.Join([]string{"hello", "world"}, "")
   }
   fmt.Errorf(s)
}

// bytes.Buffer
func BenchmarkBufferMore(b *testing.B) {
   buffer := bytes.Buffer{}
   for i := 0; i < b.N; i++ {
      buffer.WriteString("hello")
      buffer.WriteString("world")
   }
   fmt.Errorf(buffer.String())
}

// strings.Builder
func BenchmarkStringBuilderMore(b *testing.B) {
   builder := strings.Builder{}
   for i := 0; i < b.N; i++ {
      builder.WriteString("hello")
      builder.WriteString("world")
   }
   fmt.Errorf(builder.String())
}
```

运行结果：

```javascript
$ go test -bench="Concat$" -benchmem .
goos: darwin
goarch: amd64
pkg: example
BenchmarkPlusConcat-8         19      56 ms/op   530 MB/op   10026 allocs/op
BenchmarkSprintfConcat-8      10     112 ms/op   835 MB/op   37435 allocs/op
BenchmarkBuilderConcat-8    8901    0.13 ms/op   0.5 MB/op      23 allocs/op
BenchmarkBufferConcat-8     8130    0.14 ms/op   0.4 MB/op      13 allocs/op
BenchmarkByteConcat-8       8984    0.12 ms/op   0.6 MB/op      24 allocs/op
BenchmarkPreByteConcat-8   17379    0.07 ms/op   0.2 MB/op       2 allocs/op
PASS
ok      example 8.627s
```

从基准测试的结果来看，使用 `+` 和 `fmt.Sprintf` 的效率是最低的，和其余的方式相比，性能相差约 1000 倍，而且消耗了超过 1000 倍的内存。当然 `fmt.Sprintf` 通常是用来格式化字符串的，一般不会用来拼接字符串。

`strings.Builder`、`bytes.Buffer` 和 `[]byte` 的性能差距不大，而且消耗的内存也十分接近，性能最好且消耗内存最小的是 `preByteConcat`，这种方式**预分配了内存**，在字符串拼接的过程中，不需要进行字符串的拷贝，也不需要分配新的内存，因此性能最好，且内存消耗最小

可以看到 strings.Builder 的性能是最好的，当然在少量的字符串拼接上，还是可以直接使用`+`的

`fmt.Sprintf`, `+`, `strings.Join`这几个的性能都差不多，它们性能不佳的原因主要是每次+都要两个string复制到新分配的string中，拼接后的字符串越长，需要重新分配的内存越大，同样需要销毁的空间也越来越大

`bytes.Buffer`和 `strings.Builder`的原理类似，这是 Go 官方对 `strings.Builder` 的解释：

> A Builder is used to efficiently build a string using Write methods. It minimizes memory copying.

同时`string.Builder` 也提供了预分配内存的方式 `Grow`：

```go
func builderConcat(n int, str string) string {
   var builder strings.Builder
   builder.Grow(n * len(str))
   for i := 0; i < n; i++ {
      builder.WriteString(str)
   }
   return builder.String()
}
```

使用了 Grow 优化后的版本的 benchmark 结果如下：

```javascript
BenchmarkBuilderConcat-8   16855    0.07 ns/op   0.1 MB/op       1 allocs/op
BenchmarkPreByteConcat-8   17379    0.07 ms/op   0.2 MB/op       2 allocs/op
```

与预分配内存的 `[]byte` 相比，因为省去了 `[]byte` 和字符串(string) 之间的转换，内存分配次数还减少了 1 次，内存消耗减半

### [](https://km.woa.com/group/571/articles/show/528065#%E6%AF%94%E8%BE%83-stringsbuilder-%E5%92%8C-codefcf73e375744495bf6750ee1776154e9)比较 strings.Builder 和 `+`

`strings.Builder` 和 `+` 性能和内存消耗差距如此巨大，是因为两者的内存分配方式不一样。

字符串在 Go 语言中是不可变类型，占用内存大小是固定的，当使用 `+` 拼接 2 个字符串时，生成一个新的字符串，那么就需要开辟一段新的空间，新空间的大小是原来两个字符串的大小之和。拼接第三个字符串时，再开辟一段新空间，新空间大小是三个字符串大小之和，以此类推。假设一个字符串大小为 10 byte，拼接 1w 次，需要申请的内存大小为：

```javascript
10 + 2 * 10 + 3 * 10 + ... + 10000 * 10 byte = 500 MB 
```

而 `strings.Builder`，`bytes.Buffer`，包括切片 `[]byte` 的内存是以倍数申请的。例如，初始大小为 0，当第一次写入大小为 10 byte 的字符串时，则会申请大小为 16 byte 的内存（恰好大于 10 byte 的 2 的指数），第二次写入 10 byte 时，内存不够，则申请 32 byte 的内存，第三次写入内存足够，则不申请新的，以此类推。在实际过程中，超过一定大小，比如 2048 byte 后，申请策略上会有些许调整。我们可以通过打印 `builder.Cap()` 查看字符串拼接过程中，`strings.Builder` 的内存申请过程。

```go
func TestBuilderConcat(t *testing.T) {
   var str = "1"
   var builder strings.Builder
   cap := 0
   for i := 0; i < 10000; i++ {
      if builder.Cap() != cap {
         fmt.Print(builder.Cap(), " ")
         cap = builder.Cap()
      }
      builder.WriteString(str)
   }
}

func TestBufferConcat(t *testing.T) {

   var str = "1"
   buffer := bytes.Buffer{}
   cap := 0
   for i := 0; i < 10000; i++ {
      if buffer.Cap() != cap {
         fmt.Print(buffer.Cap(), " ")
         cap = buffer.Cap()
      }
      buffer.WriteString(str)
   }
}
```

运行结果如下：

```shell
➜  test go test -run="TestBufferConcat" . -v
=== RUN   TestBufferConcat
64 129 259 519 1039 2079 4159 8319 16639 --- PASS: TestBufferConcat (0.00s)
PASS
ok      StudyProject/src/second/test    0.321s
➜  test go test -run="TestBuilderConcat" . -v
=== RUN   TestBuilderConcat
8 16 32 64 128 256 512 896 1408 2048 3072 4096 5376 6912 9472 12288 --- PASS: TestBuilderConcat (0.00s)
PASS
ok      StudyProject/src/second/test    0.235s
```

`bytes.Buffer`无脑2n+1，`strings.Builder`前期为2n，到了512之后停止翻倍扩充

### [](https://km.woa.com/group/571/articles/show/528065#%E6%AF%94%E8%BE%83-stringsbuilder-%E5%92%8C-bytesbuffer)比较 strings.Builder 和 bytes.Buffer

`strings.Builder` 和 `bytes.Buffer` 底层都是 `[]byte` 数组，但 `strings.Builder` 性能比 `bytes.Buffer` 略快约 10% 。一个比较重要的区别在于，`bytes.Buffer` 转化为字符串时重新申请了一块空间，存放生成的字符串变量，而 `strings.Builder` 直接将底层的 `[]byte` 转换成了字符串类型返回了回来

- bytes.Buffer
    

```go
// To build strings more efficiently, see the strings.Builder type.
func (b *Buffer) String() string {
   if b == nil {
      // Special case, useful in debugging.
      return "<nil>"
   }
   return string(b.buf[b.off:])
}
```

- strings.Builder
    

```go
// String returns the accumulated string.
func (b *Builder) String() string {
   return *(*string)(unsafe.Pointer(&amp;b.buf))
}
```

`bytes.Buffer` 的注释中还特意提到了：

> To build strings more efficiently, see the strings.Builder type.

### [](https://km.woa.com/group/571/articles/show/528065#%E9%98%B6%E6%AE%B5%E6%80%BB%E7%BB%93)阶段总结

1. 字符串最高效的拼接方式是结合预分配内存方式 `Grow` 使用 `string.Builder`
    
2. 当使用 `+` 拼接字符串时，生成新字符串，需要开辟新的空间
    
3. 当使用 `strings.Builder`，`bytes.Buffer` 或 `[]byte` 的内存是按倍数申请的，在原基础上不断增加
    
4. `strings.Builder` 比 `bytes.Buffer` 性能更快，一个重要区别在于 `bytes.Buffer` 转化为字符串重新申请了一块空间存放生成的字符串变量；而 `strings.Builder` 直接将底层的 `[]byte` 转换成字符串类型返回
    

## [](https://km.woa.com/group/571/articles/show/528065#%E9%81%BF%E5%85%8D%E9%87%8D%E5%A4%8D%E7%9A%84%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%88%B0%E5%AD%97%E8%8A%82%E5%88%87%E7%89%87%E7%9A%84%E8%BD%AC%E6%8D%A2)避免重复的字符串到字节切片的转换

在`golang`中，字符串的底层是字节`slice`，灵活地运用二者之间的类型转换可以避免很多的内存分配

`byte`切片转换成`string`的场景很多，有时候只是应用于在临时需要字符串的场景下，`byte`切片转换成`string`时并不会拷贝内存，而是直接返回一个`string`，这个**string的指针(string.str)指向切片的内存地址**

比如，编译器会识别如下临时场景：

- 使用`m[string(b)]`来查找`map`（`map`中的`key`类型是`string`时，临时把切片`b`转成`string`）  
      
    编译器为这种情况实现特定的优化：
    

```go
var m map[string]string
v, ok := m[string(bytes)]
```

如上面这样写，编译器会避免将字节切片转换为字符串到 map 中查找，这是非常特定的细节，如果你像下面这样写，这个优化就会失效：

```go
key := string(bytes)
val, ok := m[key]
```

- 字符串拼接，如<" + "string(b)" + ">  
    
- 字符串比较： string(b) == "foo"
    

由于只是临时把byte切片转换成string，也就避免了因byte切片内容修改而导致string数据变化的问题，所以此时可以不必拷贝内存

但反过来并不是这样的，string转成byte切片需要一次内存拷贝的动作，其过程如下：

- 申请切片内存空间
    
- 将`string`拷贝到切片中
    

所以不要反复从固定字符串创建字节`slice`，因为重复的切片初始化会带来性能损耗；尽可能的执行一次转换并捕获结果，以下是相关的`Benchmark`

```go
// BenchmarkBad 
//  @param b *testing.B 
//  @author: Kevineluo 2022-11-06 08:43:48 
// BenchmarkBad-16      540784207            2.166 ns/op        0 B/op         0 allocs/op
func BenchmarkBad(b *testing.B) {
   for i := 0; i < b.N; i++ {
      doNothing([]byte("Hello world"))
   }
}

// BenchmarkGood 
//  @param b *testing.B 
//  @author: Kevineluo 2022-11-06 08:43:37 
// BenchmarkGood-16     778790289            1.591 ns/op        0 B/op         0 allocs/op
func BenchmarkGood(b *testing.B) {
   bytes := []byte("Hello world")
   b.ResetTimer()
   for i := 0; i < b.N; i++ {
      doNothing(bytes)
   }
}

func doNothing(input []byte) {
}
```

### [](https://km.woa.com/group/571/articles/show/528065#%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%92%8C%E5%AD%97%E8%8A%82%E5%88%87%E7%89%87%E7%9A%84%E6%9B%B4%E9%AB%98%E6%95%88%E4%BA%92%E8%BD%AC)字符串和字节切片的更高效互转

在面对原地转换的场景时，利用 `string` 和 `byte slice` 底层数据结构实现的技巧（需要比较`aggressive`的使用到`unsafe.Pointer`），如果需要大量互转可以考虑使用这种方式：

```go
/*
type StringHeader struct { // reflect.StringHeader
    Data uintptr
    Len  int
}
type SliceHeader struct { // reflect.SliceHeader
    Data uintptr
    Len  int
    Cap  int
}
*/

// NOTE：注意之后不要修改 string, 它们共享了底层的Data
func str2bytes(s string) []byte {
   x := (*[2]uintptr)(unsafe.Pointer(&amp;s))
   b := [3]uintptr{x[0], x[1], x[1]}
   return *(*[]byte)(unsafe.Pointer(&amp;b))
}

func bytes2str(b []byte) string {
   return *(*string)(unsafe.Pointer(&amp;b))
}
```

`str2bytes`中，直接复用`string`中的`Data`和`Len`，再将`Cap`设为与`Len`相同，即可直接强转成`*[]byte`

反之亦然，在`bytes2str`中，直接复用`byte slice`中的 `Data`和`Len`，即可强转为`*string`

## [](https://km.woa.com/group/571/articles/show/528065#%E5%9C%A8%E5%90%88%E9%80%82%E7%9A%84%E6%97%B6%E6%9C%BA%E4%BD%BF%E7%94%A8%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%B1%A0string-interning)在合适的时机使用字符串池(string interning)

`string interning`是一种在内存中仅存储每个唯一字符串的一个副本的技术，基于它我们可以显著地优化使用大量重复字符串应用的内存使用量

而在`golang`中，`string`类型的[数据结构](https://go.dev/src/runtime/string.go)为：

```go
type stringStruct struct {
  str unsafe.Pointer
  len int
}
```

结构很简单：

- str: 字符串的首地址
    
- len: 字符串的长度
    

字符串生成时，会先构建stringStruct对象，再转成string；转换的源码如下：

```go
// go:nosplit
func gostringnocopy(str *byte) string {
  // 先构造stringStruct
  ss := stringStruct{str: unsafe.Pointer(str), len: findnull(str)}
  // 再将stringStruct转成string
  s := *(*string)(unsafe.Pointer(&amp;ss))
  return s
}
```

并且在`golang`中，字符串是不可变的（不管是否显式使用`const`声明），我们可以通过以下试验看到：多个字符串可以共享底层数据

```go
package main

import (
    "fmt"
    "reflect"
    "unsafe"
)

// stringptr 返回指向字符串底层数据的指针
func stringptr(s string) uintptr {
    return (*reflect.StringHeader)(unsafe.Pointer(&amp;s)).Data
}

func main() {
    s1 := "1234"
    s2 := s1[:2] // "12"
    // s1 和 s2 指向了同一个底层Data地址
    fmt.Println(stringptr(s1) == stringptr(s2)) // true
}
```

大多数语言（包括`golang`）都支持对编译期字符串常量的`string interning`：

```go
s1 := "12"
s2 := "1"+"2"
fmt.Println(stringptr(s1) == stringptr(s2)) // true
```

但对于运行时的字符串变量并不支持（或者说没有进行`string interning`）：

```go
s1 := "12"
s2 := strconv.Itoa(12)
fmt.Println(stringptr(s1) == stringptr(s2)) // false
```

### [](https://km.woa.com/group/571/articles/show/528065#%E5%AE%9E%E7%8E%B0-string-interning)实现 string interning

要实现`string interning`很简单，我们只需要设计一个字符串池的数据结构，实现其`Get`和`Set`方法即可

简单的实现`string interning(thread unsafe)`:

```go
type stringInterner map[string]string

func (si stringInterner) InternBytes(b []byte) string {
   if interned, ok := si[string(b)]; ok {
      return interned
   }
   s := string(b)
   si[s] = s
   return s
}
```

以上的 `stringInterner` 在将 `[]byte` 转换为 `string` 的同时，检查此 `string` 是否已经被 `interned`，如果是则直接返回老的 `string`，否则 `intern` 新的 `string`

### [](https://km.woa.com/group/571/articles/show/528065#%E5%87%8F%E5%B0%91%E9%87%8D%E5%A4%8D%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D)减少重复内存分配

由于 `string(b)` 的原理是直接将 b 作为 `string` 的 `Data`，如果程序不断读入新的字节流，会不断生成新的 `string` 对象，`string` 占用的空间呈线性增长。上面这段函数，只使用 `[]byte` 作为比较的依据，新生成的 `string(b)` （分配在栈上）会随着函数结束而被回收，只有未命中的 `string` 会被缓存在堆上

### [](https://km.woa.com/group/571/articles/show/528065#%E5%87%8F%E5%B0%91%E6%AF%94%E8%BE%83%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%BC%80%E9%94%80)减少比较字符串开销

除了降低内存的重复分配外，`string interning`还可以在稳定的时间内比较被占用的字符串是否相等；编译器只需要检查两个指针是否相等(转到源代码)，而不是检查字符：

```javascript
TEXT cmpbody<>(SB),NOSPLIT,$0-0
    CMPQ    SI, DI
    JEQ allsame
```

以下是使用和不使用`string interning`情况下的字符串比较`Benchmark`:

```go
func benchmarkStringCompare(b *testing.B, count int) {
    s1 := strings.Repeat("a", count)
    s2 := strings.Repeat("a", count)
    b.ResetTimer()
    for n := 0; n < b.N; n++ {
        if s1 != s2 {
            b.Fatal()
        }
    }
}

func benchmarkStringCompareIntern(b *testing.B, count int) {
    si := stringInterner{}
    s1 := si.Intern(strings.Repeat("a", count))
    s2 := si.Intern(strings.Repeat("a", count))
    b.ResetTimer()
    for n := 0; n < b.N; n++ {
        if s1 != s2 {
            b.Fatal()
        }
    }
}

func BenchmarkStringCompare1(b *testing.B)   { benchmarkStringCompare(b, 1) }
func BenchmarkStringCompare10(b *testing.B)  { benchmarkStringCompare(b, 10) }
func BenchmarkStringCompare100(b *testing.B) { benchmarkStringCompare(b, 100) }

func BenchmarkStringCompareIntern1(b *testing.B)   { benchmarkStringCompareIntern(b, 1) }
func BenchmarkStringCompareIntern10(b *testing.B)  { benchmarkStringCompareIntern(b, 10) }
func BenchmarkStringCompareIntern100(b *testing.B) { benchmarkStringCompareIntern(b, 100) }
```

使用`string interning`字符串的比较速度保持恒定，不受字符数量的影响：

```go
BenchmarkStringCompare1-4               500000000            2.93 ns/op
BenchmarkStringCompare10-4              300000000            6.21 ns/op
BenchmarkStringCompare100-4             100000000            13.2 ns/op
BenchmarkStringCompareIntern1-4         1000000000           2.60 ns/op
BenchmarkStringCompareIntern10-4        1000000000           2.60 ns/op
BenchmarkStringCompareIntern100-4       1000000000           2.60 ns/op
```

### [](https://km.woa.com/group/571/articles/show/528065#%E7%BA%BF%E7%A8%8B%E5%AE%89%E5%85%A8%E5%92%8C%E6%B7%98%E6%B1%B0%E7%AD%96%E7%95%A5)线程安全和淘汰策略

上面的简单实现是基于`golang`原生`map`的，很明显是非线程安全的；如果真的有`multi goroutine`使用`string intern`的场景的话，可以参考使用[github.com/josharian/intern](https://github.com/josharian/intern)

它的工作原理是暴力的直接使用`sync.Pool`，因为sync.Pool访问是并发安全的，并且它在很大程度上解决了淘汰策略的问题，因为`sync.Pool`中的内容通常最终会被GC(有关淘汰策略，可以参考[Go issue 29696](https://golang.org/issue/29696))

### [](https://km.woa.com/group/571/articles/show/528065#%E9%98%B6%E6%AE%B5%E6%80%BB%E7%BB%93-3)阶段总结

`string interning`可以以CPU时间为代价来节省内存——字符串池中的每次查找都需要对输入字符串进行哈希散列(这可能不适用于CPU资源吃紧的应用)

在合适的场景使用`string interning`能显著的减少内存资源使用，以下资源监控图来自真实的`string interning`优化案例：

![](https://km.woa.com/asset/a3da4a35e40a43b6ab4f45d2bf5735fc?height=394&width=1208)

## [](https://km.woa.com/group/571/articles/show/528065#%E6%80%BB%E7%BB%93)总结

总结下来，对`Golang`的字符串我们需要注意以下几点：

- 牢记`string`和`[]byte`的底层数据结构
    
- 在字符串拼接时注意使用效率（根本的性能差异也是由拼接时的内存分配方式导致的）
    
- 在合适的场景利用好字符串池，可以收获惊人的内存优化