---
title: 'Go语言的栈和指针机制 '
date: 2018-09-11 11:32:32
tags:
---

阅读前请悉知：本文是一篇翻译文章，出于对原文的喜爱与敬畏，所以需要强调：如果读者英文阅读能力好，请直接移步文末原文链接；如果对这篇翻译所述知识感兴趣，也请一定要再看下英文原文，加深理解。翻译中为了表达的需要，加入了自己的一些理解，不过因为知识有限，翻译过程难免纰漏，如有问题，欢迎留言指正。

# 介绍
我不想夸赞指针，因为它很难理解，如若使用不当，极易造成bug, 甚至引发性能问题，这在编写并发或者多线程软件时，显的尤为突出。也就难怪许多编程语言都试图对程序员隐藏指针特性了。但是，当使用Go编写软件，你是没有办法避开指针的。如果对指针没有深入的理解，你将很难写出简洁高效的代码。

# 帧（边界）
函数在独立的内存空间（帧）执行，而这个独立的内存空间是有边界的，这个边界我们称之为帧边界。每个帧都允许函数在自己的上下文中运行，并提供流量控制(flow control, 暂且这么翻译)。
函数只能直接访问帧内的内存，帧外的内存不能间接访问。如果函数需要访问帧外的存储空间，则该内存必须与函数共享。为了理解接下来的内容，我们需要首先理解帧概念和机制。（我的理解是：帧是一段有限的供函数运行的内存块）

当一个函数被调用，会有两个帧发生交互, 即：代码从调用函数的帧转换到被调用函数的帧，如果函数调用需要传递数据，那么该数据必须从一个帧传递到另一个帧。在Go中，数据在两帧之间是**按值**传递的。

**按值**传递的提高了代码的可读性。函数调用中数据值从一个函数复制传递，另一个函数接收到这个值，整个过程很直观，所以你写代码时不必为了可读性而特意掩盖函数间交互的过程，因为它就是这么直观，因而这种直观可以帮助你理解每个函数调用是在如何影响程序运行的。

## Listing 1
```
01 package main
02
03 func main() {
04
05    // Declare variable of type int with a value of 10.
06    count := 10
07
08    // Display the "value of" and "address of" count.
09    println("count:\tValue Of[", count, "]\tAddr Of[", &count, "]")
10
11    // Pass the "value of" the count.
12    increment(count)
13
14    println("count:\tValue Of[", count, "]\tAddr Of[", &count, "]")
15 }
16
17 //go:noinline
18 func increment(inc int) {
19
20    // Increment the "value of" inc.
21    inc++
22    println("inc:\tValue Of[", inc, "]\tAddr Of[", &inc, "]")
23 }
```
当你执行以上代码时，golang中`runtime`会创建一个主`goroutine`，这个主`goroutine`会开始执行所有的`main`函数内所有代码的初始化。需要明白的是，`goroutine`是挂在操作系统线程上的，该线程最终在机器的某个核心上执行。在1.8版本中，每个`goroutine`都有一个初始化大小为2048个字节的连续内存块，这构成了它的堆栈空间。这个初始堆栈大小在过去几年发生了变化，将来可能会再次发生变化。

栈很重要，因为它为每个单独的函数提供了有限的物理内存空间。在主`goroutine`执行清单1中的主函数时，`goroutine`的栈是以下这样

