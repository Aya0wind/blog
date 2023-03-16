# Concept与类型擦除实现的多态

C++20是C++在C++11以后又一次巨大更新（甚至比C++11那次还大），可以说这次更新是颠覆性的，预计在明年各大编译器会支持差不多所有的C++20的特性。而其中概念是一个千呼万唤始出来的特性，在各大库用自己发明的各种千奇百怪的做法实现概念后，终于由标准库一统江湖，这篇文章来讲讲概念的基本用法以及如何使用概念约束配合类型擦除来实现本来应该用继承实现的运行期多态，默认你已经对概念的基本语法有所了解（什么？不了解？那还不快去看？）
## 一、使用概念实现的静态多态

C++的概念不仅能约束类型，也能约束接口，比如我可以写一个这样的概念：

```cpp
template<class T>
concept FlyConcept=requires (T obj) {
    obj.fly();
    {obj.land()}->std::convertible_to<bool>;
};
```
这个FlyConcept概念约定，满足这个概念的T类型有一个可无参调用的，公有的，非const的fly成员函数，以及一个返回值可转换为bool的可无参调用的，公有的，非const的成员函数。
于是我们写两个满足这个Concept的类，就可以这样调用：

```cpp
class Plane{
    void fly()
    {
        std::cout<<"Plane fly"<<'\n';
    }
    bool land()
    {
        std::cout<<"PaperPlane landing"<<'\n';//飞机着陆成功，返回true
        return true;
    }
};
class PaperPlane{
public:
    void fly()
    {
        std::cout<<"PaperPlane fly"<<'\n';
    }
    bool land()
    {
        std::cout<<"PaperPlane landing"<<'\n';//纸飞机着陆成功，返回true
        return false;
    }
};

template<FlyConcept T>
void testFly(T flyObject)
{
    flyObject.fly();
    bool success=flyObject.land();
    std::cout<<"着陆"<<(success?"成功":"失败")<<'\n';
}

int main()
{
    Plane p;
    PaperPlane pp;
    testFly(p);
    testFly(pp);
    return 0;
}
```

使用GCC10编译并运行该程序，得到下面的输出。
> Plane fly  
> PaperPlane landing  
> 着陆成功  
> PaperPlane fly  
> PaperPlane landing  
> 着陆失败  

这就是利用概念实现的静态多态，如果我们把PaperPlane的fly函数改成不满足FlyConcept的要求呢？比如我们把fly改成需要一个int参数:

```cpp
    void fly() const
    {
        std::cout<<"Plane fly"<<'\n';
    }
```
再次编译，得到如下报错(删掉了前面的目录)：
> In function 'int main()':  
>  error: use of function 'void testFly(T) [with T = PaperPlane]' with unsatisfied constraints  
>
> note: declared here  
>
> note: constraints not satisfied  
>  In instantiation of 'void testFly(T) [with T = PaperPlane]':  
>    required from here  
>  required for the satisfaction of 'FlyConcept<T>' [with T = PaperPlane]  
>   in requirements with 'T obj' [with T = PaperPlane]  
> note: the required expression 'obj.fly()' is invalid  
>  obj.fly();  

明确提示fly通不过Concept检查，报错清晰友好。
## 二、使用概念和类型擦除实现动态多态

下面进入本文正题，说到类型擦除，C程序员可能第一个想到void*，想到qsort。C++程序员可能第一个想到的是std::any，是std::functional。没错，这些都使用了类型擦除技法，不过类型擦除的具体使用方式有许多，各有特点。
### std::any

std::any是完完全全的类型擦除，可以说除了类型本身以外，所有的信息都没了，所以很多人说std::any没什么卵用，拿一个any，连内部是什么类型都需要我们用any_cast自己去试，试错了还要抛异常，实在是鸡肋。
于是人们就开始思考，类型擦除是为了什么？还不是为了运行期多态，qsort为什么一份代码能排序任何对象？还不是因为擦除了类型，然后对象的大小由长度参数给出，程序就知道怎么搬运对象了。
所以就牵涉到了运行期多态的本质，不管你写出什么花来，运行期多态必须要在某个位置保存对象或类的元数据，在运行期的时候去查找并且根据这个歌元数据去执行对应的行为。
### std::functional
那么std::functional就出现了，其内部把要调用的函数指针和对象指针保存（为了优化，存小对象的时候会改为直接存在std::funtional里面）。调用时就直接使用函数指针和对象指针，把对象指针cast一下，然后调用函数（过程比较复杂，由于要支持成员和非成员函数，有非常多的特殊处理）。这就是一种类型擦除，只保留了对象指针和这个对象可调用的一个函数的指针。
运行期多态
对于继承实现的运行期多态，我们通常使用虚表（别跟我说标准没说要用虚表），把虚表指针存对象里，然后运行时查表调用函数。
那么使用类型擦除，我们自然可以想到把对象指针和虚表指针绑定为一个擦除类，这个擦除类是统一大小，统一类型的，然后使用擦除类的虚函数去真正调用对应类的函数。
而概念，就是保证这一约定正确性的有力手段，因为这属于约定，万一写错了，如果没有概念检查，是很容易出错的。好了，talk is cheap,show me your code。
还是一样的FlyConcept概念，这次我们要进行运行期多态，我们定义一个类型擦除类，作为这个概念的配套组件，名字就叫Fly，它长这样：

