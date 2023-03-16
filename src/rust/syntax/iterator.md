# Rust语言迭代器使用总结

## 1.概览

本文基于本人使用Rust语言以及刷leetcode每日一题总结的经验。刚开始使用Rust的时候由于其循环语句和Cpp等语言的循环使用方式区别较大，导致初学者经常想套用其他语言比较好实现的循环过程，却常常难以下手或者写出不少带来额外开销的写法，这里简单总结一下Rust迭代器的特点，性质和使用技巧，以及怎么尽量做到zero overhead。

## 2.迭代器基本介绍

Rust语言中的迭代器是实现了Iterator trait的类型，并需要至少实现一个next函数，用于让迭代器指向下一个迭代对象，并返回一个Option<T>用于指示对象是否存在。next函数定义大致如下，Item为一个关联类型，表示所迭代的对象的类型。

```rust
fn next(&mut self) -> Option<Self::Item>;
```

例如常见的Vec就提供了一个方法返回自己的迭代器。

```rust
fn main() {
    let v=vec![1,2,3,4,5];
    for i in v.iter(){
        eprintln!("{}",i);
    }
}
```

Rust中for循环实质上是一个语法糖，in后面的对象要求是一个迭代器，for循环就是对这个迭代器循环调用next，而in前面的名称就是每一次迭代后返回的结果，如果next返回Option::None则退出循环。了解这一点后我们可以自己编写自己的迭代器类型，然后使用for循环进行迭代。也就是说下面这两种写法可以说是一样的（使用while循环而不是loop亦可）。

```rust
    //1
    let mut iter=v.iter();
    loop{
        match iter.next(){
            None => {break}
            Some(element) => {//for循环体}
        }
    }

   //2
    for element in v.iter() {
        //for 循环体
    }
```

那么为什么要使用迭代器呢？有什么好处？首先我们看下面这两段代码。

```rust
//1
fn main() {
    let vec = vec![1, 2, 3, 4, 5];
    for i in 0..5 {
        eprint!("{}",vec[i]);
    }
}
//2
fn main() {
    let vec = vec![1, 2, 3, 4, 5];
    for num in vec.iter() {
        eprint!("{}", num);
    }
}
```