## Figure 1
![image.png](https://upload-images.jianshu.io/upload_images/44480-b255d954cb7affb1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

你可以在图1中看到，栈的一部分已经被主函数所占据，即`main`所属的内存帧（帧在栈上分配的），这个方框表示栈上的主函数边界。帧的范围作为调用函数时执行的代码的一部分建立。你还可以看到`count`变量的内存已经放在main所在帧的地址0x10429fa4上。

图1还说明了另一个有趣的问题。活动帧以下的所有栈内存都无效，但活动帧以上的栈内存是有效的。我需要明确帧的有效部分和无效部分之间的界限（是否有被使用）。

# 地址

变量的作用是特定的内存位置赋予名字，以提高代码的可读性，并帮助你分析正在使用的数据。如果你有一个变量，那么对应内存中一个值，如果内存中有一个值，那么它必须有一个地址。在第09行，主函数调用内置函数println来显示`count`变量的“值”和“地址”。

## Listing 2

```
09    println("count:\tValue Of[", count, "]\tAddr Of[", &count, "]")
```
使用&运算符来获取变量位置的地址并不新奇，其它语言也使用这个运算符。第09行的输出应该类似于下面的输出，如果你在一个32位架构(如游乐场)上运行代码:

## Listing 3
```
count:  Value Of[ 10 ]  Addr Of[ 0x10429fa4 ]
```

# 函数调用

在第12行上，`main`函数调用了`increment`函数。

## Listing 4
```
12    increment(count)
```
调用函数意味着`goroutine`需要在栈上开辟一个新的内存空间。然而事情要可能比你想像的还要复杂一些哦。要成功地进行此函数调用，需要在转换过程中在两个帧之间传递数据。具体地说，一个整数值将在调用期间被复制和传递。通过查看第18行上的`increment`函数的声明，你可以看到这一点。

## Listing 5

```
18 func increment(inc int) {
```
如果你在第12行再次看到递增的函数调用，你会看到代码正在传递`count`变量的“值”。该值将被复制、传递给`increment`函数所在的帧中。记住，`increment`函数只能在它自己的空间内直接读写内存，因此它需要`inc`变量接收、存储和访问它自己传递的`count`值的副本。

在`increment`函数内部的代码开始执行之前，`goroutine`的栈看起来是这样的:
Figure 2
![image.png](https://upload-images.jianshu.io/upload_images/44480-0c74d10a3c1675b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到栈上现在有两个帧，一个是`main`，一个`increment`。在`increment`的帧中，你可以看到`inc`变量，它包含在函数调用期间复制和传递的值10。`inc`变量的地址是0x10429f98，内存更小，因为帧在栈中是由高地址向低地址扩展的，不过这只是一个实现细节，没有任何意义。重要的是`goroutine`从`main`的帧中获取`count`的值，并使用`inc`变量在帧中存储了该值的副本。

`increment`函数中的其余代码显示`inc`变量的“值”和“地址”。

## Listing 6
```
21    inc++
22    println("inc:\tValue Of[", inc, "]\tAddr Of[", &inc, "]")
```
22行的输出如下
## Listing 7
```
inc:    Value Of[ 11 ]  Addr Of[ 0x10429f98 ]
```
这是在执行到第22行后栈的样子:
## Figure 3
![image.png](https://upload-images.jianshu.io/upload_images/44480-03d0709e82b2e9df.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

执行第21和22行之后，`increment`函数返回到`main`函数。然后主函数在第14行再次`count`变量的“值”和“地址”。

## Listing 8

```
14    println("count:\tValue Of[",count, "]\tAddr Of[", &count, "]")
```
输出如下如示
```
count:  Value Of[ 10 ]  Addr Of[ 0x10429fa4 ]
inc:    Value Of[ 11 ]  Addr Of[ 0x10429f98 ]
count:  Value Of[ 10 ]  Addr Of[ 0x10429fa4 ]
```
`main`所在帧中`count`的值在调用`increment`前后相同。

# 函数返回

当一个函数返回到调用方函数时，栈上的内存实际发生了什么?其实什么都没有。这是`increment`函数执行完成返回后栈的样子:
## Figure 4
![image.png](https://upload-images.jianshu.io/upload_images/44480-47c7323bcebfdd41.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

与图3几乎完全相同，只是`increment`函数关联的帧现在被认为是无效内存。这是因为`main`的帧现在是活动帧。`increment`函数所在的帧在内存中保持不变，此时它是非活动帧。

清理返回函数的帧的内存会浪费时间，因为你不知道是否还需要这个内存。所以内存就保持原样了。在每次函数调用期间，在获取帧时，该帧的栈内存将被清除。这是通过初始化放置在帧中的任何值来完成的。因为所有的值都被初始化为至少它们的“零值”，所以栈在每次函数调用时都会自动清理。
（这里我理解因为每个帧其实是有边界的，程序运行时知道此时帧的边界在哪里，比如若此时`main`调用另一个函数`increment2`,可能会占据原`increment`的帧，完成初始化，相当于是覆盖了）

# 共享

有什么办法能让`increment`函数直接操作`main`的帧中存在的`count`变量呢?答案是指针。指针的存在只有一个目的，即与函数共享一个值，以便函数可以读写该值，即使该值并不直接存在于其自身的帧中。

如果你不知道共享，你就不需要使用指针。学习指针时，重要的是要使用清晰的词汇表，而不是操作符或语法。所以请记住，指针是用于共享的，并在你读取代码时将`&`操作符替换为“共享”。

# 指针类型

Go有许多内置类型， 这些内置类型都能很方便的声明为指针类型。比如已经存在一个名为`int`的内置类型，因此有一个指针类型称为`*int`。如果声明了一个名为`User`的类型，就可以获得一个名为`*User`的指针类型。

所有指针类型都具有相同的两个特征。首先，他们从角色*开始。其次，它们都具有相同的内存大小和表示形式，即表示地址的4或8字节。在32位架构上，指针需要4字节的内存，而在64位架构(如你的机器)上，它们需要8字节的内存。

# 间接访问内存

看看这个小程序，它执行一个函数调用，通过**按值**传递地址。这将与`increment`函数共享`main`的帧中的`count`变量

## Listing 10
```
01 package main
02
03 func main() {
04
05    // Declare variable of type int with a value of 10.
06    count := 10
07
08    // Display the "value of" and "address of" count.
09    println("count:\tValue Of[", count, "]\t\tAddr Of[", &count, "]")
10
11    // Pass the "address of" count.
12    increment(&count)
13
14    println("count:\tValue Of[", count, "]\t\tAddr Of[", &count, "]")
15 }
16
17 //go:noinline
18 func increment(inc *int) {
19
20    // Increment the "value of" count that the "pointer points to". (dereferencing)
21    *inc++
22    println("inc:\tValue Of[", inc, "]\tAddr Of[", &inc, "]\tValue Points To[", *inc, "]")
23 }
```

从最初的程序中，有三个有趣的变化。这是第12行上的第一个变化
## Listing 11
```
12    increment(&count)
```
这一次在第12行，代码不是复制和传递计数的“值”，而是传递计数的“地址”。现在你可以说，我正在与`increment`函数“共享”`count`变量。这就是`&`操作符功能：“共享”。

请理解这仍然是一个“按值传递”，唯一的区别是你传递的值是一个地址而不是整数。地址也是值; 这是正在复制并通过帧传递给函数调用者的内容。

由于正在复制和传递地址的值，因此需要在`increment`帧内设置一个变量来接收和存储这个整数的地址。这就是整型指针变量的声明在第18行出现的地方。

## Listing 12
```
18 func increment(inc *int) {
```

如果要传递`User`值的地址，则需要将变量声明为`*User`。即使所有指针变量都存储地址值，它们也不能传递任何地址，只能传递与指针类型关联的地址。这是关键，共享一个值的原因是因为接收函数需要对该值执行读写操作。你需要任何值的类型信息才能对其进行读写。编译器将确保只有与正确指针类型关联的值与该函数共享。

这是函数调用increment后栈的样子:
## Figure 5
![image.png](https://upload-images.jianshu.io/upload_images/44480-61d79b5c0754f31b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在图5中可以看到，当使用地址作为值执行“传递值”时，栈是什么样子的。`increment`函数帧的指针变量现在指向`count`变量，它位于`main`所在的帧内。

现在使用指针变量，函数可以对`main`帧内的`count`变量执行间接的读修改写操作。

## Listing 13
```
21    *inc++
```
这一次，*字符充当操作符并应用于指针变量。使用*作为运算符意味着，“指针指向的值”。指针变量允许在使用它的函数帧之外间接访问内存。有时这种间接的读或写被称为取消指针引用。`increment`函数在它的帧内仍然必须有一个指针变量，它可以直接读取来执行间接访问。

现在，在图6中，你可以看到第21行执行之后的栈是什么样子的。

## Figure 6
![image.png](https://upload-images.jianshu.io/upload_images/44480-c5bcd92c558e97a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

以下是输出数据
## Listing 14

```
count:  Value Of[ 10 ]   	   	Addr Of[ 0x10429fa4 ]
inc:    Value Of[ 0x10429fa4 ]  Addr Of[ 0x10429f98 ]   Value Points To[ 11 ]
count:  Value Of[ 11 ]   	   	Addr Of[ 0x10429fa4 ]
```

你可以看到，`inc`指针变量的“值”与计数变量的“地址”相同。这样就建立了共享关系，允许对帧之外的内存进行间接访问。当`increment`函数通过指针执行写操作时，`main`函数会在返回时看到更改。

# 指针没什么特别

指针变量并不特殊，因为它与其他变量一样都只是变量而已。它们有一个内存分配和一个值。所有的指针变量，不管它们指向的值是什么类型，大小和表示方式都是一样的。令人困惑的是`*`字符在代码中充当操作符，用于声明指针类型。

# 总结
这篇文章描述了指针背后的目的，以及栈和指针机制如何在`Go`中工作，如果你理解了这种设计的理念与机制，恭喜你，在编写简洁高效代码的的旅途中，你迈出了第一步。

总之，看到这里，你可以学到许多:

- 函数在帧边界范围内执行，帧为每个单独的函数提供单独的内存空间。
- 当调用一个函数时，在两个帧之间会产生交互。
- **按值**传递数据的好处是可读性。
- 栈很重要，因为它为每个单独的函数提供了有边界的物理内存空间。
- 活动帧以下的所有栈内存都无效，但活动帧以上的内存是有效的。
- 调用函数意味着`goroutine`需要在堆栈上开辟一段新的内存空间。
- 在每次函数调用期间，在获取帧时，该帧的堆栈内存将被清除（覆盖）。
- 指针的意义，即与函数共享一个值，以便函数可以读写该值，即使该值并不直接存在于其所在帧中。
- 对于由你或语言本身声明的每一种类型，你都可以免费获得用于共享的恭维指针类型。
- 指针变量允许在使用它的函数的帧之外间接访问内存。
- 指针变量并不特殊，因为它们和其他变量一样都是变量。它们有一个内存分配和一个值。
---
版权声明：

1. 任何个人或机构如需转载本文，无须再获得作者书面授权，但是转载者必须保留作者署名，并注明出处。

2. 作者保留对本文的修改权。他人未经作者许可，不得擅自修改，破坏作品的完整性。

3. 作者保留对本文的其他各项著作权权利。

原文阅读：
[Language Mechanics On Stacks And Pointers
](https://www.ardanlabs.com/blog/2017/05/language-mechanics-on-stacks-and-pointers.html)