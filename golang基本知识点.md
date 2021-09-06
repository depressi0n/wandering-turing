[toc]

## Golang 基本知识

#### 1. 字符边界

golang默认使用的字符集是UTF-8，是一种变长编码【与之对标的是定长编码，边界清晰但浪费内存】，其编码方式是

|    区间    |               编码方式                |
| :--------: | :-----------------------------------: |
|   0-127    |               0xxxxxxx                |
|  128-2047  |           110xxxxx 10xxxxxx           |
| 2048-65536 | 1110xxxx 10xxxxxx 10xxxxxx 10xxxxxxxx |

#### 2. string、slice、map的结构

```go
type stringStruct struct {
	str unsafe.Pointer // 指向底层数组
	len int
}

type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}

type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // 已有记录数目# live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8 
	B         uint8  // 一共2^B个桶，log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // 使用的溢出桶的数目，approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // 桶的位置，array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // 旧桶的位置，previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // 下一个迁移的旧桶的位置，progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}
type mapextra struct {
	// If both key and elem do not contain pointers and are inline, then we mark bucket
	// type as containing no pointers. This avoids scanning such maps.
	// However, bmap.overflow is a pointer. In order to keep overflow buckets
	// alive, we store pointers to all overflow buckets in hmap.extra.overflow and hmap.extra.oldoverflow.
	// overflow and oldoverflow are only used if key and elem do not contain pointers.
	// overflow contains overflow buckets for hmap.buckets.
	// oldoverflow contains overflow buckets for hmap.oldbuckets.
	// The indirection allows to store a pointer to the slice in hiter.
	overflow    *[]*bmap // 已经使用的溢出桶
	oldoverflow *[]*bmap // 旧桶中用到的溢出桶

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap // 下一个可使用的溢出桶
}

// A bucket for a Go map.
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
	tophash [bucketCnt]uint8
	// Followed by bucketCnt keys and then bucketCnt elems.
	// NOTE: packing all the keys together and then all the elems together makes the
	// code a bit more complicated than alternating key/elem/key/elem/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
}

```

#### 3. slice append扩容

（1）首先预估扩容后的容量如果oldCap\*2<cap则newCap=cap；否则如果oldLen<1024，newCap=2\*oldCap，或newCap=1.25\*oldCap

（2）容量\*元素类型大小为实际申请的申请的大小，向语言本身实现的内存管理模块（提前向OS申请了固定尺寸的内存）申请

#### 4. map 扩容规则

（1）扩容情况有两种，一种是当前记录数目`count/2^B>6.5` -> 翻倍扩容，另一种是溢出桶使用过多（数目超过`2^min{B,15}`）->等量扩容

（2）渐进式扩容，为了避免一次性扩容带来的性能抖动

（3）bmap是实际使用的桶，前64个字节是topHash，接下来keys，再接下来是values

#### 5. 内存对齐

**关键句：数据起始存储地址和占用字节数是对齐边界的整数倍**

（1）什么是对齐边界

与平台和类型大小有关，指针宽度和寄存器宽度（一定意义上与机器字长相同，最大对齐边界），数据类型的对齐边界取平台最大对齐边界与类型大小中的较小值

（2）结构体为什么也要对齐？怎么对齐？

```go
// 在64位平台下的对齐边界
type T struct{
	a int8 // 1 byte, 0%1=0
	b int64 // 8 byte, 8%8=0
	c int32 // 4 byte, 16%4=0
	d int16 // 2 byte, 20%2=0
} // 8 byte 24/8 == 0
```

结构体的对齐边界取其成员的最大对齐边界

| 0    | 1    | 2    | 3    | 4    | 5    | 6    | 7    | 8    | 9    | 10   | 11   | 12   | 13   | 14   | 15   |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| a    | ^    | ^    | ^    | ^    | ^    | ^    | ^    | b    | b    | b    | b    | b    | b    | b    | b    |
| 16   | 17   | 18   | 19   | 20   | 21   | 22   | 23   | 24   | 25   | 26   | 27   | 28   | 29   | 30   | 31   |
| c    | c    | c    | c    | d    | d    | ^    | ^    | ^    | ^    | ^    | ^    | ^    | ^    | ^    | ^    |

如果结构体不对齐，当出现结构体数组时，只可保证结构体内的相对地址能对齐。
在结构体对齐的前提下，结构体数组中各成员的对齐规则均可满足。

#### 6. 函数栈帧（函数调用栈）

（1）| caller's bp | local values | return values | parameters | return address 【返回地址不算在栈帧内】

call指令入栈的返回地址之后，调用者栈基，局部变量区间，调用其他函数时传递返回值和参数的区间作为函数栈帧

一次性分配函数栈帧，在有多个call指令时分配最大的空间【避免越界，大小可以在编译时确定，会插入检测代码，如果需要更多空间，则重新分配】，被调用函数寻找参数和返回值时通过栈指针（sp）和偏移来实现。 

（2）call 指令的作用

一是将下一条指令地址入栈（即上面的返回地址），二是跳转到被调用函数处执行

（3）**返回值赋值发生在defer函数执行之前**

（4）ret指令的作用

一是弹出返回地址，而是跳转到返回地址

（5）参数入栈的顺序是从右往左

