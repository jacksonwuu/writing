---
title: Go Interface 实现
date: 2024-06-29
tags:
    - Go
permalink: /go-interface/
---

这篇文章主要记录了一些我在学习 Go 语言实现原理过程中的笔记。

本人使用的 `go version go1.22.4 darwin/amd64`。

## Go interface

interface 是 Go 语言中很有意思的一个特性，它能让你像纯动态语言一样使用`duck typing`，而且编译器还能像静态语言一样做类型检查。

> 如果一个东西叫起来像鸭子，走路也像鸭子，那么它就是鸭子。

也就是说，interface 允许你定义一组方法的集合，任何实现了这组方法的类型都可以被看作是这个 interface 的实现。

有了 interface，可以做很多有意思的事情。

1、可以通过 interface 来抽象，然后再通过组合与继承，让程序足够有表达力。

2、如果不用 interface，一个函数就只能接受具体的类型作为参数，而且也只能返回具体的类型作为返回值，这个函数的适用范围就比较局限。有了 interface，就能让这个函数接受和返回更抽象的类型的参数（某个 interface），相当于让这个函数的适用范围更广了。

3、interface 让程序可以很方便地实现各种编程模式，提升程序的抽象性和解耦度，让程序更容易维护和扩展。

---

下面我们主要关注 interface 的底层实现原理，也就是要理解这段代码的原理（直接用 AI 生成几段代码给我们）：

类型断言

```go
var i interface{} = 42
n, ok := i.(int) // n == 42, ok == true
s, ok := i.(string) // s == "", ok == false
```

类型转化：把一种类型转化为

```go

// 定义一个接口
type Speaker interface {
    Speak() string
}

// 定义一个实现了该接口的具体类型
type Dog struct{}

var d Dog

// 具体类型 转 空接口
var a interface{} = d

// 空接口 转 特定接口
var s Speaker
s = a.(Speaker) // 使用类型断言

```

类型 switch：尝试把某个变量转化为其他类型，是类型断言和类型转化的组合。

```go
switch v := i.(type) {
case T1:
    // v 是 T1 类型
case T2:
    // v 是 T2 类型
default:
    // v 是其他类型
}
```

动态派发：在运行的时候确定应该调用哪个函数。

```go
type Speaker interface {
    Speak() string
}

type Dog struct{}

func (d Dog) Speak() string {
    return "Woof!"
}

type Cat struct{}

func (c Cat) Speak() string {
    return "Meow!"
}

func makeSound(s Speaker) {
    fmt.Println(s.Speak())
}

func main() {
    d := Dog{}
    c := Cat{}
    makeSound(d) // 输出 "Woof!"
    makeSound(c) // 输出 "Meow!"
}
```

## Go interface 的实现原理

interface 在 Go 语言的实现其实就靠一个叫做 `iface` 的数据结构。

```go

type iface struct {
    tab  *itab
    data unsafe.Pointer // 数据指针
}

type itab struct {
	inter *interfacetype
	_type *_type
	hash  uint32 // copy of _type.hash. Used for type switches.
	_     [4]byte
	fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}

```

把所有子数据结构也展开，就是如下：

```go
type iface struct { // `iface`
    tab *struct { // `itab`
        inter *struct { // `interfacetype`
            typ struct { // `_type`
                size       uintptr
                ptrdata    uintptr
                hash       uint32
                tflag      tflag
                align      uint8
                fieldalign uint8
                kind       uint8
                alg        *typeAlg
                gcdata     *byte
                str        nameOff
                ptrToThis  typeOff
            }
            pkgpath name
            mhdr    []struct { // `imethod`
                name nameOff
                ityp typeOff
            }
        }
        _type *struct { // `_type`
            size       uintptr
            ptrdata    uintptr
            hash       uint32
            tflag      tflag
            align      uint8
            fieldalign uint8
            kind       uint8
            alg        *typeAlg
            gcdata     *byte
            str        nameOff
            ptrToThis  typeOff
        }
        hash uint32
        _    [4]byte
        fun  [1]uintptr
    }
    data unsafe.Pointer
}
```

用图来表示就是这样：

![](2024-06-go-interface/iface.png)

理解 `iface` 的运作原理的最好方式就是直接查看其汇编代码。

