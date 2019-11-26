---
title: auto与decltype关键字使用
date: 2019-11-13 08:56:45
tags:
---

# auto简介
我们知道C语言中其实也有auto关键字，它和早期C++中的auto关键字一样，它修饰局部变量，表示自动存储期，声明变量的时候可以省略掉此关键字。而本部分是对C++11引入关键字更进一步深入。
```c++
#include<iostream>
#include<vector>
int main()
{
    std::vector<int> vec{1,2,3,4,5};
    for(std::vector<int>::const_iterator it = vec.begin();it != vec.end();++it)
    { // 对于it的类型，你自己能快速写出来吗？我反正是写不出来。
        std::cout<<*it<<std::endl;
    }
    return 0;
}

// auto版本 
#include<iostream>
#include<vector>
int main()
{
    std::vector<int> vec{1,2,3,4,5};
    for(auto it = vec.begin();it != vec.end();++it)
    {
        std::cout<<*it<<std::endl;
    }
    return 0;
}
```
## auto的作用
通过变量的初值，来分析变量的类型。（初值都没，则怎能谈分析类型）

## auto用法
### 普通类型推导
```C++
auto i = 10;//i为int类型
auto d = 10.2//d 为double类型
auto f = 10.2f//f为float类型
```
### const关键字修饰的类型推导
通常auto会忽略掉顶层const（本身是常量，如int \*cosnt p），而会保留底层const（指向的对象是常量，如const int\* p）

```C++
const int ci = 10;
auto aci = ci;//忽略顶层const，推导ci是int，所以aci类型是int
const auto ca = ci//推导ci是int，但是前面有const，所以ca是const int

const int arr[] = {11};
auto p = arr;//arr 是const int *,这是底层const，推导后，保留底层const，所以p是 const int*
```
arr数组名被当成指针是，是const int*类型，或者说是int const*，它指向的对象是只读的，因此是底层const，保留，最终p的类型也是int const *。

```c++
onst int ci = 10;
auto &cp = ci;//cp是一个整型常量引用 const int &

如果是字面值，则必须加上const：
const auto &ref = 10;//10是字面值，常量引用才能绑定字面值

std::vector<int> vec;
auto size = vec.size(); //它是std::vector::size_type。
```
注意auto无法推导模板类型。
```c++
  vector<string> aa;
//vector<string> bb = aa;//无法推导出模板类型
```
下面这段代码帮你看到变量真正的名称。
```c++
#include <iostream>
#include <vector>
#include <cxxabi.h>
#include <typeinfo>
int main()
{
    int     status;
    char   *realname;
    auto type = 1.1;
    realname = abi::__cxa_demangle(typeid(type).name(), 0, 0, &status);
    std::cout << typeid(type).name() << " => " << realname <<std::endl;
    free(realname);
    return 0;
}
//输出结果为double
```

# decltype简介
我们之前使用的typeid运算符来查询一个变量的类型，$\color{red}{这种类型查询在运行时进行。}$RTTI机制为每一个类型产生一个type_info类型的数据，而typeid查询返回的变量相应type_info数据，通过name成员函数返回类型的名称。同时在C++11中typeid还提供了hash_code这个成员函数，用于返回类型的唯一哈希值。$\color{red}{RTTI会导致运行时效率降低，且在泛型编程中，我们更需要的是编译时就要确定类型，RTTI并无法满足这样的要求。编译时类型推导的出现正是为了泛型编程，在非泛型编程中，我们的类型都是确定的，根本不需要再进行推导。}$
而编译时类型推导，除了我们说过的auto关键字，还有decltype。
decltype与auto关键字一样，用于进行编译时类型推导，不过它与auto还是有一些区别的。
decltype的类型推导并不是像auto一样是从变量声明的初始化表达式获得变量的类型，$\color{red}{而是总是以一个普通表达式作为参数，返回该表达式的类型,而且decltype并不会对表达式进行求值。}$

## 常见用途