（6）要区分匿名返回值和命名返回值和defer语句的情况，使用函数调用栈来分析

```go
func incr(a int)int{
	var b int
  defer func(){
		a++
  	b++
  }()
  a++
  b=a
  return b
}
func main(){
  var a,b int
  b=incr(a)
  fmt.Println(a,b) // 0,1
}
```

```go
func incr(a int)(b int){
  defer func(){
		a++
  	b++
  }()
  a++
  return a
}
func main(){
  var a,b int
  b=incr(a)
  fmt.Println(a,b) // 0,2
}
```



 #### 7. 闭包

（1）什么是闭包？

闭包跟函数最大的不同在于，当捕捉闭包的时候，它的自由变量会在捕捉时被确定，这样即便脱离了捕捉时的上下文，它也能照常运行。

（2）golang对闭包处理

函数是通过funcValue（本质上是一个指针，指向一个runtime.funcval结构体，结构体中只有一个指向函数入口地址的地址，结构体分配在堆上）来处理的，而闭包是funcValue+捕获列表。

```go
type funcval struct {
	fn uintptr
	// variable-size, fn-specific data here
}
```

go语言通过一个function value调用函数时，会把对应的function value的地址存入特定寄存器，通过寄存器+偏移确定捕获变量。

**闭包的要求：被捕获的变量要在外层函数与闭包函数中保持一致性**

如果被捕获变量只有初始化赋值从未被修改，则直接拷贝值到捕获列表中

如果被捕获的局部变量进行了修改，则将该局部变量改为分配到堆上（属于变量逃逸的一种场景），局部变量中存放的是变量的地址，捕获列表中存放的也是捕获变量的地址

如果修改并被捕获的是参数，涉及到函数原型，与局部变量处理方式不一样，参数通过调用者栈帧传递，参数会拷贝一份到堆上，外层函数和闭包使用堆上的数据

如果捕获的是返回值，则将返回值也拷贝一份到堆上，外层函数和闭包使用堆上的数据，在外层函数返回前将堆上的数据拷贝到栈上

**函数本身作为变量、参数和返回值时，都是以function value的形式存在的**

#### 8. 方法表达式和方法变量

go中函数类型只与参数和返回值相关

方法接收者会作为方的第一个参数传入，本质上与函数一致。

不涉及接口的前提下，方法调用时会在**编译期间**进行指针和类型的转换（即指针接收者可以调用类型接收者的方法，类型接收者也可以调用指针接收者的方法）。

**编译期间**完成转换也意味着字面量【编译期间无法拿到地址】无法使用这种语法糖，无法通过编译。

```go
type A struct{
  name string
}
func(a A) GetName() string{
  return a.name
}
func main(){
  a:=A{name:"eggo"}
  f1:=A.GetName // 方法表达式，本质上是Function Value
  f1(a)
  f2:=a.GetName // 方法变量，本质上是Function Value，会捕获方法接收者，形成闭包，在这种情况下会被编译器转化成类型A的方法调用并传入a作为参数
  f2()
}
```

```go
type A struct{
  name string
}
func(a A) GetName() string{
  return a.name
}
func GetFunc() func() string{
  a:= A{name:"eggo in GetFunc"}
  return a.GetName // 相当于一个捕获了局部变量的闭包
}
func main(){
  a:=A{name:"eggo"}
  f2:=a.GetName // 方法变量，会捕获方法接收者，形成闭包，在这种情况下会被编译器转化成类型A的方法调用并传入a作为参数
  fmt.Println(f2())
  f3:=GetFunc() // 形成了闭包，捕获了局部变量a
  fmt.Println(f3())
}
```

#### 9. defer

defer会注册到一个链表，每个goroutine在运行时会有一个对应的g，g中有一个字段指向defer的链表头。

【go1.12】注册时参数和返回值会拷贝到defer结构体（堆上），在执行defer函数时参数和返回值又被拷贝到栈上。在有注册defer函数的函数返回前，执行deferreturn时，判断defer链表头上的defer是不是自己注册的（通过栈指针sp判断），执行，从再链表去除。

【go1.13】在编译阶段增加局部变量，将defer信息保存在当前函数栈帧的局部变量区域， 再通过defferprocstack注册到链表中，在某些场景下（循环中的defer）会沿用go1.12的方式注册到栈上。执行时拷贝从栈上局部变量空间拷贝到参数空间。

【go1.14，open coded defer】在编译阶段插入代码，将defer函数的执行逻辑展开在所属函数中，defer的参数作为局部变量，在返回前调用defer的函数，免于创建defer结构体，并不需要注册到defer列表。代价是当出现panic或者runtime.Goexit时defer函数将无法执行，需要通过栈扫描的方式来发现。
**加快了defer但panic变得更慢了。**

