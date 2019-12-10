---
date: "2018-09-08" 
title: "[翻译]c++ 11特性走览"
categories: ["C++11"] 
---

## 原文链接

http://soft.vub.ac.be/~cderoove/structuur2/C++11.pdf

## 空指针常量 - nullptr

```c++
void foo(char*);   // 1
void foo(int);     // 2

fool(NULL);        //<c++03> 这个函数会调用函数2
fool(nullptr);     //<c++11> 这个函数会调用函数1
```

## 标准类型

在`c++03`中，不同的类型在不同的编译器下面占用的内存空间大小可能不同，如，`short`在32位编译器下占用的大小为16bit，而在64位编译器下面占用的大小为32bit。`c++11`为了解决这个问题，给出了标准类型如下：

```c++
int8_t
uint8_t
int16_t
uint16_t
int32_t
uint32_t
int64_t
uint64_t
```

## raw string literals

raw string中没有转义字符，字符串表示的内容所见即所得。

```c++
string test03 = "C:\\A\\B\\file.txt";
string test11 = R"(C:\A\B\file.txt)";  //<c++11> 不同转义字符

cout << test03 << endl; // output is: C:\A\B\file.txt
cout << test11 << endl; // output is: C:\A\B\file.txt

string testBlock =
R"(First lint.
Second line.
Third Line.)";
cout << testBlock <<endl;
/* The output is:
First line.
Second line.
Third line.
*/
```

## class内部直接进行成员初始化

在03的版本中，c++类成员变量的初始化通常位于类的构造函数中，而在c++11中，可以直接进行初始化，如下：

```c++
// C++11
class A
{
public:
    A() {}
    A(int in_a) ： a(in_a) {}
    A(C c) {}
private:
    int a=4;
    int b=2;
    string h="text1";
    string s="text2";
};

// 上面相同效果的实现在c++03版本中为
class A
{
public:
    A() : a(4), b(2), h("text1"), s("text2") {}
    A(int in_a) : a(in_a), b(2), h("text1"), s("text2") {}
    A(C c) : a(4), b(2), h("text1"), s("text2") {}
private:
    int a;
    int b;
    string h;
    string s;
};
```

## 委派构造函数 Delegating constructor

在c++03版本中，如果需要实现在多个构造函数中做相同的事情，通常需要实现一个独立的函数，分别给不同的构造函数进行调用，而在c++11中则可以让一个构造函数调用另一个构造函数。

```c++
// c++03
class A
{
    int a;
    void validate(int x)
    {
        if (0<x && x<=42) a=x; else throw bad_A(x);
    }
public:
    A(int x) { validate(x); }
    A() { validate(42); }
    A(string s) { int x=stoi(s); validate(x); }
};

// c++11
class A
{
    int a;
public:
    A(int x)
    {
        if (0<x && x<=42) a=x; else throw bad_A(x);
    }
    A() : A(42) { }
    A(string s) : A(stoi(s)) { }
};
```

## override

新增加的关键字，如果派生类在虚函数声明的时候使用了override关键字，那么该函数必须重载其基类中的同名函数，否则代码将无法通过编译。

```c++
// c++03
struct Base
{
    virtual void some_func(float);
};

struct Derived : Base
{
    virtual void some_func(int);
    // 这个会产生警告
};

// c++11
struct Derived : Base
{
    void some_func(int) override;
    // 这个会产生错误
};
```

## final

新引入的关键字，指定派生类不能被继承，或不能覆写虚函数。

```c++
// c++11
struct Base1 final {};
struct Derived1 : Base1 {}; // 这个会引起错误，派生类不能被继承

struct Base2 {
    virtual void f() final;
};

struct Derived2 : Base2 {
    void f(); // error: 虚函数不能被覆写
};
```

## static_assert

这个关键字是用于编译期间的断言，因此叫做静态断言。

```c++
// c++11
template<class T>
void f(T v) {
    static_cast(sizeof(v) == 4, "v must have size of 4 bytes");
}

void g() {
    int64_t v;
    f(v); // 会产生一个编译错误，在vs2010/2012下：error C2338：v must have size of 4 bytes
}
```