写一段简单的代码：

```go
package main

type Speaker interface {
	Speak() string
}

type Dog struct {
}

//go:noinline
func (*Dog) Speak() string {
	return "Dog"
}

type Cat struct {
}

//go:noinline
func (*Cat) Speak() string {
	return "Cat"
}

//go:noinline
func JustSpeak(s Speaker) string {
	return s.Speak()
}

//go:noinline
func IsSpeaker(s interface{}) bool {
	v, ok := s.(Speaker)
	if ok {
		v.Speak()
	}
	return ok
}

//go:noinline
func Which(s interface{}) string {
	switch s.(type) {
	case Cat:
		return "Cat"
	case Dog:
		return "Dog"
	case Speaker:
		return "Speaker"
	default:
		return "interface{}"
	}
}

//go:noinline
func DogSpeak() {
	var s Speaker = &Dog{}
	JustSpeak(s)
}

func main() {
}
```

然后直接一个命令把代码编译为 Plan 9 汇编代码：

`GOOS=linux GOARCH=amd64 go build -gcflags="-S" ./main.go 2> ./main.S`

先来看 `JustSpeak` 函数的汇编代码：

```
main.JustSpeak STEXT size=67 args=0x10 locals=0x10 funcid=0x0 align=0x0
	0x0000 00000 (/main.go:24)	TEXT	main.JustSpeak(SB), ABIInternal, $16-16
	0x0000 00000 (/main.go:24)	CMPQ	SP, 16(R14)
	0x0004 00004 (/main.go:24)	PCDATA	$0, $-2
	0x0004 00004 (/main.go:24)	JLS	40
	0x0006 00006 (/main.go:24)	PCDATA	$0, $-1
	0x0006 00006 (/main.go:24)	PUSHQ	BP
	0x0007 00007 (/main.go:24)	MOVQ	SP, BP
	0x000a 00010 (/main.go:24)	SUBQ	$8, SP
	0x000e 00014 (/main.go:24)	MOVQ	AX, main.s+24(FP)
	0x0013 00019 (/main.go:24)	MOVQ	BX, main.s+32(FP)
	0x0018 00024 (/main.go:24)	FUNCDATA	$0, gclocals·IuErl7MOXaHVn7EZYWzfFA==(SB)
	0x0018 00024 (/main.go:24)	FUNCDATA	$1, gclocals·J5F+7Qw7O7ve2QcWC7DpeQ==(SB)
	0x0018 00024 (/main.go:24)	FUNCDATA	$5, main.JustSpeak.arginfo1(SB)
	0x0018 00024 (/main.go:24)	FUNCDATA	$6, main.JustSpeak.argliveinfo(SB)
	0x0018 00024 (/main.go:24)	PCDATA	$3, $1
	0x0018 00024 (/main.go:25)	MOVQ	24(AX), CX
	0x001c 00028 (/main.go:25)	MOVQ	BX, AX
	0x001f 00031 (/main.go:25)	PCDATA	$1, $1
	0x001f 00031 (/main.go:25)	NOP
	0x0020 00032 (/main.go:25)	CALL	CX
	0x0022 00034 (/main.go:25)	ADDQ	$8, SP
	0x0026 00038 (/main.go:25)	POPQ	BP
	0x0027 00039 (/main.go:25)	RET
	0x0028 00040 (/main.go:25)	NOP
	0x0028 00040 (/main.go:24)	PCDATA	$1, $-1
	0x0028 00040 (/main.go:24)	PCDATA	$0, $-2
	0x0028 00040 (/main.go:24)	MOVQ	AX, 8(SP)
	0x002d 00045 (/main.go:24)	MOVQ	BX, 16(SP)
	0x0032 00050 (/main.go:24)	CALL	runtime.morestack_noctxt(SB)
	0x0037 00055 (/main.go:24)	PCDATA	$0, $-1
	0x0037 00055 (/main.go:24)	MOVQ	8(SP), AX
	0x003c 00060 (/main.go:24)	MOVQ	16(SP), BX
	0x0041 00065 (/main.go:24)	JMP	0
	0x0000 49 3b 66 10 76 22 55 48 89 e5 48 83 ec 08 48 89  I;f.v"UH..H...H.
	0x0010 44 24 18 48 89 5c 24 20 48 8b 48 18 48 89 d8 90  D$.H.\$ H.H.H...
	0x0020 ff d1 48 83 c4 08 5d c3 48 89 44 24 08 48 89 5c  ..H...].H.D$.H.\
	0x0030 24 10 e8 00 00 00 00 48 8b 44 24 08 48 8b 5c 24  $......H.D$.H.\$
	0x0040 10 eb bd                                         ...
	rel 2+0 t=R_USEIFACEMETHOD type:main.Speaker+96
	rel 32+0 t=R_CALLIND +0
	rel 51+4 t=R_CALL runtime.morestack_noctxt+0
```