```go
type _defer struct {
	siz     int32 // 参数与返回值共占用字节数，includes both arguments and results
	started bool // 是否已经执行
	heap    bool // 标记是否为堆分配
	// openDefer indicates that this _defer is for a frame with open-coded
	// defers. We have only one defer record for the entire frame (which may
	// currently have 0, 1, or more defers active).
	openDefer bool // go1.14添加用于栈扫描
	sp        uintptr  // 注册defer的函数栈指针，用于判断自己注册的defer是否已经执行完成，sp at time of defer
	pc        uintptr  // deferproc的返回地址，pc at time of defer
	fn        *funcval // 要注册的函数，表现为function value，可以有捕获列表，can be nil for open-coded defers
	_panic    *_panic  // 指明defer由哪个panic触发， panic that is running defer
	link      *_defer // 下一个defer

  // go1.14添加用于栈扫描
	// If openDefer is true, the fields below record values about the stack
	// frame and associated function that has the open-coded defer(s). sp
	// above will be the sp for the frame, and pc will be address of the
	// deferreturn call in the function.
	fd   unsafe.Pointer // funcdata for the function associated with the frame
	varp uintptr        // value of varp for the stack frame
	// framepc is the current pc associated with the stack frame. Together,
	// with sp above (which is the sp associated with the stack frame),
	// framepc/sp can be used as pc/sp pair to continue a stack trace via
	// gentraceback().
	framepc uintptr
}
```

#### 10. panic 和 recover

panic执行defer的逻辑：先标记，后释放【为了终止之前发生的panic】

panic打印异常信息（没有recovery）的逻辑：按照panic的顺序

panic的处理流程中，会在每个defer执行完后，都会检查当前panic是否已经被defer所恢复，若已经被恢复则从panic链表中移除，此后相应的defer也会被移除，在移除之前保存defer的sp和defer的pc，然后跳出panic的处理流程，跳转到deferreturn，由其负责当前函数注册的defer函数的执行。

recover函数会修改panic的一个参数--recovred为true，然后移除并跳出当前panic（在发生recover的函数正常返回以后，才进入检测panic是否被恢复的流程，然后才能删除被恢复的panic。）如果发生recover的defer函数在返回之前又发生panic，则由新的panic去执行defer，此时会发现defer链表头已经由之前的panic执行完成了，则将相应的panic标记为已终止，并将defer函数从链表中移除，继续执行下一个defer函数，输出异常信息时也是从尾到头，对于panic链表中被恢复到panic，加上recovered标记，每一项都输出后，程序退出。

```go
type _panic struct {
	argp      unsafe.Pointer // defer参数空间地址，pointer to arguments of deferred call run during panic; cannot move - known to liblink
	arg       interface{}    // panic的参数，argument to panic
	link      *_panic        // 指向之前发生的panic，link to earlier panic
	pc        uintptr        // where to return to in runtime if this panic is bypassed
	sp        unsafe.Pointer // where to return to in runtime if this panic is bypassed
	recovered bool           // 是否被恢复，whether this panic is over
	aborted   bool           // 是否被终止，the panic was aborted
	goexit    bool
}
```

#### 11. 类型系统

类型元数据共同构成了Go语言的类型系统。

```go
// _type作为每个类型元数据的header，其后存储额外描述信息，自定义类型则还需要uncommontype结构体
type _type struct {
	size       uintptr
	ptrdata    uintptr // size of memory prefix holding all pointers
	hash       uint32
	tflag      tflag
	align      uint8
	fieldAlign uint8
	kind       uint8
	// function for comparing objects of this type
	// (ptr to object A, ptr to object B) -> ==?
	equal func(unsafe.Pointer, unsafe.Pointer) bool
	// gcdata stores the GC type data for the garbage collector.
	// If the KindGCProg bit is set in kind, gcdata is a GC program.
	// Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
	gcdata    *byte
	str       nameOff
	ptrToThis typeOff
}
// 举个🌰，[]string的类型元数据
type slicetype struct {
	typ  _type
	elem *_type // -> string类型元数据
} 
// 自定义类型则还需要uncommontype结构体
type uncommontype struct {
	pkgpath nameOff // 记录包路径
	mcount  uint16 // 方法数目，number of methods
	xcount  uint16 // number of exported methods
	moff    uint32 // 方法元数据组成的数组相对于uncommontype的偏移字节数，offset from this uncommontype to [mcount]method
	_       uint32 // unused
}
// 方法描述信息
type method struct {
	name nameOff
	mtyp typeOff
	ifn  textOff
	tfn  textOff
}
```

```go
// 举个🌰
type myslice []string

func(ms myslice) Len(){
  fmt.Println(len(ms))
}

func(ms myslice) Cap(){
  fmt.Println(cap(ms))
}
==============
// 对应的类型元数据结构
slicetype
uncommontype //addr
...
method[0] //addr+moff
method[1]
```

```go
type MyType1 = int32 // 别名，都指向int32类型元数据
type MyType2 int32 // 自定义类型 ，有自己的类型元数据，即使与int32类型元数据相同
```

接口

