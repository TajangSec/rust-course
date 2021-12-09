# 所有权

所有的程序都必须和计算机内存打交道，如何从内存中申请空间来存放程序的运行内容，如何在不需要的时候释放这些空间，成了重中之重，也是所有编程语言设计的难点之一。在计算机语言不断演变过程中，出现了三种流派：
- **垃圾回收机制(GC)**，在程序运行时不断寻找不再使用的内存，典型代表：Java、Go
- **手动管理内存的分配和释放**, 在程序中，通过函数调用的方式来申请和释放内存，典型代表：C++
- **通过所有权来管理内存**，编译器在编译时会根据一系列规则进行检查

其中Rust选择了第三种，最妙的是，这种只发生在编译器，因此对于程序运行期，不会有任何性能上的损失。

因为所有权是一个新概念，因此读者需要花费一些时间来掌握它，一旦掌握，海阔天空任你跃，在本章，我们将通过`字符串`来引导讲解所有权的相关知识。

## 一段不安全的代码

先来看看C语言的一段糟糕代码：
```c
int* foo() {
    int a;          // 变量a的作用域开始
    a = 100;
    char *c = "xyz";   // 变量c的作用域开始
    return &a;
}                   // 变量a和c的作用域结束
```

这段代码虽然可以编译通过，但是其实非常糟糕，变量`a`和`c`都是局部变量，函数结束后将局部变量`a`的地址返回，但局部变量`a`存在栈中，在离开作用域后，局部变量所申请的栈上内存都会被系统回收，从而造成了`悬空指针(Dangling Pointer)`的问题。这是一个非常典型的内存安全问题。很多编程语言都存在类似这样的内存安全问题。再来看变量`c`，`c`的值是常量字符串，存储于常量区，可能这个函数我们只调用了一次，我们可能不再想使用这个字符串，但`xyz`只有当整个程序结束后系统才能回收这片内存，这点让程序员是不是也很无奈？

所以内存安全问题，一直都是程序员非常头疼的问题，好在在Rust中，这些问题即将成为历史，那Rust如何做到这一点呢？

在正式进入主题前，先来一个预热知识。

## 栈（Stack）与堆（Heap）

栈和堆是编程语言最核心的数据结构，但是在很多语言中，你并不需要经常考虑到栈与堆。 不过对于Rust这样的系统编程语言，值是位于栈上还是堆上非常重要,因为这会影响程序的行为和性能。

