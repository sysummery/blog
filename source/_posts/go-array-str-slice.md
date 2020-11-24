---
title: Go语言中的数组、字符串、切片
date: 2019-12-23 12:18:35
tags:
    - go
photos :
    - ["http://sysummery.top/gocover1.jpeg"]
---
之所以把他们三个放一起是因为他们3个的底层的原始数据有着一样的数据结构。
<!--more-->
## 数组
数组是由**固定长度的**特定类型的元素组成的序列。这些元素在内存中的存储是连续的。数组一经定义，他的长度是不能改变的，数组的长度可以是0.

下面几种数组的定义方式都是正确的。在go语言中，数组就是数组，并不是指向首元素的指针。
```go

//长度为3的数组，数组的每个元素的值都是0
var test [3]int

//与上面的一样，加了{}就得用等于号
var test = [3]int{}

//长度为3的数组，在定义的时候也初始化了每个元素的值
var test = [3]int{1,2,3}

//长度为3的数组，第一个元素的值为1其余的值为0
var test = [3]int{1}

//长度为3的数组，下标为1的元素的值是2，下标为0的元素的值是3，其余的值是0
var test = [3]int{1:2,0:3}

//长度为3的数组，下标为0的元素值是1，下标为1的元素值是2，下标为2的元素值是3
var test = [3]int{1,1:2,2:3}
```

当一个数组被赋值或者当做函数参数被传递的时候，会复制整个数组，如果此时数组比价大，那么开销也是很大的。可以使用数组指针替代。数组指针在使用上和数组是很相似的
```go
func main () {
	var test = [3]int{1:2,0:3}

	p := &test

	fmt.Println(test[0])
	fmt.Println(p[0])

	test[0] = 100

	fmt.Println(test[0])
	fmt.Println(p[0])
}
```
结果
```
3
3
100
100
```

空数组时不占内存的，这一点与空结构体是一样的。
```go

var times [5][0]int

for range times {
    fmt.Println("hello")
}
```

## 字符串
字符串是一个连续的不可改变的字节序列。
看一下go语言中字符串的定义其实是个结构体
```go
type StringHeader struct {
    Data uintptr
    Len int
}
```
可以看出，字符串的底层是一个非负数的字节数组，另外还有一个长度。
所以当复制一个字符串给一个变量或者作为参数传递的时候只是复制了指向基础数据的指针和字符串的长度，并没有复制底层的数据。

下面先看一段代码
```go
func main () {
	str := "Hello 世界"
	for i:= 0; i < len(str); i++ {
		ch := str[i]
		fmt.Println(ch)
	}

	fmt.Println("-------unicode遍历-------")
	for _, ch := range str {
		fmt.Println(ch)
	}
}
```
结果为
72
101
108
108
111
32
228
184
150
231
149
140
-----------unicode遍历-------------
72
101
108
108
111
32
19990
30028

从结果我们可以看出

1. 两个循环输出的都是一堆数字，说明go中的字符串是以byte方式存储的
2. for循环输出的是字节的“数值”，for range输出的是以utf-8为编码方式的unicode字符的“数值”

如果我想直接输出字符而不是字符的“数值”怎么办？很简单，用强制转换就好

```go
func main () {
	str := "Hello 世界"
	for i:= 0; i < len(str); i++ {
		ch := str[i]
		fmt.Println(string(ch))
	}

	fmt.Println("------unicode遍历--------")
	for _, ch := range str {
		fmt.Println(string(ch))
	}
}
```
输出的结果为
H
e
l
l
o
 
ä
¸

ç


-----------unicode遍历-------------
H
e
l
l
o
 
世
界

第一个循环用6个字节“表示”了世界，因为大部分中文字符在utf-8编码方式下使用3个字节。所以在代码中多使用for range来循环字符串

再看一段代码
```go
func main () {
	str := "Hello 世界"
	fmt.Println(string(str[6]))
}
```

结果
ä

这说明了字符串默认是以byte数组存放的而不是rune数组。如果字符串中包含中文字符并且要读取这个字符的时候要先转成rune类型的数组切片
```go
func main () {
	str     := "Hello 世界"
	strRune := []rune(str)
	fmt.Println(string(strRune[6]))
}
```
结果
世


go中的单引号与双引号是不同的,正常的字符串用双引号表示，单引号内只能是一个字符

```go
func main () {
	str     := '我'
	fmt.Println(str)
}
```
结果
25105

str存放的其实是一个rune

```go
func main () {
	str     := '我'
	fmt.Println(string(str))
}
```
结果
我

最后说说go语言中的反引号(\`)。简单地说反引号中的内容不支持任何的转义，所看即所得。

## 切片
与数组不同的是切片的大小可以变化，并且切边多了一个cap属性，意为最大数量。先看一下切片在go语言中的定义
```go
type SliceHeader struct {
    Data uintptr
    Len int
    Cap int
}
```
这也说明了复制一个切片只是复制了指向底层数据的指针以及len和cap，并不会复制底层的数据。

下面说一下切片的声明和初始化
```go
// 不是空切片是nil，这一点与数组是不一样的
var a []int

//空切片
var b = []int{}

//len和cap都为3
var c = []int{1,2,3}

//截取c的第0个和第1个元素生成一个新的切片，len是2，因为从c的0位置开始截取，所以cap与c的cap是一样的3
var d = c[:2]

//截取c的第1个元素生成一个新的切片，len是1，cap是2
var d = c[1:2]

//截取c的第0个元素和第一个元素生成一个新的切片，len是2，cap是3
var e = c[0:2:cap(c)]

//len和cap都是3，每个元素的值是0
var f = make([]int, 3)

//len是2cap是3，每个元素都是0
var g = make([]int, 2, 3)
```

切片的追加。如果追加后超过了原始切片的cap，那么追加后切片的cap是追加前的两倍
```go
func main () {
	var c = []int{1,2,3,4,5,6}

	fmt.Println(len(c))
	fmt.Println(cap(c))

	c = append(c, 7)

	fmt.Println(len(c))
	fmt.Println(cap(c))
}
```
结果
```
6
6
7
12
```
删除元素
```go
func main () {
	a = []int{1, 2, 3}
    // 删除尾部1个元素
    a = a[:len(a)-1]
 
    // 删除尾部N个元素
    a = a[:len(a)-N]
}
```