别被这一大段汇编代码吓到了，其实很好读懂。

最关键的是这一行：

```
	0x0020 00032 (/main.go:25)	CALL	CX
```

这就是动态调用的实现。

在调用这个函数之前，它会先构造一个 iface 来表示传入的 Speaker 类型的变量，然后它会把传入这个的参数的偏移 24 字节作为函数进行调用，实际上就是调用的 Speak() 函数。

那么 iface 是如何被构造出来的？继续看这段代码。

```
go:itab.*main.Dog,main.Speaker SRODATA dupok size=32
	0x0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0010 29 fe 4c 82 00 00 00 00 00 00 00 00 00 00 00 00  ).L.............
	rel 0+8 t=R_ADDR type:main.Speaker+0
	rel 8+8 t=R_ADDR type:*main.Dog+0
	rel 24+8 t=RelocType(-32767) main.(*Dog).Speak+0
main.DogSpeak STEXT size=50 args=0x0 locals=0x18 funcid=0x0 align=0x0
	0x0000 00000 (/main.go:56)	TEXT	main.DogSpeak(SB), ABIInternal, $24-0
	0x0000 00000 (/main.go:56)	CMPQ	SP, 16(R14)
	0x0004 00004 (/main.go:56)	PCDATA	$0, $-2
	0x0004 00004 (/main.go:56)	JLS	43
	0x0006 00006 (/main.go:56)	PCDATA	$0, $-1
	0x0006 00006 (/main.go:56)	PUSHQ	BP
	0x0007 00007 (/main.go:56)	MOVQ	SP, BP
	0x000a 00010 (/main.go:56)	SUBQ	$16, SP
	0x000e 00014 (/main.go:56)	FUNCDATA	$0, gclocals·g2BeySu+wFnoycgXfElmcg==(SB)
	0x000e 00014 (/main.go:56)	FUNCDATA	$1, gclocals·g2BeySu+wFnoycgXfElmcg==(SB)
	0x000e 00014 (/main.go:58)	LEAQ	go:itab.*main.Dog,main.Speaker(SB), AX
	0x0015 00021 (/main.go:58)	LEAQ	runtime.zerobase(SB), BX
	0x001c 00028 (/main.go:58)	PCDATA	$1, $0
	0x001c 00028 (/main.go:58)	NOP
	0x0020 00032 (/main.go:58)	CALL	main.JustSpeak(SB)
	0x0025 00037 (/main.go:59)	ADDQ	$16, SP
	0x0029 00041 (/main.go:59)	POPQ	BP
	0x002a 00042 (/main.go:59)	RET
	0x002b 00043 (/main.go:59)	NOP
	0x002b 00043 (/main.go:56)	PCDATA	$1, $-1
	0x002b 00043 (/main.go:56)	PCDATA	$0, $-2
	0x002b 00043 (/main.go:56)	CALL	runtime.morestack_noctxt(SB)
	0x0030 00048 (/main.go:56)	PCDATA	$0, $-1
	0x0030 00048 (/main.go:56)	JMP	0
	0x0000 49 3b 66 10 76 25 55 48 89 e5 48 83 ec 10 48 8d  I;f.v%UH..H...H.
	0x0010 05 00 00 00 00 48 8d 1d 00 00 00 00 0f 1f 40 00  .....H........@.
	0x0020 e8 00 00 00 00 48 83 c4 10 5d c3 e8 00 00 00 00  .....H...]......
	0x0030 eb ce                                            ..
	rel 2+0 t=R_USEIFACE type:*main.Dog+0
	rel 17+4 t=R_PCREL go:itab.*main.Dog,main.Speaker+0
	rel 24+4 t=R_PCREL runtime.zerobase+0
	rel 33+4 t=R_CALL main.JustSpeak+0
	rel 44+4 t=R_CALL runtime.morestack_noctxt+0
```