栈和堆的核心目标就是为程序在运行时提供可供使用的内存空间，关于它们的详细解释和实现方式，请参见[Rust代码鉴赏](https://codes.rs/data-structures/heap.html)一书.

#### 栈

栈以放入值的顺序存储值并以相反顺序取出值。这也被称作 **后进先出**。想象一下一叠盘子：当增加更多盘子时，把它们放在盘子堆的顶部，当需要盘子时，也从顶部拿走。不能从中间也不能从底部增加或拿走盘子！增加数据叫做 **进栈**，移出数据则叫做 **出栈**。

因为上述的实现方式，栈中的所有数据都必须占用已知且固定的大小的内存空间，如果数据大小是未知的，那么在取出数据时，你将无法取到你想要的数据。


#### 堆

与栈不同，对于大小位置或者可能变化的数据，我们需要将它存储在堆上。

当向堆上放入数据时，你需要请求一定大小的空间。操作系统在堆的某处找到一块足够大的空位，把它标记为已使用，并返回一个表示该位置地址的 **指针**。这个过程称作 **在堆上分配内存**，有时简称为 “分配”（allocating）。 接着，该指针会被推入`栈`中，因为指针的大小时已知且固定的，在后续使用过程中，你将通过栈中的指针，来获取数据在堆上的实际内存位置，进而访问该数据。

因此堆是一种缺乏组织的数据结构。想象一下去餐馆就座吃饭: 进入餐馆，告知服务员有几个人，然后它找到一个够大的空桌子(堆上分配的内存空间)并领你们过去。如果有人来迟了，他们也可以通过桌号(栈上的指针)来找到你们坐在哪。

#### 性能区别

入栈比在堆上分配内存要快，因为入栈时操作系统无需分配新的空间：新数据的位置放入栈顶。相比之下，在堆上分配内存则需要更多的工作，这是因为操作系统必须首先找到一块足够存放数据的内存空间，并接着做一些记录为下一次分配做准备。

访问堆上的数据比访问栈上的数据慢，因为必须通过指针来访问。得益于CPU高速缓存，现代处理器访问内存的次数越少则越快。继续类比，假设有一个服务员在餐厅里处理多个桌子的点菜,在一个桌子报完所有菜后再移动到下一个桌子是最有效率的。但是如果在桌子A点菜完，再去一个比较远的桌子B点菜，就会比较慢。

出于同样原因，处理器在处理栈上数据的时候比处理堆上的数据更加高效，同时，在堆上分配大量的空间也可能消耗时间。


#### 所有权与堆栈

当你的代码调用一个函数时，传递给函数的参数(包括可能指向堆上数据的指针和函数的局部变量)依次被压入栈中, 当函数调用结束时，这些值将被从栈中按照相反的顺序依次移除。

因为堆上的数据缺乏组织，因此跟踪这些数据何时分配和释放是非常重要的，否则堆上的数据将产生内存泄漏:这些数据将永远无法被回收。这就是Rust所有权系统为我们提供的强大保障。 

对于其他很多编程语言，你确实无需理解堆栈的原理，但是在Rust中，明白堆栈的原理，对于我们理解所有权的工作原理会有帮助.



## 所有权原则

理解了堆栈，接下来看一下*关于所有权的规则*，首先请谨记以下规则：
> 1. Rust中每一个值都`有且只有`一个所有者(变量)
> 2. 当所有者(变量)离开作用域范围时，这个值将被丢弃



#### 变量作用域

作用域是一个变量在程序中有效的范围, 假如有这样一个变量：
```rust
let s = "hello"
```

变量`s`绑定到了一个字符串字面值，该字符串字面值是硬编码到程序代码中的。`s`变量从声明的点开始直到当前作用域的结束都是有效的：
```rust
{                      // s 在这里无效, 它尚未声明
    let s = "hello";   // 从此处起，s 是有效的

    // 使用 s
}                      // 此作用域已结束，s 不再有效
```

简而言之，`s`从创建伊始就开始有效，然后有效期持续到它离开作用域为止，可以看出，就作用域这个概念，Rust语言跟其他编程语言没有区别。

#### 简单介绍String类型

之前提到过，本章会用String作为例子，因此这里会进行一下简单的介绍，具体的String学习请参见[String类型](../string.md)。

我们已经见过字符串字面值`let s ="hello"`，即被硬编码进程序里的字符串值。字符串字面值是很方便的，但是它并不适用于所有场景。原因有二：
- 字符串字面值是不可变的,因为被硬编码到程序代码中
- 并非所有字符串的值都能在编写代码时就知道


例如，要是想获取用户输入并存储该怎么办呢？这种情况，字符串字面值就完全无用武之地，为此，Rust 有第二个字符串类型，String。这个类型被分配到堆上，所以能够存储在编译时未知大小的文本，可以使用下面的方法基于字符串字面量来创建`String`类型:
```rust
let s = String::from("hello");
```

`::`是一种调用操作符，这里表示调用`String`中的`from`方法，因为String存储在堆上，你也可以修改它：
```rust
let mut s = String::from("hello");

s.push_str(", world!"); // push_str() 在字符串后追加字面值

println!("{}", s); // 将打印 `hello, world!`
```

那么问题来了，为啥`String`可变，而字符串字面值却不可以？


## 内存与分配

就字符串字面值来说，我们在编译时就知道其内容，所以文本被直接硬编码进最终的可执行文件中。这使得字符串字面值快速且高效。不过这些特性都只得益于字符串字面值的不可变性。不幸的是，我们不能为了每一个在编译时大小未知的文本而将一块内存放入二进制文件中，并且它的大小还可能随着程序运行而改变。

对于 `String` 类型，为了支持一个可变、可增长的文本片段，需要在堆上分配一块在编译时未知大小的内存来存放内容，这些都是在程序运行时完成的：
- 首先向操作系统请求内存来存放`String`对象
- 在使用完成后，将内存释放，归还给操作系统

其中第一个由`String::from`完成，它创建了一个全新的String.

重点来了，到了第二部分，就是百家齐放的环节，在有**垃圾回收GC**的语言中，GC来负责标记并清除这些不再使用的内存对象，这些都是自动完成，无需开发者关心，非常简单好用；在无GC的语言，是开发者手动去释放这些内存对象，就像创建对象一样，需要通过编写代码来完成，因为未能正确释放对象造成的经济简直不可估量.

对于Rust而言，安全和性能是写到骨子里的核心特性，使用GC牺牲了性能，使用手动管理内存牺牲了安全，那该怎么办？为此，Rust的开发者想出了一个无比惊艳的办法：变量在离开作用域后，就自动释放其占用的内存:

```rust
{
    let s = String::from("hello"); // 从此处起，s 是有效的

    // 使用 s
}                                  // 此作用域已结束，
                                   // s 不再有效，内存被释放
```

与其它系统编程语言的`free`函数相同，Rust也提供了一个释放内存的函数:`drop`，但是不同的是，其它语言要手动调用`free`来释放每一个变量占用的内存，而Rust则在变量离开作用域时，自动调用`drop`函数: 上面代码中，Rust 在结尾的 `}` 处自动调用 `drop`。

> 其实，在 C++ 中，也有这种概念: *Resource Acquisition Is Initialization (RAII)*。如果你使用过 RAII 模式的话应该对 Rust 的 `drop` 函数并不陌生

这个模式对编写 Rust 代码的方式有着深远的影响，不过上面的例子还是太简单，来看看其它场景。

## 变量绑定背后的数据交互

#### 转移所有权

先来看一段代码：
```rust
let x = 5;
let y = x;
```

代码背后的逻辑很简单:“将 5 绑定到变量x；接着拷贝x的值赋给y”,最终`x`和`y`都等于`5`,因为整数是由固定大小的简单值，因此这两个值都被存在栈中，完全无需在堆上分配内存。

可能有同学会有疑问：这种拷贝不消耗性能吗？实际上，这种栈上数据的拷贝非常非常快，而且数据本身也足够简单，只要复制一个整数大小的内存即可，因此在这种情况下，拷贝的速度远比在堆上创建内存来得快的多。实际上，上一章我们讲到的Rust基本类型都是通过自动拷贝的方式来赋值的，就像上面代码一样。

然后再来看一段代码：
```rust
let s1 = String::from("hello");
let s2 = s1;
```
此时，可能某个大聪明(善意昵称)已经想到了：嗯，把s1的内容拷贝一份赋值给s2，实际上，并不是这样。之前也提到了，对于基本类型(存储在栈上)，Rust会自动拷贝，但是`String`不是基本类型，而是存储在堆上的，因此并不能自动拷贝。

实际上，`String`类型是一个复杂类型，由存储在栈中的堆指针、字符串长度、字符串容量共同组成，其中堆指针是最重要的，它指向了真实存储字符串内容的堆内存，至于长度和容量，如果你有Go语言的经验，这里就很好理解：容量是堆内存分配空间的大小，长度是目前已经使用的大小, 详情见[字符串](../string.md#String底层剖析)一节.

总之`String`类型指向了一个堆上的空间，这里存储着它的真实数据，这里对上面代码中的`let s2 = s1`分成两种情况讨论：
1. 拷贝所有数据
如果该语句是拷贝所有数据(深拷贝)，那么无论是`String`本身还是底层的堆上数据，都会被全部拷贝，这对于性能而言会造成非常大的影响

2. 只拷贝`String`本身
这样的拷贝非常快，因为在64位机器上就拷贝了`8字节的指针`、`8字节的长度`、`8字节的容量`，总计24字节，但是带来了新的问题，还记得我们之前提到的所有权规则吧？其中有一条就是，一个值只允许有一个所有者，而现在这个值(堆上的真实字符串数据)有了两个所有者：`s1`和`s2`。

好吧，就假定一个值可以拥有两个所有者，会发生什么呢？

之前我们提到过当变量离开作用域后，Rust 自动调用 `drop` 函数并清理变量的堆内存。不过由于两个`String`指向了同一位置。这就有了一个问题：当 s2 和 s1 离开作用域，它们都会尝试释放相同的内存。这是一个叫做 二次释放（double free）的错误，也是之前提到过的内存安全性 bug 之一。两次释放（相同）内存会导致内存污染，它可能会导致潜在的安全漏洞。

因此，Rust这样解决问题：**当`s1`赋予`s2`后，Rust认为`s1`不再有效，因此也无需在`s1`离开作用域后`drop`任何东西，这就是把所有权从`s1`转移给了`s2`**.

再来看看，在所有权转移后再来使用旧的所有者，会发生什么：
```rust
let s1 = String::from("hello");
let s2 = s1;

println!("{}, world!", s1);
```

因为Rust禁止你使用无效的引用，你会看到以下的错误
```console
error[E0382]: use of moved value: `s1`
 --> src/main.rs:5:28
  |
3 |     let s2 = s1;
  |         -- value moved here
4 |
5 |     println!("{}, world!", s1);
  |                            ^^ value used here after move
  |
  = note: move occurs because `s1` has type `std::string::String`, which does
  not implement the `Copy` trait
```

现在再回头看看之前的规则，相信你已经有了更深刻的理解：
> 1. Rust中每一个值都`有且只有`一个所有者(变量)
> 2. 当所有者(变量)离开作用域范围时，这个值将被丢弃

如果你在其他语言中听说过术语 浅拷贝（shallow copy）和 深拷贝（deep copy），那么拷贝指针、长度和容量而不拷贝数据可能听起来像浅拷贝。不过因为 Rust 同时使第一个变量`s1`无效了，因此这个操作被称为 移动（move），而不是浅拷贝。上面的例子可以解读为 s1 被 移动 到了 s2 中。那么具体发生了什么，用一张图简单说明：

<img alt="s1 moved to s2" src="/img/ownership01.svg" class="center" style="width: 50%;" />

这样就解决了我们的问题，s1不再指向任何数据，只有s2是有效的，当`s2`离开作用域，它就会释放内存。 相信此刻，你应该明白了，为什么Rust称呼`let a = b`这种为**变量绑定**了吧？


#### 克隆(深拷贝)

首先，**Rust 永远也不会自动创建数据的 “深拷贝”**。因此，任何**自动**的复制可以被认为对运行时性能影响较小。

如果我们**确实**需要深度复制`String`中堆上的数据，而不仅仅是栈上的数据，可以使用一个叫做`clone`的方法。

```rust
let s1 = String::from("hello");
let s2 = s1.clone();

println!("s1 = {}, s2 = {}", s1, s2);
```

这段代码能够正常运行，因此说明s2确实完整的复制了s1的数据。

如果代码性能无关紧要，例如初始化程序时，或者在某段时间只会执行一次时，你可以使用`clone`来简化编程。但是对于执行较为频繁的代码，使用`clone`会极大的降低程序性能，需要小心使用！

#### 拷贝(浅拷贝)

浅拷贝只发生在栈上，因此性能很高，在日常编程中，浅拷贝无处不在。

再回到之前看过的例子:
```rust
let x = 5;
let y = x;

println!("x = {}, y = {}", x, y);
```

但这段代码似乎与我们刚刚学到的内容相矛盾：没有调用 `clone`，不过依然实现了类似深拷贝的效果 - 没有报所有权的错误。

原因是像整型这样的在编译时已知大小的类型被整个存储在栈上，所以拷贝其实际的值是快速的。这意味着没有理由在创建变量 `y` 后使 `x` 无效。换句话说，这里没有深浅拷贝的区别，所以这里调用 `clone` 并不会与通常的浅拷贝有什么不同，我们可以不用管它。

Rust 有一个叫做 `Copy`的特征，可以用在类似整型这样的存储在栈上的类型上。如果一个类型拥有 `Copy`特征，一个旧的变量在将其赋值给其他变量后仍然可用。

那么什么类型是 `Copy` 的呢？可以查看给定类型的文档来确认，不过作为一个通用的规则:**任何基本类型的组合可以是 `Copy` 的，不需要分配内存或某种形式资源的类型是 `Copy` 的**。如下是一些 `Copy` 的类型：

* 所有整数类型，比如 `u32`。
* 布尔类型，`bool`，它的值是 `true` 和 `false`。
* 所有浮点数类型，比如 `f64`。
* 字符类型，`char`。
* 元组，当且仅当其包含的类型也都是 `Copy` 的时候。比如，`(i32, i32)` 是 `Copy` 的，但 `(i32, String)` 就不是。

## 函数传值与返回
将值传递给函数，一样会发生`移动`或者`复制`，就跟`let`语句一样，下面的代码展示了所有权、作用域的规则：
```rust
fn main() {
    let s = String::from("hello");  // s 进入作用域

    takes_ownership(s);             // s 的值移动到函数里 ...
                                    // ... 所以到这里不再有效

    let x = 5;                      // x 进入作用域

    makes_copy(x);                  // x 应该移动函数里，
                                    // 但 i32 是 Copy 的，所以在后面可继续使用 x

} // 这里, x 先移出了作用域，然后是 s。但因为 s 的值已被移走，
  // 所以不会有特殊操作

fn takes_ownership(some_string: String) { // some_string 进入作用域
    println!("{}", some_string);
} // 这里，some_string 移出作用域并调用 `drop` 方法。占用的内存被释放

fn makes_copy(some_integer: i32) { // some_integer 进入作用域
    println!("{}", some_integer);
} // 这里，some_integer 移出作用域。不会有特殊操作
```

你可以尝试在`takes_ownership`之后，再使用`s`，看看如何报错？例如添加一行`println!("在move进函数后继续使用s: {}",s);`。


同样的，函数返回值也有所有权，例如:
```rust
fn main() {
    let s1 = gives_ownership();         // gives_ownership 将返回值
                                        // 移给 s1

    let s2 = String::from("hello");     // s2 进入作用域

    let s3 = takes_and_gives_back(s2);  // s2 被移动到
                                        // takes_and_gives_back 中,
                                        // 它也将返回值移给 s3
} // 这里, s3 移出作用域并被丢弃。s2 也移出作用域，但已被移走，
  // 所以什么也不会发生。s1 移出作用域并被丢弃

fn gives_ownership() -> String {             // gives_ownership 将返回值移动给
                                             // 调用它的函数

    let some_string = String::from("hello"); // some_string 进入作用域.

    some_string                              // 返回 some_string 并移出给调用的函数
}

// takes_and_gives_back 将传入字符串并返回该值
fn takes_and_gives_back(a_string: String) -> String { // a_string 进入作用域

    a_string  // 返回 a_string 并移出给调用的函数
}
```


所有权很强大，避免了内存的不安全性，但是也带来了一个新麻烦，你总要把一个值传来传去去使用它，传入一个函数，很可能还要从该函数传出去，结果就是语言表达变得非常啰嗦，幸运的是，Rust提供了新功能解决这个问题。