## type traits

type traits被用来判断类型的特征。

```c++
// c++11
#include <type_traits>
#include <iostream>
using namespace std;

struct A{};
struct B{ virtual void f(){} };
struct C: B{};

int main() {
    cout << "int:" << has_virtual_destructor<int>::value << endl; // int:0
    cout << "int:" << is_polymophic<int>::value << endl;          // int:0
    cout << "A:" << is_polymophic<A>::value << endl;          // A:0
    cout << "B:" << is_polymophic<B>::value << endl;          // B:1
    cout << "C:" << is_polymophic<C>::value << endl;          // C:1

    typedef int mytype[][24][60];
    cout << "(0 dim):" << extent<mytype,0>::value << endl;    // (0 dim): 0
    cout << "(1 dim):" << extent<mytype,1>::value << endl;    // (1 dim): 24
    cout << "(2 dim):" << extent<mytype,2>::value << endl;    // (2 dim): 60
    return 0;
}
```

## Auto

c++11引入的auto的作用是自动类型推断和返回值占位。下面一些情况推荐使用auto。

```c++
// c++11
auto p = new T(); // 具体类型已经出现在表达式中
auto p = make_shared<T>(arg1)

auto my_lambda = [](){}
auto it = m.begin(); // 减少代码的复杂性
```

## decltype

从表达式中推断出具体类型。

```c++
int main() {
    int i=4;
    const int j=6;
    const int& k=i;
    int a[5];
    int *p;

    int var1; // c++11: decltype(i) var1;
    int var2; // c++11: decltype(1) var2;
    int var3; // c++11: decltype(2+3) var3;
    int& var4=i; // c++11: decltype(i=1) var4=i;
    const int var5=1;  // c++11: decltype(j) var5=1;
    const int& var6=j; // c++11: decltype(k) var6=j;
    int var7[5];       // c++11: decltype(a) var7;
    int& var8=i;       // c++11: decltype(a[3]) var8=i;
    int& var9=i;       // c++11: decltype(*p) var9=i;

    return 0;
}
```

## suffix return type syntax

模板函数返回值的类型不好确定的时候，可以使用后缀返回的格式。如下：

```c++
\\ c++11
template<class T, class U>
auto add(T x, U y) -> decltype(x+y)
{
    return x + y;
}
```

## std::function

std::function是一个函数包装器模板，其对象实例可以包装下列其中可调用元素类型：函数、函数指针、类成员函数、类成员函数指针或任意类型的函数对象。

```c++
// c++11
// 函数包装
int sum(int a, int b) { return a+b; }
function<int(int,int)> fsum = &sum;
fsum(4,2);

// 包装类成员函数
struct Foo { void f(int i){} };
function<void(Foo&,int)> fmember = mem_fn(&Foo::f);
Foo foo;
fmember(foo, 42);

// 借用bind
function<void(int)> fmember = bind(&Foo::f, foo, _1);
fmember(42);
```

## std::bind

std::bind可以将调用对象和其参数一起进行绑定。绑定后可以使用std::function进行保存，并延迟到需要调用的时候进行调用。

```c++
// c++11
float div(float a, float b) {return a/b;}
div(6,1); // 直接调用 6/1

function<float(float,float)> inv_div = bind(div, _2, _1);
inv_div(6,1); // 变换参数绑定后进行调用 1/6

function<float(float)> div_by_6 = bind(div, _1, 6);
div_by_6(1); // 添加默认参数后，进行调用 1/6

// 简化实现的例子
linear_congruential_engine<uint64_t, 1103545, 123, 21478> generator(1127590);
uniform_int_distribution<int> distribution(1,6);
int rnd = distribution(generator);

auto dice = bind(distribution, generator);
int rnd = dice() + dice() + dice();
```

## function objects

**废弃的binders和adaptor**

- unary_function,
- binary_function,
- ptr_fun,
- `pointer_to_unary_function`
- `pointer_to_binary_function`
- `mem_fun`
- `mem_fun_t`
- `mem_fun1_t`
- `const_mem_fun_t`
- `const_mem_fun1_t`
- `mem_fun_ref`
- `mem_fun_ref_t`
- `mem_fun1_ref_t`
- `const_mem_fun_ref_t`
- `const_mem_fun1_ref_t`
- `binder1st`
- `binder2nd`
- `bind1st`
- `bind2nd`