```go
// 空接口
type eface struct {
	_type *_type // 动态类型
	data  unsafe.Pointer // 动态值
}

==== 空接口变量赋值前后变量
var e interface{} // _type=nil,data=nil
f,_:=os.Open("eggo.txt") // *os.File类型元素包括_type,...,uncommontype
e=f // _type=*os.File类型元数据,data=f
// 非空接口
type iface struct {
	tab  *itab // 方法列表，接口动态类型信息
	data unsafe.Pointer // 动态值
}

type itab struct {
	inter *interfacetype // 接口的类型元数据
	_type *_type // 动态类型元数据
	hash  uint32 // 类型哈希值，用于快速判断类型是否相同，copy of _type.hash. Used for type switches.
	_     [4]byte
	fun   [1]uintptr // 方法地址数组，接口要求的方法地址，会从动态类型元数据中拷贝接口要求的方法的地址， variable sized. fun[0]==0 means _type does not implement inter. 断言失败的类型组合对应的itab结构体也会被缓存起来
}

type interfacetype struct {
	typ     _type
	pkgpath name
	mhdr    []imethod //接口要求的方法
}
==== 非空接口赋值前后变化
var rw io.ReadWriter // tab=nil,data=nil
f,_:=os.Open("eggo.txt")
rw = f // tab见下，data=f
====
tab
 |                
itab [ inter, _type, hash, _, fun[0], fun[1]]
         |      |
         |      *os.File类型元数据 [_type ... uncommontype]
io.ReadWriiter interfacetype [typ, pkgpath, mhdr[0],mhdr[1]]
                                               |      | 
                                              Read   Writer
一旦接口类型（inter）确定了，则动态类型(_type)也确定了，那么itab的内容就不会改变了，所以itab结构体是可复用的。Go会以接口类型和动态类型的组合作为key(接口类型的hash值与动态类型的hash值进行异或运算)，将itab结构体缓存起来【哈希表，不同于map，一种更为简便的设计】

```

类型断言

```go
// 空接口.(具体类型)
var e interface{}
f,_:=os.Open("eggo.txt") // f="eggo"
e = f
r,ok:=e.(*os.File) // 判断e._type == *os.File类型元数据

// 非空接口.(具体类型)
var rw io.ReadWriter
f,_:=os.Open("eggo.txt") // f="eggo"
rw = f
r,ok:=rw.(*os.File) // 判断rw.tab == itab缓存中对应的<io.ReadWriter,*os.File>对应的itab结构体地址

// 空接口.(非空接口)
var e interface{}
f,_:=os.Open("eggo.txt") // f="eggo"
rw = f // e._type=*os.File类型元数据，e.data=f
rw,ok:=e.(io.ReadWriter) // 在itab缓存中查找<io.ReadWriter,*os.File>，并检查itab.fun[0]==0，否则再去检查*os.File的方法列表

// 非空接口.(非空接口)
var w io.Writer
f,_:=os.Open("eggo.txt")
w = f 
rw,ok:=w.(io.ReadWriter) // <io.ReadWriter,*os.File>  
```

#### 12. reflect

runtime中定义的类型元数据、空接口、非空接口的结构体并未导出，所以在reflect包中也定义了 一套，二者保持一致。