还记得上面的 `iface` 数据结构吗？

```go
type iface struct {
    tab  *itab
    data unsafe.Pointer // 数据指针
}
```

你不用很懂 Plan 9 Assembly 也可以看出来，这段代码的核心就是在把 `iface` 给构造出来并通过指针传递给下一个函数，而构造 `iface` 的核心步骤之一就是把 `go:itab.*main.Dog,main.Speaker` 复制到了 AX 上，而这是一个 rodata 只读数据，我也把它列出来了。

`go:itab.*main.Dog,main.Speaker` 看名字我们也知道，实际上就是 Dog 这个 struct 实现的 Speaker 的函数表（当然 itab 里也包含了变量类型和接口类型的相关信息）。

因此，我们可以大胆地做两个推测：

-   所有的 interface 实现都有一个对应的 itab，如果 3 个 struct 分别实现了 5 个 interface，那么就会有 15 个这样的 itab rodata。
-   每个变量在底层都有一个对应的 iface，iface 包含了该变量所需的类型信息和函数表信息，底层是通过把变量组装为 iface，然后以此作为“介质”进行更高级的类型推断或类型转化的实现的。类型转换无非就是从这个样的 iface 转换为另外一个 iface 而已，类型推断无非就是对 itab 做一些相应的判断。

## runtime 代码分析

在那个汇编文件里，我们会看到很多 runtime 函数，这些 runtime 函数在类型推断和转化的过程中扮演了关键角色。

-   runtime.typeAssert
-   runtime.interfaceSwitch

我们直接来看函数的实现：

当进行 type assert 时，会先把把 abi.TypeAssert 和 \_type 这俩给构造出来，然后把它们的指针放到寄存器里，然后再调用 runtime.typeAssert 函数。

