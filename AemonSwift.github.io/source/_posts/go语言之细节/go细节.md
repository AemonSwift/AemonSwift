---
title: go细节
date: 2019-11-13 08:56:45
tags:
---
# go语法实现细节

1. for-range

- 当使用for-range遍历一个容器时，其实遍历的是此容器的一个副本。
- 循环变量是被遍历的容器的每个键值（或索引）和元素对的副本。
- 在循环过程中,循坏变量的地址保持不变（并发时需要注意）
```go
package main
import "fmt"
type T struct {
  n int
}
func main() {
  ts := [2]T{}
  for i, t := range ts {
    switch i {
    case 0:
      t.n = 3
      ts[1].n = 9
    case 1:
      fmt.Print(t.n, " ")
    }
  }
  fmt.Print(ts)
}
//结果为0 [{0} {9}]，由于for-range时对数组进行了拷贝（值拷贝）

package main
import "fmt"
type T struct {
  n int
}
func main() {
  ts := [2]T{}
  for i, t := range &ts {
    switch i {
    case 0:
      t.n = 3
      ts[1].n = 9
    case 1:
      fmt.Print(t.n, " ")
    }
  }
  fmt.Print(ts)
}
//结果为9 [{0} {9}]，由于for-range时对数组进行了指针拷贝，而t是ts中的元素的副本，t.n的形式改变不会影响原来结果

package main
import "fmt"
func main() {
  ts := []int{1,2,3,4,5,6,7}
  for i, t := range ts {
      go func() {
          fmt.Println(t)
      }
  }
}
// 结果有可能为7,7,7,7,7,7,7

package main
import "fmt"
func main() {
  ts := []int{1,2,3,4,5,6,7}
  for i, t := range ts {
      tmp:=t
      go func() {
          fmt.Println(tmp)
      }
  }
}

//结果为1,2,3,4,5,6,7
```
2. cap只能对数组，slice，channel有效，而对map使用时错误的。
3. 声明变量必须要指定类型，否则会报错
```go
func main() {  
    var x = nil 
//nil 用于表示 interface、函数、maps、slices 和 channels 的“零值”。如果不指定变量的类型，编译器猜不出变量的具体类型，导致编译错误。
    _ = x
}
```
4. 不能使用短变量声明设置结构体字段值
```go
type info struct {
    result int
}

func work() (int,error) {
    return 13,nil
}

func main() {
    var data info

    data.result, err := work() //--->更改
    //*****更改后
    var err error
    data.result, err = work()
    //******
    fmt.Printf("info: %+v\n",data)
}
```
4. 值接收者和指针接收者的区别
- 不管方法的接收者是什么类型，该类型的值和指针都可以调用，不必严格符合接收者的类型。
```go
package main

import "fmt"

type Person struct {
	age int
}

func (p Person) howOld() int {
	return p.age
}

func (p *Person) growUp() {
	p.age += 1
}

func main() {
	// qcrao 是值类型
	qcrao := Person{age: 18}

	// 值类型 调用接收者也是值类型的方法
	fmt.Println(qcrao.howOld())

	// 值类型 调用接收者是指针类型的方法
	qcrao.growUp()
	fmt.Println(qcrao.howOld())

	// ----------------------

	// stefno 是指针类型
	stefno := &Person{age: 100}

	// 指针类型 调用接收者是值类型的方法
	fmt.Println(stefno.howOld())

	// 指针类型 调用接收者也是指针类型的方法
	stefno.growUp()
	fmt.Println(stefno.howOld())
}

// 结果为：
18
19
100
101
```
实际上，当类型和方法的接收者类型不同时，其实是编译器在背后做了一些工作
*||
:-:|:-:|:-:
-|值接收者|指针接收者
值类型调用者| 方法会使用调用者的一个副本，类似于“传值”| 使用值的引用来调用方法，上例中，qcrao.growUp() 实际上是 (&qcrao).growUp()
指针类型调用者| 指针被解引用为值，上例中，stefno.howOld() 实际上是 (*stefno).howOld()| 实际上也是“传值”，方法里的操作会影响到调用者，类似于指针传参，拷贝了一份指针
- 实现了接收者是值类型的方法，相当于自动实现了接收者是指针类型的方法；而实现了接收者是指针类型的方法，不会自动生成对应接收者是值类型的方法。
```go
package main
import "fmt"
type coder interface {
    code()
    debug()
}
type Gopher struct {
    language string
}
func (p Gopher) code() {
    fmt.Printf("I am coding %s language\n", p.language)
}
func (p *Gopher) debug() {
    fmt.Printf("I am debuging %s language\n", p.language)
}
func main() {
    var c coder = &Gopher{"Go"}
    c.code()
    c.debug()
}
// 运行结果为:
I am coding Go language
I am debuging Go language


func main() {
    var c coder = Gopher{"Go"}
    c.code()
    c.debug()
}
// 运行报错
Gopher 没有实现 coder。很明显了吧，因为 Gopher 类型并没有实现 debug 方法；表面上看， *Gopher 类型也没有实现 code 方法，但是因为 Gopher 类型实现了 code 方法，所以让 *Gopher 类型自动拥有了 code 方法。
```
当然，上面的说法有一个简单的解释：接收者是指针类型的方法，很可能在方法中会对接收者的属性进行更改操作，从而影响接收者；而对于接收者是值类型的方法，在方法中不会对接收者本身产生影响。
所以，当实现了一个接收者是值类型的方法，就可以自动生成一个接收者是对应指针类型的方法，因为两者都不会影响接收者。但是，当实现了一个接收者是指针类型的方法，如果此时自动生成一个接收者是值类型的方法，原本期望对接收者的改变（通过指针实现），现在无法实现，因为值类型会产生一个拷贝，不会真正影响调用者。