```go
//提供TypeOf函数，用于获取变量的类型信息
func TypeOf(i interface{}) Type {
	eface := *(*emptyInterface)(unsafe.Pointer(&i))
	return toType(eface.typ)
}
type Type interface {
	Align() int // 对齐边界

	// FieldAlign returns the alignment in bytes of a value of
	// this type when used as a field in a struct.
	FieldAlign() int

	// Method returns the i'th method in the type's method set.
	// It panics if i is not in the range [0, NumMethod()).
	//
	// For a non-interface type T or *T, the returned Method's Type and Func
	// fields describe a function whose first argument is the receiver.
	//
	// For an interface type, the returned Method's Type field gives the
	// method signature, without a receiver, and the Func field is nil.
	//
	// Only exported methods are accessible and they are sorted in
	// lexicographic order.
	Method(int) Method // 方法

	// MethodByName returns the method with that name in the type's
	// method set and a boolean indicating if the method was found.
	//
	// For a non-interface type T or *T, the returned Method's Type and Func
	// fields describe a function whose first argument is the receiver.
	//
	// For an interface type, the returned Method's Type field gives the
	// method signature, without a receiver, and the Func field is nil.
	MethodByName(string) (Method, bool)

	// NumMethod returns the number of exported methods in the type's method set.
	NumMethod() int

	// Name returns the type's name within its package for a defined type.
	// For other (non-defined) types it returns the empty string.
 	Name() string // 类型名称

	// PkgPath returns a defined type's package path, that is, the import path
	// that uniquely identifies the package, such as "encoding/base64".
	// If the type was predeclared (string, error) or not defined (*T, struct{},
	// []int, or A where A is an alias for a non-defined type), the package path
	// will be the empty string.
	PkgPath() string //包路径

	// Size returns the number of bytes needed to store
	// a value of the given type; it is analogous to unsafe.Sizeof.
	Size() uintptr

	// String returns a string representation of the type.
	// The string representation may use shortened package names
	// (e.g., base64 instead of "encoding/base64") and is not
	// guaranteed to be unique among types. To test for type identity,
	// compare the Types directly.
	String() string

	// Kind returns the specific kind of this type.
	Kind() Kind

	// Implements reports whether the type implements the interface type u.
	Implements(u Type) bool // 是否实现了指定接口

	// AssignableTo reports whether a value of the type is assignable to type u.
	AssignableTo(u Type) bool

	// ConvertibleTo reports whether a value of the type is convertible to type u.
	ConvertibleTo(u Type) bool

	// Comparable reports whether values of this type are comparable.
	Comparable() bool

	// Methods applicable only to some types, depending on Kind.
	// The methods allowed for each kind are:
	//
	//	Int*, Uint*, Float*, Complex*: Bits
	//	Array: Elem, Len
	//	Chan: ChanDir, Elem
	//	Func: In, NumIn, Out, NumOut, IsVariadic.
	//	Map: Key, Elem
	//	Ptr: Elem
	//	Slice: Elem
	//	Struct: Field, FieldByIndex, FieldByName, FieldByNameFunc, NumField

	// Bits returns the size of the type in bits.
	// It panics if the type's Kind is not one of the
	// sized or unsized Int, Uint, Float, or Complex kinds.
	Bits() int

	// ChanDir returns a channel type's direction.
	// It panics if the type's Kind is not Chan.
	ChanDir() ChanDir

	// IsVariadic reports whether a function type's final input parameter
	// is a "..." parameter. If so, t.In(t.NumIn() - 1) returns the parameter's
	// implicit actual type []T.
	//
	// For concreteness, if t represents func(x int, y ... float64), then
	//
	//	t.NumIn() == 2
	//	t.In(0) is the reflect.Type for "int"
	//	t.In(1) is the reflect.Type for "[]float64"
	//	t.IsVariadic() == true
	//
	// IsVariadic panics if the type's Kind is not Func.
	IsVariadic() bool

	// Elem returns a type's element type.
	// It panics if the type's Kind is not Array, Chan, Map, Ptr, or Slice.
	Elem() Type

	// Field returns a struct type's i'th field.
	// It panics if the type's Kind is not Struct.
	// It panics if i is not in the range [0, NumField()).
	Field(i int) StructField

	// FieldByIndex returns the nested field corresponding
	// to the index sequence. It is equivalent to calling Field
	// successively for each index i.
	// It panics if the type's Kind is not Struct.
	FieldByIndex(index []int) StructField

	// FieldByName returns the struct field with the given name
	// and a boolean indicating if the field was found.
	FieldByName(name string) (StructField, bool)

	// FieldByNameFunc returns the struct field with a name
	// that satisfies the match function and a boolean indicating if
	// the field was found.
	//
	// FieldByNameFunc considers the fields in the struct itself
	// and then the fields in any embedded structs, in breadth first order,
	// stopping at the shallowest nesting depth containing one or more
	// fields satisfying the match function. If multiple fields at that depth
	// satisfy the match function, they cancel each other
	// and FieldByNameFunc returns no match.
	// This behavior mirrors Go's handling of name lookup in
	// structs containing embedded fields.
	FieldByNameFunc(match func(string) bool) (StructField, bool)

	// In returns the type of a function type's i'th input parameter.
	// It panics if the type's Kind is not Func.
	// It panics if i is not in the range [0, NumIn()).
	In(i int) Type

	// Key returns a map type's key type.
	// It panics if the type's Kind is not Map.
	Key() Type

	// Len returns an array type's length.
	// It panics if the type's Kind is not Array.
	Len() int

	// NumField returns a struct type's field count.
	// It panics if the type's Kind is not Struct.
	NumField() int

	// NumIn returns a function type's input parameter count.
	// It panics if the type's Kind is not Func.
	NumIn() int

	// NumOut returns a function type's output parameter count.
	// It panics if the type's Kind is not Func.
	NumOut() int

	// Out returns the type of a function type's i'th output parameter.
	// It panics if the type's Kind is not Func.
	// It panics if i is not in the range [0, NumOut()).
	Out(i int) Type

	common() *rtype
	uncommon() *uncommonType
}
```

```go
package main

import "reflect"
type Eggo struct{
	Name string
}
func(e Eggo)A(){
	println("A")
}
func(e Eggo)B(){
	println("B")
}
func main(){
	a:=Eggo{Name:"eggo"}
	t:=reflect.TypeOf(a) // 当参数是非接口类型，编译器会增加一个临时变量作为a的拷贝，在参数空间中使用copy of a的地址。
	println(t.Name(),t.NumMethod())	
}
```

```go
type Value struct {
	// typ holds the type of the value represented by a Value.
	typ *rtype // 反射变量的类型元数据指针

	// Pointer-valued data or, if flagIndir is set, pointer to data.
	// Valid when either flagIndir is set or typ.pointers() is true.
	ptr unsafe.Pointer // 数据地址

	// flag holds metadata about the value.
	// The lowest bits are flag bits:
	//	- flagStickyRO: obtained via unexported not embedded field, so read-only
	//	- flagEmbedRO: obtained via unexported embedded field, so read-only
	//	- flagIndir: val holds a pointer to the data
	//	- flagAddr: v.CanAddr is true (implies flagIndir)
	//	- flagMethod: v is a method value.
	// The next five bits give the Kind of the value.
	// This repeats typ.Kind() except for method values.
	// The remaining 23+ bits give a method number for method values.
	// If flag.kind() != Func, code can assume that flagMethod is unset.
	// If ifaceIndir(typ), code can assume that flagIndir is set.
	flag // 位标识符，存储反射值的一些描述信息

	// A method value represents a curried method invocation
	// like r.Read for some receiver r. The typ+val+flag bits describe
	// the receiver r, but the flag's Kind bits say Func (methods are
	// functions), and the top bits of the flag give the method number
	// in r's type's method table.
}
func ValueOf(i interface{}) Value {
	if i == nil {
		return Value{}
	}

	// TODO: Maybe allow contents of a Value to live on the stack.
	// For now we make the contents always escape to the heap. It
	// makes life easier in a few places (see chanrecv/mapassign
	// comment below).
	escapes(i) // 显式将参数指向的变量逃逸到堆上

	return unpackEface(i)
}
```

