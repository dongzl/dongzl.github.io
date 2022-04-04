---
title: Go 语言编程知识学习总结（持续更新中）
date: 2020-06-08 11:08:31
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/go_study.png

# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文旨在汇总学习 Go 语言知识相关内容，总结 Go 语言相关知识和一些语法糖的使用。

categories: 
  - Go语言

tags: 
  - Go
---

## 背景

2020年的一个 `Flag`，学习 `Go` 语言的使用，虽然说编程语言都是相通的，已经能够熟练使用 Java 语言进行程序开发了，但是这并不能决定对于学习 Go 语言具备什么样的巨大优势，有些时候学习一门编程语言的语法和基本使用，写出一个 `Hello World!` 的小程序并不会很难，但是如果说要使用一门新的语言做出一个小工具、小产品，并且能够使用优雅的 `code` 完成这个工作，并不是一件容易的事情。本文汇总自己 `Go` 语言学习过程中的一些知识内容，不会一蹴而就，会持续不断的更新。

## 知识整理

### Go 语言版本 Hello World!

```go
package main

import "fmt"

func main()  {
	fmt.Println("Hello, World!")
}
```

- 每个源文件都以一条 `package` 声明语句开始；
- `main` 包笔记特殊。它定义了一个独立的可执行程序，而不是一个库；
- 缺少了必要的包或者导入了不需要的包，程序都无法编译通过；
- `import` 声明必须跟在文件的 `package` 声明之后；
- 函数（func）、变量（var）、常量（const）、类型（type）；
- `Go` 语言不需要在语句或者声明的末尾添加分号，除非一行上有多条语句；
- `Go` 语言在代码格式上采取了很强硬的态度，以法令方式规定标准的代码格式可以避免无尽的无意义的琐碎争执（译注：也导致了Go语言的TIOBE排名较低，因为缺少撕逼的话题）；
- 在 `Go` 语言中，`j = i++`、`--i ++i` 都是非法的。

### Go main package

> Package main is special. It defines a standalone executable program, not a library. Within package main the function main is also special—it’s where execution of the program begins. Whatever main does is what the program does.

`main package` 不同于其它 `library package`，它定义了一个可执行程序。其中的 `main` 函数即是可执行文件的入口函数。

### Go import

- 普通导入；
- 别名导入，如果两个包的包名存在冲突，或者包名太长需要简写时，我们可以使用别名导入来解决。
- `.`导入，点导入可以让包内的方法注册到当前包的上下文中，直接调用方法名即可，不需要再加包前缀。
- `_`导入，下划线导入是包引用操作，只会执行包下各模块中的 `init` 方法，并不会真正的导入包，所以不可以调用包内的其他方法。