```go

type TypeAssert struct {
	Cache   *TypeAssertCache
	Inter   *InterfaceType
	CanFail bool
}
type TypeAssertCache struct {
	Mask    uintptr
	Entries [1]TypeAssertCacheEntry
}
type TypeAssertCacheEntry struct {
	// type of source value (a *runtime._type)
	Typ uintptr
	// itab to use for result (a *runtime.itab)
	// nil if CanFail is set and conversion would fail.
	Itab uintptr
}

type InterfaceType struct {
	Type
	PkgPath Name      // import path
	Methods []Imethod // sorted by hash
}

type _type = abi.Type  // 就是上面的 Type 类型

// typeAssert builds an itab for the concrete type t and the
// interface type s.Inter. If the conversion is not possible it
// panics if s.CanFail is false and returns nil if s.CanFail is true.
func typeAssert(s *abi.TypeAssert, t *_type) *itab {
	var tab *itab
	if t == nil {
		if !s.CanFail {
			panic(&TypeAssertionError{nil, nil, &s.Inter.Type, ""})
		}
	} else {
		tab = getitab(s.Inter, t, s.CanFail)
	}

	if !abi.UseInterfaceSwitchCache(GOARCH) {
		return tab
	}

	// Maybe update the cache, so the next time the generated code
	// doesn't need to call into the runtime.
	if cheaprand()&1023 != 0 {
		// Only bother updating the cache ~1 in 1000 times.
		return tab
	}
	// Load the current cache.
	oldC := (*abi.TypeAssertCache)(atomic.Loadp(unsafe.Pointer(&s.Cache)))

	if cheaprand()&uint32(oldC.Mask) != 0 {
		// As cache gets larger, choose to update it less often
		// so we can amortize the cost of building a new cache.
		return tab
	}

	// Make a new cache.
	newC := buildTypeAssertCache(oldC, t, tab)

	// Update cache. Use compare-and-swap so if multiple threads
	// are fighting to update the cache, at least one of their
	// updates will stick.
	atomic_casPointer((*unsafe.Pointer)(unsafe.Pointer(&s.Cache)), unsafe.Pointer(oldC), unsafe.Pointer(newC))

	return tab
}

func getitab(inter *interfacetype, typ *_type, canfail bool) *itab {
	if len(inter.Methods) == 0 {
		throw("internal error - misuse of itab")
	}

	// easy case
	if typ.TFlag&abi.TFlagUncommon == 0 {
		if canfail {
			return nil
		}
		name := toRType(&inter.Type).nameOff(inter.Methods[0].Name)
		panic(&TypeAssertionError{nil, typ, &inter.Type, name.Name()})
	}

	var m *itab

	// First, look in the existing table to see if we can find the itab we need.
	// This is by far the most common case, so do it without locks.
	// Use atomic to ensure we see any previous writes done by the thread
	// that updates the itabTable field (with atomic.Storep in itabAdd).
	t := (*itabTableType)(atomic.Loadp(unsafe.Pointer(&itabTable)))
	if m = t.find(inter, typ); m != nil {
		goto finish
	}

	// Not found.  Grab the lock and try again.
	lock(&itabLock)
	if m = itabTable.find(inter, typ); m != nil {
		unlock(&itabLock)
		goto finish
	}

	// Entry doesn't exist yet. Make a new entry & add it.
	m = (*itab)(persistentalloc(unsafe.Sizeof(itab{})+uintptr(len(inter.Methods)-1)*goarch.PtrSize, 0, &memstats.other_sys))
	m.inter = inter
	m._type = typ
	// The hash is used in type switches. However, compiler statically generates itab's
	// for all interface/type pairs used in switches (which are added to itabTable
	// in itabsinit). The dynamically-generated itab's never participate in type switches,
	// and thus the hash is irrelevant.
	// Note: m.hash is _not_ the hash used for the runtime itabTable hash table.
	m.hash = 0
	m.init()
	itabAdd(m)
	unlock(&itabLock)
finish:
	if m.fun[0] != 0 {
		return m
	}
	if canfail {
		return nil
	}
	// this can only happen if the conversion
	// was already done once using the , ok form
	// and we have a cached negative result.
	// The cached result doesn't record which
	// interface function was missing, so initialize
	// the itab again to get the missing function name.
	panic(&TypeAssertionError{concrete: typ, asserted: &inter.Type, missingMethod: m.init()})
}

// init fills in the m.fun array with all the code pointers for
// the m.inter/m._type pair. If the type does not implement the interface,
// it sets m.fun[0] to 0 and returns the name of an interface function that is missing.
// It is ok to call this multiple times on the same m, even concurrently.
func (m *itab) init() string {
	inter := m.inter
	typ := m._type
	x := typ.Uncommon()

	// both inter and typ have method sorted by name,
	// and interface names are unique,
	// so can iterate over both in lock step;
	// the loop is O(ni+nt) not O(ni*nt).
	ni := len(inter.Methods)
	nt := int(x.Mcount)
	xmhdr := (*[1 << 16]abi.Method)(add(unsafe.Pointer(x), uintptr(x.Moff)))[:nt:nt]
	j := 0
	methods := (*[1 << 16]unsafe.Pointer)(unsafe.Pointer(&m.fun[0]))[:ni:ni]
	var fun0 unsafe.Pointer
imethods:
	for k := 0; k < ni; k++ {
		i := &inter.Methods[k]
		itype := toRType(&inter.Type).typeOff(i.Typ)
		name := toRType(&inter.Type).nameOff(i.Name)
		iname := name.Name()
		ipkg := pkgPath(name)
		if ipkg == "" {
			ipkg = inter.PkgPath.Name()
		}
		for ; j < nt; j++ {
			t := &xmhdr[j]
			rtyp := toRType(typ)
			tname := rtyp.nameOff(t.Name)
			if rtyp.typeOff(t.Mtyp) == itype && tname.Name() == iname {
				pkgPath := pkgPath(tname)
				if pkgPath == "" {
					pkgPath = rtyp.nameOff(x.PkgPath).Name()
				}
				if tname.IsExported() || pkgPath == ipkg {
					ifn := rtyp.textOff(t.Ifn)
					if k == 0 {
						fun0 = ifn // we'll set m.fun[0] at the end
					} else {
						methods[k] = ifn
					}
					continue imethods
				}
			}
		}
		// didn't find method
		m.fun[0] = 0
		return iname
	}
	m.fun[0] = uintptr(fun0)
	return ""
}

// interfaceSwitch compares t against the list of cases in s.
// If t matches case i, interfaceSwitch returns the case index i and
// an itab for the pair <t, s.Cases[i]>.
// If there is no match, return N,nil, where N is the number
// of cases.
func interfaceSwitch(s *abi.InterfaceSwitch, t *_type) (int, *itab) {
	cases := unsafe.Slice(&s.Cases[0], s.NCases)

	// Results if we don't find a match.
	case_ := len(cases)
	var tab *itab

	// Look through each case in order.
	for i, c := range cases {
		tab = getitab(c, t, true)
		if tab != nil {
			case_ = i
			break
		}
	}

	if !abi.UseInterfaceSwitchCache(GOARCH) {
		return case_, tab
	}

	// Maybe update the cache, so the next time the generated code
	// doesn't need to call into the runtime.
	if cheaprand()&1023 != 0 {
		// Only bother updating the cache ~1 in 1000 times.
		// This ensures we don't waste memory on switches, or
		// switch arguments, that only happen a few times.
		return case_, tab
	}
	// Load the current cache.
	oldC := (*abi.InterfaceSwitchCache)(atomic.Loadp(unsafe.Pointer(&s.Cache)))

	if cheaprand()&uint32(oldC.Mask) != 0 {
		// As cache gets larger, choose to update it less often
		// so we can amortize the cost of building a new cache
		// (that cost is linear in oldc.Mask).
		return case_, tab
	}

	// Make a new cache.
	newC := buildInterfaceSwitchCache(oldC, t, case_, tab)

	// Update cache. Use compare-and-swap so if multiple threads
	// are fighting to update the cache, at least one of their
	// updates will stick.
	atomic_casPointer((*unsafe.Pointer)(unsafe.Pointer(&s.Cache)), unsafe.Pointer(oldC), unsafe.Pointer(newC))

	return case_, tab
}
```