```cpp
template<class T>
concept FlyConcept=requires (T obj) {
    obj.fly();
    {obj.land()}->std::convertible_to<bool>;
};

struct Fly{
    struct internal{
        virtual void fly(void* obj)=0;
        virtual bool land(void* obj)=0;
        virtual ~internal()=default;
    };
    void fly()
    {
        internalObject->fly(obj);
    }
    bool land()
    {
        return internalObject->land(obj);
    }
    template<class T>
    struct Impl: internal{
        void fly(void* obj)override
        {
            static_cast<T*>(obj)->fly();
        }
        bool land(void* obj)override
        {
            return static_cast<T*>(obj)->land();
        }
    };
    template<FlyConcept T>
    Fly(T* p):obj(p), internalObject(std::make_unique<Impl<T>>())
    {
    }
    std::unique_ptr<internal> internalObject;
    void* obj;
};
```

擦除类Fly使用一个模板构造函数，把真正的对象指针存在了内部，类型是一个满足FlyConcept概念的类型。然后定义了一个虚基类，内部使用一个模板类继承虚基类，重写其中的虚函数，内部去调用保存对象对应的真正的函数（这个过程是静态绑定的，运行时使用虚函数来派发）。
那么有人就要问了，你写一大堆内容，实现了一个继承一下就能做的东西，这么做的目的在哪呢？有什么好处吗？
下面再请出我们的Plane和PaperPlane老兄，让它们来演示一下：

```cpp
class PaperPlane{
public:
    void fly()
    {
        std::cout<<"PaperPlane fly"<<'\n';
    }
    bool land()
    {
        std::cout<<"PaperPlane landing"<<'\n';
        return false;
    }
};

class Plane{
public:
    void fly()
    {
        std::cout<<"Plane fly"<<'\n';
    }
    bool land()
    {
        std::cout<<"PaperPlane landing"<<'\n';
        return true;
    }
};
int main()
{
    Plane p;
    PaperPlane pp;
    std::vector<Fly> flyObjects;
    flyObjects.emplace_back(&p);
    flyObjects.emplace_back(&pp);
    for(auto&&e:flyObjects){
        e.fly();
        bool success=e.land();
        std::cout<<"着陆"<<(success?"成功":"失败")<<'\n';
    }
    return 0;
}
```
编译运行，打印以下结果：  
> Plane fly
> PaperPlane landing
> 着陆成功
> PaperPlane fly
> PaperPlane landing
> 着陆失败
成功实现运行期多态，并且Plane和PaperPlane没有继承任何基类，这两个类没有任何关系。我们甚至可以随便加个其他类，比如Bird。  


```cpp
class Bird{
public:
    void fly()
    {
        std::cout<<"Bird fly"<<'\n';
    }
    bool land()
    {
        std::cout<<"Bird landing"<<'\n';
        return true;
    }
};
int main()
{
    Plane p;
    PaperPlane pp;
    Bird b;
    std::vector<Fly> flyObjects;
    flyObjects.emplace_back(&p);
    flyObjects.emplace_back(&pp);
    flyObjects.emplace_back(&b);
    for(auto&&e:flyObjects){
        e.fly();
        bool success=e.land();
        std::cout<<"着陆"<<(success?"成功":"失败")<<'\n';
    }
    return 0;
}
```
打印：
> Plane fly  
> PaperPlane landing  
> 着陆成功  
> PaperPlane fly  
> PaperPlane landing  
> 着陆失败  
> Bird fly  
> Bird landing  
> 着陆成功  
那么概念在这其中起什么作用内？没错就是约束，如果我们尝试传入一个不满足FlyConcept约束的类型的指针进去呢？  

```cpp
int main()
{
    class Unimplemented{} obj;
    std::vector<Fly> flyObjects;
    flyObjects.emplace_back(&obj);
    for(auto&&e:flyObjects){
        e.fly();
        bool success=e.land();
        std::cout<<"着陆"<<(success?"成功":"失败")<<'\n';
    }
    return 0;
}
```
编译一下，发现报错了，太长我就不贴上来了，大概就是说Unimplemented类型在FlyConcept检查中失败了巴拉巴拉。
更多功能
那么又双叒叕有人问了，你这人家写个Fly interface，实现一下不就完了，整这么麻烦就为了不继承基类吗？那当然不是，所以还有一个功能没有讲，那就是对基本类型的运行时多态。
我们都知道Java有个“假”泛型容器，表面上是泛型，实际上要把所有的类型都转为Object类型，再统一装进容器，基本类型由于不是Object类型，就会触发装箱，于是就出现了这种很rz的在堆上new一个int然后把引用存起来的做法，这种做法实际上就是无脑类型擦除实现泛型。

但是基于类型擦除的多态可以是非侵入式的，并且不需要对原类型做任何修改，那就是将