### 推导出表达式类型

```c++
int i=4;
decltype(i) a; //推导结果为int。a的类型为int。
```
与using/typedef合用，用于定义类型。
```c++
using size_t = decltype(sizeof(0));//sizeof(a)的返回值定义size_t类型
using ptrdiff_t = decltype((int*)0 - (int*)0);
using nullptr_t = decltype(nullptr);

vector<int >vec;
typedef decltype(vec.begin()) vectype; //定义vectype类型
for (vectype i = vec.begin; i != vec.end(); i++)
{
    //...
}
```

### 重用匿名类型（本质仍然为推导出表达式类型）
```C++
struct 
{
    int d ;
    doubel b;
}anon_s; //匿名对象

decltype(anon_s) as ;//定义了一个上面匿名的结构体
```

### 泛型编程中结合auto，用于追踪函数的返回值类型（最大用途）
```c++
template <typename _Tx, typename _Ty>
auto multiply(_Tx x, _Ty y)->decltype(_Tx*_Ty)
{
    return x*y;
}
```

## decltype推导规则
1. 如果e是一个没有带括号的标记符表达式或者类成员访问表达式，那么的decltype（e）就是e所命名的实体的类型。此外，如果e是一个被重载的函数，则会导致编译错误。
2. 否则 ，假设e的类型是T，如果e是一个将亡值，那么decltype（e）为T&&
3. 否则，假设e的类型是T，如果e是一个左值，那么decltype（e）为T&。
4. 否则，假设e的类型是T，则decltype（e）为T。

标记符指的是除去关键字、字面量等编译器需要使用的标记之外的程序员自己定义的标记，而单个标记符对应的表达式即为标记符表达式。例如：
```c++
int arr[4]; //arr为一个标记符表达式，而arr[3]+0不是。

int i=10;
decltype(i) a; //a推导为int
decltype((i))b=i;//b推导为int&，必须为其初始化，否则编译错误
```
仅仅为i加上了()，就导致类型推导结果的差异。这是因为，i是一个标记符表达式，根据推导规则1，类型被推导为int。而(i)为一个左值表达式，所以类型被推导为int&。
规则具体含义：
```c++
int i = 4;
int arr[5] = { 0 };
int *ptr = arr;
struct S{ double d; }s ;
void Overloaded(int);
void Overloaded(char);//重载的函数
int && RvalRef();
const bool Func(int);

//规则一：推导为其类型
decltype (arr) var1; //int 标记符表达式

decltype (ptr) var2;//int *  标记符表达式

decltype(s.d) var3;//doubel 成员访问表达式

//decltype(Overloaded) var4;//重载函数。编译错误。

//规则二：将亡值。推导为类型的右值引用。

decltype (RvalRef()) var5 = 1;

//规则三：左值，推导为类型的引用。

decltype ((i))var6 = i;     //int&

decltype (true ? i : i) var7 = i; //int&  条件表达式返回左值。

decltype (++i) var8 = i; //int&  ++i返回i的左值。

decltype(arr[5]) var9 = i;//int&. []操作返回左值

decltype(*ptr)var10 = i;//int& *操作返回左值

decltype("hello")var11 = "hello"; //const char(&)[9]  字符串字面常量为左值，且为const左值。


//规则四：以上都不是，则推导为本类型

decltype(1) var12;//const int

decltype(Func(1)) var13=true;//const bool

decltype(i++) var14 = i;//int i++返回右值
```
这里需要提示的是，字符串字面值常量是个左值，且是const左值，而非字符串字面值常量则是个右值。
规则太难记忆，我们可以使用C++11标准库中添加的模板类is_lvalue_reference,is_rvalue_reference来判断表达式是否为左值：

```c++
 cout << is_lvalue_reference<decltype(++i)>::value << endl; //结果1表示为左值，结果为0为非左值。
 cout << is_rvalue_reference<decltype(++i)>::value << endl; //结果1表示为右值，结果为0为非右值。
```