源代码虽然有点长，但是稍微耐心点是很容看懂的。

最关键的是要理解 `func getitab(inter *interfacetype, typ *_type, canfail bool) *itab` 这个函数在做什么。

它尝试从 interface type 和 struct type 里构造出一个 itab，如果能构造成功，那么就意味着这个 struct 实现了该 interface。

而且你可以看到，它还用了缓存机制，第一次构造成功之后就会缓存起来，后续再进行这样的构造时就直接从缓存里拿了，空间换时间，提高性能。

最终核心的函数是`func (m *itab) init() string`。它会尝试从可执行文件里寻找所有 interface/struct pair（主要是 rodata 和 text 段的数据），也就是看这个 struct 是否都实现了 interface 所规定的所有函数。

type switch 呢？一样的，最终都会落到 `func getitab(inter *interfacetype, typ *_type, canfail bool) *itab` 这个函数上。

自此，我们基本上理解了 interface 在 Go 中是如何实现的了，当然其中还有很多细节，但是对于我们理解整体的实现原理并无影响。

<!-- ## 变量实现和指针实现接口有什么不同？

稍微要涉及一点汇编代码的细节了。

TODO -->

## 总结一下

- 查看汇编代码可以很直观地理解 go 语言运行的过程。
-   程序的底层是以 iface/eface/interfacetype/\_type 等各种 abi 数据结构为“介质”来进行交互的。如果说调用一个函数的参数是某种 interface 类型，那么从汇编程序中就可以看出，程序现在堆栈上构造出 iface/eface，然后再调用函数，之后就可以实现 interface 的各种功能了。
-   go 会为每一种 `type x interface` 记录一条 `itab`，这个非常重要。比如说有三种 interface，而且分别有三个具体类型都实现这三个接口，那么汇编代码里就有 9 个 itab 的 rodata。有了这些，就可以实现 interface 那些神奇的功能了。
-   很多 interface 最终都是调用了 go 写的函数而不是汇编级代码，比如说类型 switch 就对应了 runtime.interfaceSwitch，类型推断对应了 runtime.typeAssert，等等。
-   编译器和 runtime 之间需要有一个规约（abi），才能相互无缝地相互配合，让程序运行起来。
-   真正理解了 itab iface eface 等数据结构，那么 interface 的各种特性的实现则是不言自明的。

## 反射的实现

聊完 interface 的实现后再来聊 reflection 就顺理成章了。