[golang 之 import 和 package 的使用](https://studygolang.com/articles/18395?fr=sidebar)

### for 循环语句

```go
for initialization; condition; post {

}

for condition {

}

for {

}

for _, arg := range os.Args[1:] {

}
```

- for 语句两边不需要加括号；
- initialization 语句可选，在循环开始之前执行，必须是一条简单语句；
- for  循环的另一种形式, 在某种数据类型的区间（range）上遍历，如字符串或切片；每次循环迭代，range 产生一对值；索引以及在该索引处的元素值；
- `_` 代表空标识符，空标识符可用于任何语法需要变量名但程序逻辑不需要的时候, Go 语言不允许使用无用的局部变量（local variables），因为这会导致编译错误，所以出现了空标识符；

```go
f, err := os.Open(arg)
if err != nil {
	fmt.Println(os.Stderr, "dup2: %v\n", err)
	continue
}
countLines(f, counts)
f.Close()
```

`os.Open` 函数返回两个值。第一个值是被打开的文件 `(*os.File)`，其后被 `Scanner` 读取。`os.Open` 返回的第二个值是内置 `error` 类型的值。如果 `err` 等于内置值 `nil`（译注：相当于其它语言里的 `NULL`），那么文件被成功打开。读取文件，直到文件结束，然后调用 `Close` 关闭该文件，并释放占用的所有资源。相反的话，如果 `err` 的值不是 `nil`，说明打开文件时出错了。

### 访问权限

如果一个名字是在函数内部定义，那么它就只在函数内部有效。如果是在函数外部定义，那么将在当前包的所有文件中都可以访问。名字的开头字母的大小写决定了名字在包外的可见性。如果一个名字是大写字母开头的（译注：必须是在函数外部定义的包级名字；包级函数名本身也是包级名字），那么它将是导出的，也就是说可以被外部的包访问，例如 `fmt` 包的 `Printf` 函数就是导出的，可以在 `fmt` 包外部访问。包本身的名字一般总是用小写字母。

**在包一级声明语句声明的名字可在整个包对应的每个源文件中访问，而不是仅仅在其声明语句所在的源文件中访问。**

### 函数定义

```go
func fToC(f float64) float64 {
	return (f - 32) * 5 / 9
}
```

- 函数名
- 参数列表
- 返回值列表
- 函数体

### 变量

```go
var 变量名字 类型 = 表达式

var i, j, k int

var b, f, s = true, 2.3, "four"

var f, err = os.Open(name)

## 简短变量声明（局部变量的声明和初始化）
名字 := 表达式
```

- `var` 形式的声明语句往往是用于需要显式指定变量类型的地方，或者因为变量稍后会被重新赋值而初始值无关紧要的地方；

- 简短变量声明左边的变量可能并不是全部都是刚刚声明的。如果有一些已经在相同的词法域声明过了，那么简短变量声明语句对这些已经声明过的变量就只有赋值行为了；

```go
in, err := os.Open(infile) // 声明 in、err
out, err := os.Create(outfile) // out 声明、err 赋值
```

- 简短变量声明语句中必须至少要声明一个新的变量。

```go
f, err := os.Open(infile)
f, err := os.Create(outfile)  //compile error: no new variables
```

### 基础数据类型

`Go` 语言数据类型分类：

- 基础类型（数字、字符串、布尔型）
- 复合类型（数组、结构体）
- 引用类型（指针、切片、字典、函数、通道）
- 接口类型

`&^` 用于按位置零（AND NOT）：如果对应y中bit位为1的话, 表达式  z = x &^ y  结果z的对应的bit位为0，否则z对应的bit位等于x相应的bit位的值。

### 复合数据类型

#### 数组

**在数组字面值中，如果在数组的长度位置出现的是“...”省略号，则表示数组的长度是根据初始化值的个数来计算。**

```go
var  a [ 3 ] int // array of 3 integers

var  q [ 3 ] int  = [ 3 ] int { 1 ,  2 ,  3 }

var  r [ 3 ] int  = [ 3 ] int { 1 ,  2 }

q := [...] int { 1 ,  2 ,  3 }
```

如果一个数组的元素类型是可以相互比较的，那么数组类型也是可以相互比较的，这个时候我们可以直接通过 `==` 比较运算符来比较两个数组，只有当两个数组的所有元素都是相等的时候数组才是相等的。

#### Slice

`Slice（切片）`代表变长的序列，序列中每个元素都有相同的类型。一个slice类型一般写作[]T，其中T代表slice中元素的类型；slice的语法和数组很像，只是没有固定长度而已。

`Slice` 的切片操作 `s[i:j]`，其中 `0 ≤ i≤ j≤ cap(s)`，用于创建一个新的 `Slice`，引用 `s` 的从第 `i` 个元素开始到第 `j-1` 个元素的子序列。新的 `Slice` 将只有 `j-i` 个元素。如果i位置的索引被省略的话将使用 `0` 代替，如果 `j` 位置的索引被省略的话将使用 `len(s)` 代替。

```go
months := [...] string {1:"January", /* ... */, 12:"December" }
```

#### Map

**在 Go 语言中，一个 map 就是一个哈希表的引用，map 类型可以写为 map[K]V，其中 K 和 V 分别对应 key 和 value。**
```go
ages :=  make ( map [ string ] int )

ages :=  map [ string ] int {
  "alice" :    31 ,
  "charlie" :  34 ,
}

ages :=  make ( map [ string ] int )
ages[ "alice" ] =  31 
ages[ "charlie" ] =  34
```

#### 结构体

**结构体是一种聚合的数据类型，是由零个或多个任意类型的值聚合成的实体。每个值称为结构体的成员。**

```go
type Employee struct  {
  ID         int
  Name       string
  Address    string
  DoB       time.Time
  Position   string
  Salary     int 
  ManagerID  int 
}

var  dilbert Employee
```

#### JSON

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
)

type Movie struct {
	Title    string
	Year     int
	Color    bool
	Actors   []string
}

var movies = []Movie {
	{Title: "Casablanca", Year: 1942, Color: false,
		Actors: []string{"Humphrey Bogart", "Ingrid Bergman"}},
	{Title: "Cool Hand Luke", Year: 1967, Color: true,
		Actors: []string{"Paul Newman"}},
	{Title: "Bullitt", Year: 1968, Color: true,
		Actors: []string{"Steve McQueen", "Jacqueline Bisset"}},
}

func main()  {
	//data, err := json.Marshal(movies)
	data, err := json.MarshalIndent(movies, "", "    ")
	if err != nil {
		log.Fatalf("JSON marshaling failed: %s", err)
	}
	fmt.Printf("%s\n", data)
}
```

#### 文本和 HTML 模板

### 函数

函数声明包括函数名、形式参数列表、返回值列表（可省略）以及函数体。

```go
func name(parameter-list) (result-list) {
    body
}

func hypot(x, y float64 ) float64 {
    return  math.Sqrt(x*x + y*y)
}
fmt.Println(hypot(3, 4))  // "5"
```

在 Go 中，一个函数可以返回多个值。我们已经在之前例子中看到，许多标准库中的函数返回 2 个值，一个是期望得到的返回值，另一个是函数出错时的错误信息。

调用多返回值函数时，返回给调用者的是一组值，调用者必须显式的将这些值分配给变量:

```go
links, err := findLinks(url)
```

如果某个值不被使用，可以将其分配给blank identifier:

```go
links, _ := findLinks(url)  // errors ignored
```