熟悉C/Cpp的读者看到第一种可能更加熟悉，遍历数组按通常使用一个i做下标（这里使用了Rust的范围语法，后面会讲，先理解成C语言的for(int i=0;i<5;++i）即可），然后逐个访问数组元素。

但是在Rust中，如果没有编译器优化，这两种写法其实是不等价的，并且第二种会由于第一种。由于Rust的Vec类型的下标访问每次都会做边界检查，越界会直接panic使程序退出，而循环本身也会做一次边界检查（0加到5每次循环都要检查），这样就出现了多余的边界检查，造成额外开销，虽然编译器可能会进行优化，但是这种写法依然不值得提倡。如果迭代一些复杂的类型，可能依然会出现问题。

另外使用迭代器，可以使用很短的代码实现复杂的效果。

例如leetcode [1018. 可被 5 整除的二进制前缀](https://leetcode-cn.com/problems/binary-prefix-divisible-by-5/)，使用Cpp编写通常是这样。

```c++
class Solution {
public:
    vector<bool> prefixesDivBy5(vector<int>& A) {
        int temp = 0;
        vector<bool> res(A.size(), false);
        for (int i = 0; i < A.size(); i++) {
            temp = (temp * 2 + A[i]) % 5;
            if (temp == 0) {
                res[i] = true;
            }
        }
        return res;
    }
};
```

使用Rust标准库提供的函数，就可以很简单的完成这一题，并且丝毫不影响速度。

```rust
impl Solution {
    pub fn prefixes_div_by5(a: Vec<i32>) -> Vec<bool> {
        a.iter().scan(0, |sum, &x| {*sum = (*sum * 2 + x) % 5;Some(*sum == 0)}).collect()
    }
}
```

事实上像Vec的迭代器，其实现就是保存一头一尾两个指针，直接使用指针来遍历数组，类似Cpp里的

```c++
int arr[5]={1,2,3,4,5};
for(int* begin=arr;begin!=arr+5;++begin){
    //循环体
}
```

这样就避免了重复的边界检查，并且完全不影响效率，还要少些不少代码，是提倡使用的。另外迭代器的使用和Rust的所有权系统也密切相关，正确的使用迭代器也能辅助你编写更高效的代码。

## 3.Rust标准库提供的操作迭代器方式

Rust标准库提供了了非常多的方式来操作和再包装迭代器，可以将各种各样的迭代器和包装器进行组合，非常灵活的实现自己想要的效果，下面介绍几个比较常用的例子。

## next

next是迭代器最基本的功能，不支持next函数可以说就不叫迭代器。next函数会让迭代器指向下一个对象，并且返回一个Option<Self::Item>，如果其值为None表示下一个对象可能不存在，即迭代器走到了结束。我们既可以使用for循环来让迭代器遍历整个序列，也可以手动调用next来精细控制每一次迭代。可以说next函数是Rust整个迭代器模式的基础，标准库给出的例子如下。

```rust
let a = [1, 2, 3];

let mut iter = a.iter();

// A call to next() returns the next value...
assert_eq!(Some(&1), iter.next());
assert_eq!(Some(&2), iter.next());
assert_eq!(Some(&3), iter.next());

// ... and then None once it's over.
assert_eq!(None, iter.next());

// More calls may or may not return `None`. Here, they always will.
assert_eq!(None, iter.next());
assert_eq!(None, iter.next());
```

## filter

filter是在Iterator trait内默认实现的一个函数，只要用户自定义的类型实现了Iterator trait，那么filter就会自动提供给用户。它的作用就如名字一样，过滤掉迭代过程中不满足某个条件的元素，它的参数是一个闭包，其返回值为bool类型，指示该元素是否符合条件。比如我要打印0到100内3的倍数，用Cpp风格的for循环可以这么写。

```c++
int main()
{
    for(size_t i = 0;i<100;i++){
        if(i%3==0)
            std::cout<<i<<' ';
    }
    std::cout.flush();
}
```

用Rust提供的fillter可以这么写

```rust
    for num in (0..=100).filter(|x| x % 3 == 0) {
        eprint!("{} ", num);
    }
```

这里使用到了Rust的Range语法，简单介绍一下，也是非常常用的工具。Rust中可以使用**a..b**或者**a..=b**来表示一个范围。其本质上也是一个语法糖，相当于定义一个Range<Idx>类型的对象,其中Idx是表示范围边界的类型，目前标准库是这么定义的。使用两个成员来表示整个范围的起始和结束。

```rust
pub struct Range<Idx> {
    pub start: Idx,
    pub end: Idx,
}
```

比如1..100，就是一个Range<i32>类型，表示一个从1到100的范围，如果写成1..=100，就是包含100，否则不含100。重要的是，Range是实现了Iterator trait的类型。于是我们就可以对其进行迭代，加上上面说的filter是为所有实现了Iterator trait的类型自动实现的，所以我们自然可以使用filter来操作它。

filter的原理也很简单，就是把原来的迭代器包装一下，重新返回一个新的迭代器，比如可以这么实现（与标准库有出入，仅解释原理）

```rust
struct Filter<I, P>
where
    I: Iterator,
    P: Fn(&I::Item) -> bool,
{
    iter: I,
    pred: P,
}

impl<I, P> Filter<I, P>
where
    I: Iterator,
    P: Fn(&I::Item) -> bool,
{
    pub fn new(iter: I, pred: P) -> Self {
        Self { iter, pred }
    }
}

//重点在怎么实现next
impl<I, P> Iterator for Filter<I, P>
where
    I: Iterator,
    P: Fn(&I::Item) -> bool,
{
    type Item = I::Item;

    fn next(&mut self) -> Option<Self::Item> {
        loop {
            let next_element = self.iter.next()?; //调用一次next，获取结果，是None就直接返回
            if (self.pred)(&next_element) {       //检查是否符合条件
                return Some(next_element);        //符合则返回结果，否则继续调用next
            }
        }
    }
}

fn main() {
    for num in Filter::new(0..100, |x| *x % 3 == 0) {  //可以直接参与for循环
        eprint!("{} ", num);
    }
}
```

## enumerate

这也是个非常常用的包装器，普通的迭代器使用的时候，我们只能知道当前的迭代结果，而需要自己来记录迭代次数，有了enumerate，就可以同时记录下迭代的次数。enumerate的next返回值是Option<(usize,Self::Item)>，其中(usize,Self::Item)是一个元组，第一个值表示迭代次数，第二个值表示结果。得益于Rust的模式匹配功能，我们可以这么写。

```rust
fn main() {
    let vec = vec![1, 2, 3, 4, 5];
    for (count, num) in vec.iter().enumerate() {
        eprintln!("第{}次迭代，值为：{}", count, num);
    }
}
```

打印结果

```
第0次迭代，值为：1
第1次迭代，值为：2
第2次迭代，值为：3
第3次迭代，值为：4
第4次迭代，值为：5
```

## map

顾名思义,map即是对迭代的元素进行一次映射后再返回映射后的结果。比如我要把一个i32数组的每个元素转成字符串，并且迭代访问每个字符串，那么就可以这么写。原理也是通过包装原迭代器，读者可以自己仿照上面的filter实现方式实现一下map。

```rust
fn main() {
    let vec=vec![1,2,3,4,5];
    for num_str in vec.iter().map(|x|x.to_string()){
        eprint!("{}",num_str);
    }
}
```

## collect

collect是将一个迭代器迭代的所有元素组合成一个新的集合，比如我要生成一个存有0到100的Vec<i32>，就可以这么写。

```rust
let vec = (0..=100).collect::<Vec<_>>();//Vec的泛型参数可以不写，由编译器推导为i32.
```

上面提到的map通常配合collect函数使用，来把某个可迭代序列全部元素都转换成另一种类型的对象，并且返回一个新的列表。

```rust
let vec = vec![1, 2, 3, 4, 5];
let str_vec=vec.iter().map(|x| x.to_string()).collect::<Vec<_>>();//这里的str_vec就是一个Vec<String>了
```

同样filter也可以组合collect使用，得到一个过滤后的集合。

## rev

rev函数是让迭代器反向迭代，其要求迭代器实现DoubleEndedIterator trait，也就是不能只向前迭代，要能向后迭代才能使用rev函数。比如逆序打印0到100

```rust
    for i in (0..=100).rev() {
        eprint!("{} ", i);
    }
```

## **max**

max是求迭代元素的最大值，比较简单不多说，给个例子。

```rust
fn main() {
    let vec = vec![1, 5, 3, 4, 2];
    let max = vec.iter().max().unwrap();
    eprint!("{}", max);//输出5
}
```

## **sum**

sum是求迭代元素的和，需要指定一下结果的类型。

```rust
fn main() {
    let vec = vec![1, 2, 3, 4, 5];
    let sum = vec.iter().sum::<i32>();
    eprint!("{}", sum);//输出15
}
```

## fold

fold是一个神奇的函数，它有两个参数，第一个是初始值，第二个是一个闭包，闭包第一个参数是一个累计值，第二个参数是本次迭代元素的引用，返回值作为下一次迭代的累计值。接触过其他函数式语言的读者可能对这个函数非常熟悉。

这么说可能难以理解，举个例子，还是求和，C中这么写。

```c++
int sum=0;
int a[5]={1,2,3,4,5};
for(int i=0;i<100;++i){
    sum+=array[i];
}
```

Rust中除了直接使用sum，还可以使用fold。

```rust
 let vec = vec![1, 2, 3, 4, 5];
 let res = vec.iter().fold(0, |acc, x| acc + x);
 eprint!("{}", res);
```

其中acc在第一次迭代的时候就是初始值0，也就是fold函数第一个参数，每次迭代都会返回acc+x作为下一次acc的值，也就是每次迭代都会加上这次迭代的结果，那么结果自然就是求和了。事实上很多函数式语言里给的sum函数就是用fold实现的。

## scan

scan和fold很类似，但是它允许你直接修改累计值，并且允许你选择什么时候停止迭代，取决于你传入的闭包何时返回None。比如我不仅要求数组的和，还要获取每次累加的结果，就可以这么写。

```rust
fn main() {
    let vec = vec![1, 2, 3, 4, 5];
    for step in vec.iter().scan(0, |acc, x| {
        *acc+= *x;
        Some(*acc)
    }) {
        eprint!("{} ", step);
    }//打印1 3 6 10 15
}
```

**标准库还提供了像skip（跳过迭代n个元素），nth（返回第n个元素的结果），count（计算序列的长度）,find(查找符合条件的第一个元素）,cycle(让迭代序列无限循环），position(计算某个元素从前往后第一次出现的位置）。**

**上面这些函数很多都可以使用链式调用互相组合，能简洁灵活的操作序列，获取结果。下面开始通过举一些例子来讲怎么组合使用他们**

## 使用迭代器来解决问题

[1672. 最富有客户的资产总量](https://leetcode-cn.com/problems/richest-customer-wealth/) 

本题是简单题，可以用来练习最基本的迭代器使用。题目实质就是求一个二维数组每一维的和，然后求这些和的最大值。

那么首先我们可以先使用map函数，来将每一维映射为其和，然后使用max函数求最大值即可。一句代码即可解决，使用了map，sum，max。

```rust
impl Solution {
    pub fn maximum_wealth(accounts: Vec<Vec<i32>>) -> i32 {
        accounts
            .iter()
            .map(|vec| vec.iter().sum())
            .max()
            .unwrap()
    }
}
```

[721. 账户合并](https://leetcode-cn.com/problems/accounts-merge/)

本题为中等题，这里只介绍部分使用了迭代器的核心部分。首先前两个分割线中间的代码是把给出的二维数组的每一维的第一个元素作为账户拥有者的名称。后面的作为邮箱，构造一个Account类型对象，然后组合成一个新的Vec<Account>数组。

```rust
struct Account(String, Vec<String>);
fn union_email(accounts: &mut Vec<Account>) {
    //
}
fn accounts_merge(accounts: Vec<Vec<String>>) -> Vec<Vec<String>> {
    let mut sets = Vec::<Account>::new();
    for account in accounts {
        let mut iter = account.into_iter();
        let name = iter.next().unwrap();
        sets.push(Account(name, iter.skip(1).collect()));
    }
    union_email(&mut sets);
    sets.into_iter()
        .map(|acc| {
            let mut res = vec![acc.0];
            res.extend(acc.1);
            res
        })
        .collect()
}
```

这段代码看似不多，其实涉及到很多内容。首先这段代码

```rust
    for account in accounts {
        let mut iter = account.into_iter();
        let name = iter.next().unwrap();
        sets.push(Account(name, iter.skip(1).collect()));
    }
```

这里的accounts我没有使用accounts.iter()，而是直接使用accounts，由于这里只是把题目给的数组进行转换，后序不需要再读取它，那么直接使用accounts会导致move。而因为Vec实现了IntoIterator trait，其本身可以直接作为迭代器，每次迭代都会把被迭代元素的所有权交出去，避免了多余的复制发生。后面的let mut iter = account.into_iter();也是同理，这个循环内没有发生任何的clone动作，全都是move，而String类型的move操作比clone操作的开销要低得多。包括后面的collect也都是使用move过来的String构造的新结果。

然后是处理结果之后要按格式返回结果，结果可以直接交出所有权，所以使用intoiter避免复制，然后map将Account类型再转换回Vec<String>，这里使用到了Vec的extend函数，接受一个intoiterator迭代器，将其内容一个个move进新的Vec。最后将整个的结果通过collect集合成最终的Vec<Vec<String>>并返回。可以看到Rust的所有权在这里体现的非常自然，不需要你手动去考虑怎么move对象，而是通过指定所有权来自动完成，可以很简单的避免许多不需要的复制开销，代码也很简洁。

```rust
sets.into_iter()
        .map(|acc| {
            let mut res = vec![acc.0];
            res.extend(acc.1);
            res
        })
        .collect()
```

## 总结

Rust的迭代器给用户提供了一种灵活，通用的迭代序列的做法，并且和其所有权系统密切相连，和同样是作为系统级语言的C/Cpp有着比较大的区别，要完全理解其迭代器的设计思想和设计细节不是一件容易的事。我个人在学习初期也感觉学起来非常难受，明明在C里用的好好的写法移植到Rust里要不就是啰嗦要不就是有额外开销，甚至有时候还要上unsafe。但是在熟悉了用法和底层原理之后，发现大部分场景其实使用safe的迭代器也能很好的解决，并且速度反而更快。

另外推荐一个迭代器库[itertools](https://crates.io/crates/itertools)，这个库提供了比标准库更丰富的操作迭代器的接口，和标准库的trait兼容，相当于是一个功能扩展，具体使用方式可以参照其文档。