```go
func main(){
  a:="eggo"
  v:=reflect.ValueOf(&a) // 必须是指针，否则无法修改
  v=v.Elem()
  v.SetString("new eggo")
  println(a)
}
```

#### 13.  GMP

代码经过编译后得到可执行文件，在代码段有程序入口，不同平台下程序执行入口，进行一系列检查与准备工作后，调度器初始化（创建P保存在`allp`中，并将第一个P与`m0`关联起来），创建main goroutine（会有自己的协程栈main goroutine stack）， 执行调度，会以runtime.main作为执行入口（其中调用main.main，在main.main返回之后，调用`exit()`结束进程，如果在main.main中创建了新goroutine【负责指定入口、参数，newproc床构造一个栈帧，使得协程任务结束后返回`goexit()`中进行协程资源回收处理等工作】）

数据段中有几个全局变量，`g0`维护主协程（`g0`的协程栈分配在线程栈上） ，`m0`维护主线程，`allgs`、`allm`和`allp`分别记录所有的协程、线程以及P，`sched`中维护有一个全局的存放待运行的G的队列，同时记录空闲的M和空闲的P，记录上次netpoll执行时间

引入P，维护自有的待执行的G的集合，并将P与M关联起来，M运行时从P处获取待执行的G，如果本地队列没有则从全局队列中领取任务，否则从其他的P“偷取“一些G。

创建协程时调用的函数是newproc，其栈帧分配在协程栈空间，然后会调用newproc1，但它栈帧分配在主协程栈上（因为分配在主线程的栈中，空间足够大），传递的参数有协程入口，参数地址，参数大小，父协程，返回地址。其中调用过程：禁止当前m被抢占（接下来的程序中可能会当前P保存在局部变量中，如果M被抢占，P关联到别的M，再次恢复时使用的P就会造成问题），尝试获取一个空闲G，获取失败则创建一个并添加到全局变量`allgs`中，此时状态是`_Gdead`，【参数空间，返回地址，被启动协程的函数栈帧】（如果协程入口函数有参数则将参数移动到协程栈上，将`&goexit+1`压入协程栈，并置协程入口函数起始地址、父协程调用newproc后的地址，保存现场（协程栈指针，协程入口函数的起始地址），赋值一个唯一id，在赋值前将协程状态设置为`—Grunnable`）【像是在goexit函数中调用协程入口函数，并传递用户传入的参数，指针刚跳转到协程入口处，但还没有开始执行的状态】，等这个协程被调度执行时，恢复现场，则从协程入口处开始执行，协程函数结束后便会返回到`goexit()`中，执行资源回收等收尾工作。
接下来将协程放到P的本地队列中，如果有空闲的P（`_Pidle`），而且没有处于spinning状态的M（所有的M都在忙），则启动一个M并置为spinning状态，然后允许当前M被抢占。

协程让出`gopark()`：首先禁止当前m被抢占，将当前goroutine修改为`_Gwaitting`，然后允许M被抢占，调用`mcall`（保存协程的执行现场，切换到`g0`栈，调用`runtime.park_m`（根据`g0`找到当前M，将curg置为nil，调用`schedule()`寻找下一个待执行的G））

协程恢复`goready()`会切换到`g0`栈，执行`runtime.ready()`（禁止当前m被抢占，协程状态修改为`_Grunnable`，放入当前P的本地队列中，检查是否有空闲的P，并检查是否有空闲的M，否则启动新的M，允许当前M被抢占）

每个P持有一个最小堆，用于管理自己的timer，堆顶timer就是接下来要触发的，每次调用`schedule()`是都会调用`checkTimers()`检查到时间的Timer，为了防止所有M都在忙导致时间延误，有一个监控线程（由main goroutine创建，非工作线程，不依赖P，不由GMP模型调度，重复执行一系列任务【1.保证Timer按时执行，在没有空闲M时创建新的工作线程，保障Timer顺利执行；2.按需执行netpoll--调度器、GC等过程中也会按需执行netpoll，3. 对运行时间过长的G进行抢占，借助P中的schedtick字段和插入到函数头部的栈增长代码和stackPreempt---无法处理与栈增长无关的情况，所以在go1.14后引入了**异步抢占**--如Unix中依赖sigPreempt信号，线程中断，转而执行runtime.sigHandler，检测信号为sigPreempt后调用runtime.doSigPreemp向当前被打断的协程上下文中注入异步抢占函数调用，处理完后sigHandler返回，被中断的协程恢复，执行异步抢占函数，保存现场并执行调度函数；4.强制执行GC】，会自己调整休眠时间）

系统调用时，M和G进行了绑定，在陷入系统调用前，当前M让出P并记录这个P，此时P可能会被其他M所占用，当M从系统调用中恢复后， 检查之前的P是否被占用，否则就申请一个，没有申请到P则将当前G放到全局队列中，然后当前M就睡眠了。