**保留使用的**

- Function wrappers
    - function
    - mem_fn
    - `bad_function_call`
- Bind
    - bind
    - `is_bind_expression`
    - is_placeholder
    - `_1,_2,_3,...`
- Reference wrappers
    - reference_wrapper
    - ref
    - cref

## lambdas

lambda表达式能够捕获作用域中变量的无名函数对象。

```c++
// c++03
// 在c++03，需要将函数对象传入的时候，通常会借用struct，如下
struct functor {
    int &a;
    functor(int& _a) : a(_a){}
    bool operator()(int x) const { return a==x; }
};

int a=42;
count_if(v.begin(), v.end(), functor(a));

// c++11
// 这在c++11中，可以通过lambda表达式进行简化实现
count_if(v.begin(), v.end(), [&a](int x){return x==a;});

// lambda表达式中不同传入变量的作用域,以及闭包
void test() {
    int x=4;
    int y=5;
    [&](){x=2;y=2;}();         // 最后一个(),表示闭包，即该lambda表达式会直接执行，&表示参数会以引用的方式传入
    // 此时，test作用域下：x=2，y=2；lambda作用域下：x=2，y=2

    [=]() mutable{x=3;y=5;}();   // =表示值以传值的方式传入
    // 此时，test作用域下：x=2，y=2；lambda作用域下：x=3，y=5

    [=,&x]() mutable{x=7;y=9;}(); // 表示x以引用的方式传递，y以值的方式传递
    // 此时，test作用域下：x=7，y=2；lambda作用域下：x=7，y=9
}

// 递归的lambda实现
function<int(int)> f = [&f](int n) {
    return n<=1 ? 1 : n*f(n-1);
};

int x=f(4);
```

## std::tuple

std::tuple是固定大小的异值类集，是通用化的std::pair.

```c++
// c++11

tuple<int,float,string> t(1,2.f,"text");
int x=get<0>(t);
float y=get<1>(t);
string z=get<2>(t);

int myint;
char mychar;
tuple<int,float,char> mytuple;
// 将不同的值打包成一个tuple
mytuple = make_tuple(10,2.6,'a');
// 将tuple内的值解包到各自的变量
tie(myint,ignore,mychar) = mytuple;
```

## std::tuple/std::tie用于逻辑比较简化实现

```c++
// c++03
struct Student {
    string name;
    int classId;
    int numPassedExams;

    bool operator<(const Student& rhs) const {
        if (name < rhs.name) return true;

        if (name == rhs.name) {
            if (classId < rhs.classId) return true;

            if （numPassedExams == rhs.numPassedExams) return numPassedExams < rhs.numPassedExams;
        }
        
        return false;
    }
};

// c++11
struct Stuent {
    string name;
    int classId;
    int numPassedExams;

    bool operator<(const Student& rhs) const {
        return tie(name, classId, numPassedExams) <
            tie(rhs.name, rhs.classId, rhs.numPassedExams);
    }
};
```

## 一致性初始化和std::initializer_list

```c++
// 数组初始化
int a[] = {1,2,3,4,5}; // c++03
int a[] = {1,2,3,4,5}; // c++11

// vector初始化
// c++03
vector<int> v;
for (int i=1; i<=5; ++i) v.push_back(v);

// c++11
vector<int> v = {1,2,3,4,5};

// map初始化
// c++03
map<int,string> labels;
labels.insert(make_pair(1,"open"));
labels.insert(make_pair(2,"close"));
labels.insert(make_pair(3,"reboot"));

// c++11
map<int,string> labels{ {1,"open"}, {2,"close"}, {3,"reboot"} };

// 作为返回值
// c++03
Vector3 normalize(const Vector3& v) {
    float inv_len = 1.f/length(v);
    return Vector3(v.x*inv_len,v.y*inv_len,v.z*inv_len);
}

// c++11
Vector3 normalize(const Vector3& v) {
    float inv_len = 1.f/length(v);
    return {v.x*inv_len,v.y*inv_len,v.z*inv_len};
}
```

