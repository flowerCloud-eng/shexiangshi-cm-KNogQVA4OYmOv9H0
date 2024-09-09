[合集 \- C\+\+系列(18\)](https://github.com)[1\.C\+\+性能优化 —— \_\_builtin\_prefetch()数据预读2023\-02\-02](https://github.com/qiangz/p/17085951.html)[2\.C\+\+性能优化——likely和unlikely分支预测2023\-02\-03](https://github.com/qiangz/p/17088276.html)[3\.C\+\+: 智能指针的自定义删除器 \`Custom Deleter\` 有什么用？2023\-12\-16](https://github.com/qiangz/p/17904766.html)[4\.C\+\+ shared\_ptr是线程安全的吗？2023\-11\-22](https://github.com/qiangz/p/17847987.html)[5\.C\+\+ weak\_ptr除了解决循环引用还能做什么？2023\-11\-20](https://github.com/qiangz/p/17843039.html)[6\.C\+\+ \| 每一个C\+\+程序员都应该知道的RAII2023\-10\-29](https://github.com/qiangz/p/17795846.html)[7\.C\+\+ : 仅添加一个引用\& 就直接导致程序崩溃06\-07](https://github.com/qiangz/p/18236384)[8\.C\+\+: 如何高效地往unordered\_map中插入key\-value06\-07](https://github.com/qiangz/p/18236381)[9\.C\+\+: 虚函数，一些可能被忽视的细节03\-31](https://github.com/qiangz/p/18106666)[10\.C\+\+ vector 访问元素用 at 和 \[] 有什么区别？01\-19](https://github.com/qiangz/p/17973723):[veee加速器](https://liuyunzhuge.com)[11\.为什么C\+\+ 单例局部static初始化是线程安全的？01\-14](https://github.com/qiangz/p/17964054)[12\.C\+\+源码中司空见惯的PIMPL是什么？01\-13](https://github.com/qiangz/p/17963162)[13\.万字长文全面详解现代C\+\+智能指针：原理、应用和陷阱2023\-12\-18](https://github.com/qiangz/p/17911186.html)[14\.C\+\+ 高效使用智能指针的8个建议2023\-12\-16](https://github.com/qiangz/p/17904768.html)[15\.C\+\+ : 如何用C语言实现C\+\+的虚函数机制？06\-30](https://github.com/qiangz/p/18276995)[16\.C\+\+: 16个基础的C\+\+代码性能优化实例06\-27](https://github.com/qiangz/p/18270166)[17\.C\+\+ std::shared\_ptr自定义allocator引入内存池08\-07](https://github.com/qiangz/p/18346270)18\.C\+\+17： 用折叠表达式实现一个IsAllTrue函数09\-08收起
## 前言


让我们实现一个 `IsAllTrue` 函数，支持变长参数，可传入多个表达式，必须全部计算为true，该函数才返回true。


本文记录了逐步实现与优化该函数的思维链，用到了以下现代C\+\+新特性知识，适合对C\+\+进阶知识有一定了解的人。这样一种从实际问题来学习和运用知识的过程还是挺有趣的，特此整理分享一下。


1. 可变长参数模板 (C\+\+11\)
2. 折叠表达式 (C\+\+17\)
3. 条件编译 if constexpr (C\+\+17\)
4. 类型萃取 type traits (C\+\+11\)
5. 完美转发`std::forward` (C\+\+11\)
6. 结构化绑定 `std::bind` (C\+\+11\)


## 初级版本——基于初始化列表实现


可以使用初始化列表 `std::initializer_list` 存储多个bool变量，实现传入多个bool值的目的，这种方法实际上该函数只有一个参数，实现如下：



```
bool IsAllTrue(const std::initializer_list<bool>& conditions) {
    return std::all_of(conditions.begin(), conditions.end(), [](const bool a) {
        return a;
    });
}

```

使用方法如下：



```
int a = 1;
bool b = true;
auto c = []() {return true;}
IsAllTrue({a, b, c});

```

这个方法的实现简单易用，但是对于代码有更高追求的人并不满足于此，以上实现存在如下问题：


1. 传入参数是一个初始化列表，需要写大括号{}，不够优雅。
2. 调用函数前计算了每一个条件表达式，但实际任意一个为false，即可返回，可能存在如下问题：
	1. 不必要的函数调用带来一定计算开销；
	2. 当前后表达式存在依赖关系时，比如 `p && p →a` ，如果p是指针且为空， 计算`p→a` 会导致程序崩溃。


对于不了解这个函数用法的人而言，使用这个实现是会存在一定风险的。所以我们需要想办法利用 `&&` 实现短路求值，以及对函数结果的延迟计算。


## 进阶版本——基于折叠表达式实现


### 折叠表达式(**Fold expressions**)


![](https://files.mdnice.com/user/15348/c84f774e-7c75-465f-8686-d14647c6eddd.png)


折叠表达式是C\+\+17引入的新特性，可通过二元操作符折叠可变长参数模板中的参数包。这个特性的引入是为了简化C\+\+11可变长参数模板的使用。


* 根据左右方向可分为**左折叠**和**右折叠**：


一元左折叠（Unary right fold）和一元右折叠（Unary left fold）形式如下：



```
( pack op... )  //一元右折叠，从右往左计算， 等同于(E1 op (... op (EN-1 op EN)))
( ... op pack ) //一元左折叠，从左往右计算， 等同于(((E1 op E2) op ...) op EN)

```

在大多数情况下，对于交换律成立的操作符（如 `+` 和 `*`），左折叠和右折叠的结果是相同的。然而，对于非交换的操作符，结果可能不同，例如减法或除法。


* 根据是否有初始值可分为**一元**和**二元**：


二元折叠表达式分为：二元右折叠（Binary right fold）和 二元左折叠（Binary left fold）。



```
( pack op ... op init )	 //二元右折叠
( init op ... op pack )	 //二元左折叠

```

* 使用二元左折叠的例子



```
template<typename... Args>
void printer(Args&&... args)
{
     ((std::cout<< args << " "), ...)<< "\n";
}

```

### 基于一元右折叠的IsAllTrue函数


基于 `&&`运算符的一元右折叠(Unary right fold)实现IsAllTrue如下：



```
template<typename... Args>
bool IsAllTrue(Args... args) { 
	return (std::forward(args) && ...); 
}

```

* 注：折叠表达式的最外层括号是必须的。


但以上实现，该模板本质上仍只能支持变长的多个bool参数，这会导致先计算出bool值再传入，仍未实现函数结果的延迟计算。


### 使用type traits 进一步优化


如何可以实现延迟计算呢？首先我们可以明确下，传递给该函数的参数类型，可能是bool值、可以计算出bool值的表达式或可调用对象、可转换为bool值的指针和数值。


总体可分为两类，一类是可转换为bool的表达式，另一类是可计算出bool的可调用对象。


由于参数类型(bool、函数对象、指针等)和类型特征(是否可调用、是否可以转成bool)均是可以在编译期确定的。


为了避免在编译期把模板参数类型都推断为bool，可定义 `IsTrue` 函数模板定义表达式bool值的计算方式，使模板可以推断出原表达式自身的类型，从而可以延迟其计算过程。其中用到了**编译期条件**`if constexpr` 和 一种**类型萃取**是否可调用 `std::is_invocable_v` ，这两个均是C\+\+17引入的特性。


如果具备可调用的特征，则进行函数调用并返回结果；否则，将其转换为bool值返回。实现如下：



```

template <typename T>
bool IsTrue(T&& value) {
    if constexpr (std::is_invocable_v) {
        // 如果是可调用对象，调用它并返回结果
        return std::forward(value)();
    } else {
        // 否则，将其转换为bool
        return static_cast<bool>(std::forward(value));
    }
}

```

基于以上模板改写 `IsAllTure` 模板函数 ：



```
template <typename... Args>
bool IsAllTrue(Args&&... args) {
    return (IsTrue(std::forward(args)) && ...);
}

```

该实现的本质是我们希望在用N个表达式传入该模板函数后，模板实例化为形同如下形式，从而可以实现短路机制：



```
static_cast<bool>(Expr1) && Expr2() && static_cast<bool>(Expr3) && ... && ExprN()

```

### 函数测试


对以上代码进行如下测试，注释为输出结果，可以看到，能够满足我们的需求：



```
auto lambdaTrue = []() { 
    std::cout<<" lambda true"<return true; 
};
auto lambdaFalse = []() { 
    std::cout<<" lambda false"<return false; 
};
class Foo  {
public:
    int a;
};
Foo* p = nullptr;
IsAllTrue(true, lambdaTrue);  // 输出lambda true
IsAllTrue(false, lambdaTrue); // 无输出，实现了短路机制以及延迟计算
IsAllTrue(p, p->a);  // 正常运行，不会coredump

```

以上为了方便，均使用定义了无参lambda函数进行了测试。为了延迟一般含参函数的计算结果，能够方便传入带参数的函数对象，还可以基于`std::bind`实现一个用于生成可调用对象的函数：



```
template <typename F, typename... Args>
auto make_callable(F&& f, Args&&... args) {
    return std::bind(std::forward(f), std::forward(args)...);
}

```

比如：



```
bool less(int a, int b) {
	return a < b;
}

IsAllTrue(true, make_callable(less, 1, 2));

```

完整测试代码：[https://compiler\-explorer.com/z/fTvq7Y36Y](https://github.com)


## 知识总结


本文使用了以下C\+\+知识实现了一个高效的IsAllTrue函数，优点为它的使用安全且较为高效，缺点在于代码实现较为复杂，对C\+\+知识掌握程度要求较高，过多使用也会导致代码体积膨胀。


1. 条件编译`if constexpr` ：
	* 这个关键字用于在编译时判断是否满足条件。如果 `T` 是可调用对象（例如 `lambda` 或函数对象），则调用它并返回结果。
	* 如果 `T` 不是可调用对象，则将其转换为 `bool`。
2. 类型萃取`std::is_invocable_v`：
	* 这是一个用于判断类型 `T` 是否可调用的特性。如果 `T` 是可调用对象，则 `std::is_invocable_v` 返回 `true`。
	* 需要包含  头文件
3. 完美转发 `std::forward`：
	* `std::forward(value)` 确保参数的完美转发，保留其左值或右值性质。
4. 可变长参数模板：支持可变数量的参数包，语法用 `T ... args`表示。
5. 折叠表达式
	* 使用了C\+\+17中的折叠表达式 ，它会对参数从左到右进行求值。
	* 简化了可变长参数模板的使用，提供了一种简洁而直观的方式来对参数包进行展开和操作，从而避免了递归或显式循环的繁琐。
6. 结构化绑定 `std::bind` ：可绑定参数args到一个函数f，并返回一个可调用对象。


## 参考


1. [https://en.cppreference.com/w/cpp/language/fold](https://github.com)




---


如果本文对您有帮助，请点赞、关注！


![](https://img2024.cnblogs.com/blog/1044550/202409/1044550-20240908223936653-1887993879.jpg)