`schedule()`：首先确定当前M是否和当前G绑定了，如果绑定了则当前M不能执行其他G ，阻塞当前M，等到当前G再次被调度执行时自然会唤醒M。没有被绑定则检查GC是否等待执行，如果GC在等待执行则去执行GC，执行完成后再执行调度程序。检查有没有要执行的Timer，调度程序还有几率会从全局队列中获取一部分G到本地队列中，获调用find runnable()获取待运行的G（检查是否执行GC，尝试从本地队列中寻找->全局队列->尝试执行netpoll【IO调用就绪的G，会被放到全局队列中】->从其他P处“偷取”G），如果当前G有绑定的M，则将G返回给对应的M，当前M继续执行调度。否则就在当前M上继续执行这个G（execute() --- 建立当前M和G的关联关系，修改G的状态，根据是否继承时间片决定调度计数，调用gogo函数从g.sched中恢复协程栈指针，指令指针等继续执行）

#### 14. channel

```go
type hchan struct {
	qcount   uint           // 缓冲区大小，total data in the queue
	dataqsiz uint           // 缓冲区元素类型大小， size of the circular queue
	buf      unsafe.Pointer // 缓冲区地址，指向一个环形队列，points to an array of dataqsiz elements
	elemsize uint16 // 元素类型大小
	closed   uint32 // channel是否已关闭
	elemtype *_type // 元素元数据类型，element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // 读队列，list of recv waiters
	sendq    waitq  // 写队列，list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}
type waitq struct {
	first *sudog
	last  *sudog
}
type sudog struct {
	// The following fields are protected by the hchan.lock of the
	// channel this sudog is blocking on. shrinkstack depends on
	// this for sudogs involved in channel ops.

	g *g // 记录的等待的G 

	// isSelect indicates g is participating in a select, so
	// g.selectDone must be CAS'd to win the wake-up race.
	isSelect bool
	next     *sudog
	prev     *sudog
	elem     unsafe.Pointer // 数据位置，data element (may point to stack)

	// The following fields are never accessed concurrently.
	// For channels, waitlink is only accessed by g.
	// For semaphores, all fields (including the ones above)
	// are only accessed when holding a semaRoot lock.

	acquiretime int64
	releasetime int64
	ticket      uint32
	parent      *sudog // semaRoot binary tree
	waitlink    *sudog // g.waiting list or semaRoot
	waittail    *sudog // semaRoot
	c           *hchan // channel
} 
```

#### 15.GC

标记清扫算法核心思想：可达性 近似等价于 存活性， 将栈和数据段上的数据对象作为root，基于它们进一步追踪，将能追踪到的数据都进行标记，追踪不到的就是垃圾

三色抽象：所有节点都为白色，root标记为灰色（代表当前节点尚未完成追踪），追踪任务完成后标记为黑色（存活数据，基于黑色节点找到的所有节点都被标记为灰色），直到所有灰色节点清空。

对于内存碎片化 -- 基于Big Bag Of Pages思想，划分多种规格 or 移动数据减少（标记整理算法） or 复制式回收（内存减半） 

复制式回收一般与其他回收算法结合使用，如分代回收--基于弱分代假说（大部分对象都在年轻时死亡），降低老年代对象的垃圾回收频率，并且新生代和老年代对象可以分别采用不同的回收策略，提升回收效益减少开销

引用计数式垃圾回收：回收成本分摊，但高频率的更新引用计数也会带来开销，无法解决循环引用的情况

Stop The World -- 需要将垃圾回收工作分成多次完成，即用户程序和垃圾回收交替执行，称之为增量式垃圾回收。其额外的问题 是 因为用户程序执行导致节点数据对象的存活性被误判。

强三色不变式：不出现黑色对象到白色对象的引用

弱三色不变式：允许出现黑色对象到白色对象的引用，但保证可以通过灰色对象抵达该白色对象

实现方案是 -- 建立读写屏障

写屏障：通常会有一个记录集，采用顺序存储还是哈希表，是精确记录还是定位到页的方式取决于具体实现 

​	插入写屏障：出现黑色对象到白色对象的引时，将黑色对象引用的白色对象标记为灰色，或将黑色对象退化为灰色

​	删除写屏障：删除灰色对象到白色对象的引用时，将白色对象着为灰色

读屏障：非移动式垃圾回收器中不需要，但在复制式回收时需要。确保用户不会访问到已存在副本的陈旧对象。 

并行垃圾回收时要处理同步问题，重复处理问题

并发垃圾回收指用户程序与垃圾回收并发执行，多核场景下存在用户程序和垃圾回收可能会同时使用写屏障记录集，需要考虑竞争问题。

主体并发式垃圾回收：在某些阶段采取STW方式，其他阶段支持并发。

同时可以支持主体并发增量式回收 --- golang支持

Golang中GC过程：