上述的初始化是怎么实现的呢，采用的是`std::initializer_list`。当初始化一个vector时，`vector<int> v = {1,2,3,4,5};`，其实调用的是`vector(initializer_list<T> args)`。intializer的优先级高于普通的构造函数。一致性初始化能够解决如下的问题：

```c++
struct B{ B(){} };
struct A{ A(B){} void f(){} };
int main() {
    A a(B()); // 这是一个函数定义，可以用 A a{B()}; 进行替代
    a.f(); // 此处会出现一个编译错误
    return 0；
}
```

## Using

```c++
// c++03
typedef int int32_t;
typedef void (*Fn)(double);

// 下面的使用方式是非法的，具体如下
template <int U, int V> class Type;
typedef Type<42,36> ConcreteType;
template<int V>
typedef Type<42,V> MyType; // 这个是非法的c++代码
MyType<36> object;

// 通常的解决方法，是再构建一个结构体，如下：
template<int V>
struct meta_type {
    typedef Type<42, V> type;
}；
typedef meta_type<36>::type MyType;
MyType object;

// 如果使用using的话问题可以得到明显的简化
using int32_t = int;
using Fn = void(*)(double);
template<int U, int V> class Type;
using ConcreteType = Type<42,36>;
template<int V>
using MyType = Type<42,V>;
MyType<36> object;
```

## explicit 转换操作

C++中的explicit关键字只能用于修饰只有一个参数的类构造函数 , 它的作用是表明该构造函数是显示的, 而非隐式的.在c++11中，还可以用于operator。

```c++
// c++03
struct A { explicit A(int){} };
void f(A) {}

int main() {
    A a(1);
    f(1); // error: implicit cast
    return 0;
}

// c++03
struct A { A(int){}; };
struct B {
    int m;
    B(int x) : m(x) {}
    operator A() {return A(m);}
};

void f(A){}

int main() {
    B b(1);
    A a=b; // 隐式转换
    f(b);  // 隐式转换
    return 0;
}

// c++11
// explicit在
struct A { A(int){}; };
struct B {
    int m;
    B(int x) : m(x) {}
    explicit operator A() {return A(m);}
};

void f(A){}

int main() {
    B b(1);
    A a=b; // error,可以按如下方式使用 A a=static_cast<A>(b)
    f(b);  // error,可以按如下方式使用 f(static_cast<A>(b))
    return 0;
}
```

## default 和 delete

delete的提出，为了让程序员能够显示的禁用某个函数; 编译器能够将显示声明为defaulted的函数自动生成函数体。

```c++
// c++11
class A
{
public:
    A& operator=(A) = delete; // 不允许赋值拷贝构造
    A(const A&) = delete; // 不允许拷贝构造
    A(float); //能够初始化为float
    A(long) = delete; // 但不能初始化为long
    virtual ~A() = default;
};
```

## 枚举类 enum class

```c++
// c++03
enum Alert { green, yellow, red};
//enum Color {red, blue};
//error C2356: 'red': redefinition

Alert a = 7; // error
int a2 = red; // ok,隐式转换
int a3 = Alert::red; // error

// c++11
enum class Alert { green, yellow, red };
enum class Color : int{red, blut};

Alert a=7; // error
Color c = 7; //error
int a2 = red; // error,不支持隐式转换成int
int a3=Alert::red // error
Color a4 = Color::blue; //ok

```

## 用户定义的字面量 user-defined literals

允许通过自定义后缀的方式，对整形，浮点型，char，字符串常量生成用户自定义类型。

Allow integer, floating-point, character, and string literals to produce objects of user-defined type by defining a user-defined suffix.

```c++
// c++03
123 //int
1.2 //double
1.2F //float
'a' //char
1ULL //unsigned long long

// c++11
int operator"" _km(long double val) { return (int)val; }

auto len = 1.2_km;
```

Parctical usage:
http://www.codeproject.com/Articles/447922/Application-of-Cplusplus11-User-Defined-Literals-t

## move含义