具体实现是怎么样的呢？

interface 实现源码中的  eface 和  iface  会和反射实现源码中的 emptyInterface 和 nonEmptyInterface 是一样的数据结构，它们保持同步。

反射中提供了两个核心的数据结构，Type 和 Value，在此基础上实现了各种方法，这两个都叫做反射对象。

Type 和 Value 提供了非常多的方法：例如获取对象的属性列表、获取和修改某个属性的值、对象所属结构体的名字、对象的底层类型(underlying type)等等。

Go 中的反射，在使用中最核心的就两个函数：

-   `reflect.TypeOf(x)`
-   `reflect.ValueOf(x)`

这两个函数可以分别将给定的数据对象转化为以上的 Type 和 Value，这两个都叫做反射对象。

这两个函数的实现源码如下：

```go
// TypeOf returns the reflection Type that represents the dynamic type of i.
// If i is a nil interface value, TypeOf returns nil.
func TypeOf(i interface{}) Type {
    eface := *(*emptyInterface)(unsafe.Pointer(&i))
    return toType(eface.typ)
}

// toType converts from a *rtype to a Type that can be returned
// to the client of package reflect. In gc, the only concern is that
// a nil *rtype must be replaced by a nil Type, but in gccgo this
// function takes care of ensuring that multiple *rtype for the same
// type are coalesced into a single Type.
func toType(t *abi.Type) Type {
	if t == nil {
		return nil
	}
	return toRType(t)
}

func toRType(t *abi.Type) *rtype {
	return (*rtype)(unsafe.Pointer(t))
}
```

```go
// ValueOf returns a new Value initialized to the concrete value
// stored in the interface i. ValueOf(nil) returns the zero Value.
func ValueOf(i any) Value {
	if i == nil {
		return Value{}
	}
	return unpackEface(i)
}

// unpackEface converts the empty interface i to a Value.
func unpackEface(i any) Value {
	e := (*emptyInterface)(unsafe.Pointer(&i))
	// NOTE: don't read e.word until we know whether it is really a pointer or not.
	t := e.typ
	if t == nil {
		return Value{}
	}
	f := flag(t.Kind())
	if ifaceIndir(t) {
		f |= flagIndir
	}
	return Value{t, e.word, f}
}
```

实际上也是一个 pack 和 unpack 出 emptyInterface/nonEmptyInterface 的过程。

---

在 Go 中，反射是在类型系统的基础上包装了一套更高级的类型系统的用法。上面说那些类型推断、类型转换只不过是这套类型系统的一个应用而已，而且这个应用直接集成到了代码语法上。

反射无非就是把 itab 这样的静态数据的构造从编译阶段放到了运行阶段。或者从另外一个角度来说，就是把静态类型检查从编译阶段放到了运行阶段。

Go 中反射机制的本质是，Go 会把函数和类型的元数据（尤其是 itab，比如：go.itab.\*"".MyStruct,"".MyInterface SRODATA dupok size=48）存储在 rodata 里，在运行时，通过读取这些元数据，来动态构造出 iface，然后在此基础上进行一些数据修改或函数的调用。

## 反射三定律

根据 Go 官方关于反射的博客，反射有三大定律：

-   Reflection goes from interface value to reflection object.
-   Reflection goes from reflection object to interface value.
-   To modify a reflection object, the value must be settable.

关于第三条，记住一句话：如果想要操作原变量，反射变量 Value 必须要 hold 住原变量的地址才行。

## 反射的优劣

使用反射的优势：

-   程序抽象性提升
-   程序表达力提升

适用反射的坏处：

-   程序维护性降低
-   程序性能降低
-   程序安全性降低

## 参考资料

-   [go internals Chapter II: Interfaces](https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/README.md)
-   [Go Data Structures: Interfaces](https://research.swtch.com/interfaces)
-   [A Quick Guide to Go's Assembler](https://go.dev/doc/asm)
-   [Introduction to the Go compiler](https://go.dev/src/cmd/compile/README)
-   [Go 语言设计与实现 4.2 接口](https://draveness.me/Go/docs/part2-foundation/ch04-basic/Go-interface)
<!-- -   [Finding unreachable functions with deadcode](https://go.dev/blog/deadcode) -->