- 准备阶段会为每个P创建mark worker协程，将对应的g指针存储在p中，等到标记阶段得到调度执行，并确保完成了上一轮的清扫工作
- 第一次STW，GC进入_GCMark阶段，开启写屏障，允许标记工作，结束后所有P都知道写屏障已开启，Mark worker可以得到调度执行，展开标记工作
- 没有标记任务时，第二次STW，GC进入\_GCMarkTermination阶段，确认标记工作确实已完成，停止标记工作，接下里进入\_GCOFF阶段，关闭写屏障【\_GCOFF阶段前分配的对象是黑色，之后则是白色】
- 清扫工作协程（由runtime.main在gcenable中创建，对应g指针存在全局变量sweep中）在_GCOFF阶段得到调度执行

#### 16. 方法集

T类型的方法集在编译时会被包装为*T类型的方法的一部分 => 支持接口，因为接口是动态派发的，所以像下面那种语法糖形式会无法确定是哪类数据，无法生成对应的指令来解引用，即无法确定参数类型。***即接口不能直接使用接收者为值类型的方法 **

但对于*T类型的变量调用T类型的方法其实是一种语法糖，会在调用端进行指针解引用，并不会用到上面的包装方法

#### 17. Mutex

```go
type Mutex struct {
	state int32 // 锁状态，第一位是锁状态标识，第二位记录是否已有goroutine被唤醒，第三位表示Mutex的工作模式，其他位记录有多少个等待者
	sema  uint32 // 信号量 
}
func (m *Mutex) Lock() {
	// Fast path: grab unlocked mutex.
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) { // 原子操作
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
	// Slow path (outlined so that the fast path can be inlined)，内联优化？
	m.lockSlow()
}
func (m *Mutex) Unlock() {
	if race.Enabled {
		_ = m.state
		race.Release(unsafe.Pointer(m))
	}

	// Fast path: drop lock bit.
	new := atomic.AddInt32(&m.state, -mutexLocked) // 原子操作
	if new != 0 {
		// Outlined slow path to allow inlining the fast path.内联优化？
		// To hide unlockSlow during tracing we skip one extra frame when tracing GoUnblock.
		m.unlockSlow(new)
	}
}
```

尝试加锁的goroutine会先自旋几次，尝试通过原子操作获得锁，若几次自旋之后仍旧不能获得锁，则通过信号量排队等待（FIFO），锁被释放时，并不会直接拥有锁，而是需要和后来者（处于自旋阶段尚未排队的goroutine）竞争（因为自旋中的goroutine正在CPU上运行，优先级高），此时很大概率拿不到锁（因为自旋中的goroutine可能很多），此时会被重新插入到队列头部。

当一个goroutine本次加锁等待的时间超过1ms后，将锁从正常模式切换为饥饿模式。

饥饿模式下，Mutex的所有权从执行Unlock的goroutine直接传递给等待队列头部的goroutine，后来者不会自旋，也不会尝试获得锁（即使Mutex处于Unlock状态），直接进入等待队列尾部。当一个等待者获得锁之后，如果其等待时间小于1ms或者其是等待队列最后一个等待者则将Mutex从饥饿状态转成正常状态。

正常模式是为了并发量，因为频繁挂起、唤醒goroutine会带来开销，同时为了限制自旋的开销，所以有次数的限制。

饥饿模式是为了防止尾端延迟。

即使在正常模式下，当一个goroutine尝试给Mutex加锁时，如果其他goroutine已经加锁但还没释放，也不一定开始自旋：单核场景（或没有其他的P正在运行 ）下，自旋的goroutine等待持有锁的goroutine释放锁，而持有锁的goroutine在等待自旋的goroutine让出CPU，此时自旋没有意义。除此之外，如果当前P的本地队列不为空，切换到本地goroutine更高效而不是自旋。

=> 多核 && GOMAXPROCS>1 && 至少有一个其他的P正在运行 && 当前P的本地队列为空 才会发生自旋

自旋次数超过4次 or 锁被释放 or 饥饿模式 => 结束自旋

#### 18. 信号量semaphore

通过一个大小固定的sematable来管理执行阶段数量不定的semaphore：table中存放的是251个平衡树的根，平衡树中每个节点都是一个sudog类型的对象，使用信号量时，需要提供一个记录信号量数值的变量，根据地址进行计算，映射到table中的一棵平衡树，找到对应的节点，从而找到该信号量的等待队列。

举个🌰：Mutex是通过信号量来排队的，而channel需要有接收等待队列和发送等待队列，还要有缓冲区功能，所以没有直接使用信号量来实现排队，而是自行实现了一套排队逻辑，都与runtime.mutex相关【解决多线程并发时的同步问题】

#### 19. 内存逃逸

（1）什么是内存逃逸，原理是什么？

内存逃逸是指原本应分配在栈中的数据被分配到堆上，原理是一个值被分享到函数栈帧之外，就会在堆上分配

（2）在什么情况下发生内存逃逸？常见场景有哪些？

如闭包捕获被修改的局部变量，返回局部变量的指针，发送指针到channel中，切片上存储指针或者带有指针到值、slice底层数组重新分配、接口类型调用方法

#### 20. new 和 make的区别（异同）

new	在堆上分配内存，为值类型分配空间，入参为一个类型，申请该类型大小的内存空间，并初始化为对应的零值，返回相应类型的指针，可以使用字面值快速初始化。

make 在堆上分配内存，只用于slice、map、channel等分配空间，返回类型是本身，make函数在初始化时会初始化相应类型的数据结构（长度、容量），但new不会。

#### 21. 

