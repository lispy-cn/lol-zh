.. _chapter02:

=================
第二章：闭包
=================

:Author: Doug Hoyte
:Translator: Yuqi Liu


.. _2-1-closure-oriented:

2.1 面向闭包编程
--------------------

::

  我们得到的一个结论是：在编程语言中，“对象（object）”没必要作为原语（primitive notion）；
  只需用可赋值单元和好的老式的 lambda 表达式就可以构建对象及其行为。
  —— Guy Steele 设计 Scheme 时说的话

这种 lambda 表达式有时叫做闭包，有时叫做已保存的词法环境。或者，我们更喜欢说的，let over
lambda。无论怎么称呼，掌握闭包的概念是成为专业 lisp 程序员的第一步。事实上，这项技能对于正确
使用许多现代编程语言至关重要，即使是那些没有明确包含 let 或 lambda 的语言，例如 Perl 或
Javascript。

闭包是少有的奇怪概念之一，因为它们太简单了，而与之自相矛盾的是，掌握起来很难。一旦程序员习惯了用
复杂的方法来解决问题时，用简单的方法解决同样的问题时就会感到不完整和不舒服。但是，我们很快就会看
到，对于如何组织数据和代码，闭包比对象更简单、更直接。比简单更重要的是，闭包是在构造宏时的更好的
抽象——这也是本书的主题。

我们可以使用闭包原语构建对象和类这一事实并不意味着对象系统对 lisp 程序员毫无用处。恰恰相反。事
实上，COMMON LISP 包括有史以来最强大的对象系统：CLOS，即 COMMON LISP 对象系统。虽然我对
CLOS 的灵活性和特性印象深刻，但我很少发现需要使用其更高级的特性，这要归功于可分配的值单元和好的
旧 lambda 表达式。

虽然本书的大部分内容都假定你具有合理的 lisp 技能水平，但本章试图从最基础的角度教授闭包的理论和
使用，并提供将在本书其余部分中使用的闭包通用术语。本章还检查了闭包的效率影响，并考虑了现代编译器
对它们的优化程度。


.. _2-2-environments-and-extent:

2.2 环境及拓展
-----------------

斯蒂尔所说的可分配值单元的意思是一种环境，用于存储指向数据的指针，该环境受制于所谓的无限范围。这
是一种花哨的说法，可以在未来的任何时候继续参考这样的环境。一旦分配了这个环境，它和它的引用在需要
用到时就会一直存在。想一想以下的 C 函数：

.. code-block:: c
    :linenos:

    #include <stdlib.h>

    int *environment_with_indefinite_extent(int input) {
      int *a = malloc(sizeof(int));
      *a = input;
      return a;
    }

在我们调用这个函数并接收到它返回的指针之后，可以继续无限期地引用分配的内存。在 C 中，调用函数时
会创建新环境，但 C 程序员知道在将所需的内存返回给函数外使用时 **malloc()**。


相比之下，下面的例子是有缺陷的。C 程序员认为在函数返回时会自动收集 **a**，因为环境是在堆栈上分配的。换句话说，从 lisp 程序员的角度来看， **a** 被分配了临时范围。

.. code-block:: c
    :linenos:

    int *environment_with_temporary_extent(int input) {
      int a = input;
      return &a;
    }

C 环境和 lisp 环境之间的区别在于，除非在 lisp 中显示定义，否则默认使用无限范围。换句话说，
lisp 总是像上面那样调用 **malloc()** 。可以说，这本质上比使用临时范围效率低，但收益几乎总是
超过边际性能成本。更重要的是，lisp 通常可以确定何时能安全地在堆栈上分配数据，并且会自动执行此操
作。甚至可以使用声明来告诉 lisp 显式地执行此操作。我们将在 :ref:`chapter07` 中更详细地讨论
如何声明。
​
但是由于 lisp 的动态特性，它不像 C 那样具有显式的指针值或类型。如果是 C 程序员习惯于转换指针
和值来指示类型，这可能会令人困惑。Lisp 对这一切的看法略有不同。在 lisp 中，一个方便的口头禅如
下：

::

  Variables don't have types. Only values have types.（变量无类型，值有类型）

所以，必须返回某个结构保存指针。在 lisp 中有许多可以存储指针的数据结构。lisp 程序员最喜欢的一
种结构是种简单的结构：cons。每个 cons 结构格恰好包含两个指针，亲切地称为 **car** 和
**cdr** 。当**environment-with-indefinite-extent** 被调用时，将返回一个 cons 结构，其
中 **car** 指向输入的内容，而 **cdr** 指向 **nil** 。而且，最重要的是，这个 cons 结构（以
及指向输入的指针）具有无限的范围，因此可以在需要时继续引用它：

.. code-block:: none
    :linenos:

    (defun environment-with-indefinite-extent (input)
      (cons input nil))

随着 lisp 编译技术的最新技术水平的提高，无限程度的效率劣势正在接近无关紧要。环境和范围与闭包密
切相关，本章将详细介绍它们。


.. _2-3-lexical-and-dynamic-scope:

2.3 词法作用域与动态作用域
---------------------------

变量的有效作用范围叫做作用域。现代的变成语言中比较通用的作用域是——词法作用域。当变量作用于代码
段中，那么该变量就被称为是该代码段的词法作用域的绑定。最为通用的创建编定的的关键词 ——
**let**，可以来介绍词法作用域中的变量：

.. code-block:: none
    :linenos:

    * (let ((x 2))
        x)

    2

上面 **let** 中的变量 **x** 就是通过词法作用域来访问的。同样的，由 **lambda** 和
**defun** 定义的函数中的参数也是函数中的词法绑定。词法变量指的是只能从词法作用域中对其进行访
问，比如说上面的 **let** 语句。由于词法作用域是如此直接的来限制作用域中变量的访问，所以词法作
用域看起来似乎是作用域的唯一选择。那这还有其他作用域的容身之处吗？

尽管不确定范围和词法范围的组合非常有用，但直到现在它还没有在主流编程语言中得到很好的应用。词法作
用域第一次应用在由Steve Russell 设计的 Lisp 1.5，随后演化成了 Algol60、Scheme和 Common
Lisp。由于有着悠久而丰富的历史，词法作用域的许多优点渐渐被其他的语言吸收。

即便是类 C 的语言的作用域有局限性，C 程序员仍需要进行跨平台的编程。为了实现跨平台编程，C 程序
员通常会用叫作指针作用域的粗略作用域。指针作用域比较知名的有以下几点：调试困难，安全风险高以及
（某些人为的）效率低。指针作用域的原理通过定义一种特定域语言，用来控制冯诺伊曼式计算机（现在大部
分的CPU）中的寄存器和内存，然后使用这种语言访问和操作数据结构，并对运行程序的CPU发出相当直接的
命令。在没有较好的lisp编译器之前，如果需要考虑到性能的话，指针作用域是很有必要的，但如今却成了
现代编程语言的问题，而不是特性。

尽管lisp程序员很少考虑指针，但对指针作用域的理解对构造高效的lisp代码是很有有价值的。在 :ref:`7-4-pointer-scope`
中，我们将会研究在某些需要通知编译器生成特定的代码特殊情况中实现指针作用域。目前我们
只需要讨论它的机制。在C语言中，有时需要访问定义在函数之外的变量，如 **pointer_scope_test** 函数:

.. code-block:: c
    :linenos:

    #include <stdio.h>

    void pointer_scope_test() {
      int a;
      scanf("%d", &a);
    }

在上面的函数中，使用了C语言中的 **&** 操作符将本地变量 **a** 在内存中的实际地址提供给
**scanf** 函数，这样 **scanf** 就知道将输入的数据写到哪里。lisp中的词法作用域我们无法直接实
现这种操作。在lisp中，我们可能会将一个匿名函数传递给假设的 **scanf** 函数，其中 **scanf**
可以对词法变量 **a** 进行赋值，即便 **scanf** 定义在词法作用域外:

.. code-block:: none
    :linenos:

    (let (a)
      (scanf "%d" (lambda (v) (setf a v))))

词法作用域是闭包的基础。实际上，词法作用域通常被更具体的叫做词法闭包，用来区分于别的闭包，就是因
为这两者定义很相近。除特别注明外，书中所说的都是词法闭包。
​
除词法作用域外，COMMON LISP还有动态作用域。动态作用域是 lisp 方言中临时作用域和全局作用域的
组合。这是lisp 特有的作用域类型，因为它与词法作用域的语法相同，但表现方式却不一样。在COMMON
LISP 中，我们特意将动态作用域中的变量叫做特殊变量。这些特殊的变量可以用 **defvar** 定义。一
些程序员遵循用星号作为前缀和后缀的特殊变量名的惯例，比如 \*temp-special\*。这种命名方法有些
掩耳盗铃。基于 :ref:`3-7-duality-of-syntax` 中阐述的原因，本书不会采用这种掩耳盗铃的命名方法，
因此特殊变量声明如下:

.. code-block:: none
    :linenos:

    (defvar temp-special)

当这样定义时，\*temp-special\* 就是一个特殊（我们还可以通过声明一个特殊局部变量来表示变量的
特殊）的、不会初始化的值。在这种情况下的变量变量称为 unbound 。只有全局变量可以不被绑定——词法
变量总是被绑定的，因此总是有值。另一种思路是，默认情况下，所有符号都表示词汇上未绑定的变量。与词
法变量一样，我们可以使用 **\*setq\*** 或 **\*setf\*** 为全局变量赋值。有些Lisp，如
Scheme，没有动态作用域。其他的，如 EuLisp，使用不同的语法来访问词法变量和全局变量。但在
Common Lisp 中，语法是共享的。许多 lisper 认为这是一个特性。以下是给全局变量
**\*temp-special\*** 赋值:

.. code-block:: none
    :linenos:

    (setq temp-special 1)

目前看着，这个“特殊”的变量看起来不是那么的特殊。和其他的变量没差，同样是绑定了某些全局的命名空
间。但这是因为我们只对它进行了一次的绑定——默认的全局绑定。但当全局变量在新的环境中被重新绑定或
是覆盖时就变的有趣了。假设现在定义一个简单的函数，该函数返回 **temp-special** 的值：

.. code-block:: none
    :linenos:

    (defun temp-special-returner ()
      temp-special)

当这这个函数被调用时，可以用来展示 lisp 是怎样对 **temp-special** 求值的。

.. code-block:: none
    :linenos:

    * (temp-special-returner)
    1

有时，这种情况在空词法作环境（null lexical environment）中看作是对表单的求值。空词法环境显
然不包含任何词法绑定。在这里 **temp-sepcial** 变量返回的是它全局变量的值——1。但在非空词法环
境中（其中全局变量被重新赋值）对其求值时，全局变量返回的是新的值（因为当创建一个动态绑定时，并没
有创建一个词法的环境，看起来就是如此）。

.. code-block:: none
    :linenos:

    * (let ((temp-special 2))
        (temp-special-returner))

    2

以上执行结果返回的是 2，这代表着 **emp-special** 绑定的是 **let** 作用域中的值，而不是全局
定义的值。如果这还不够有趣的话，看看这段Blub伪代码，就知道全局变量如何在其他大多数传统编程语言
中无法实现的：

.. code-block:: c
    :linenos:

    int global_var = 0;

    function whatever() {
      int global_var = 1;
      do_stuff_that_uses_global_var();
    }

    function do_stuff_that_uses_global_var() {
      // global_var is 0
    }

虽然词法绑定的内存位置或寄存器分配在编译时是已知的（这也是词法作用域有时也被叫做静态作用域的原
因），但在某种意义上，全局变量绑定是在运行时确定的。由于一个巧妙的技巧，全局变量并不是看起来那么
低效。全局变量实际上总是指向内存中的相同位置。在用 **let** 绑定全局变量时，实际上是在编译代
码，这些代码将存储变量的副本，用一个新值覆盖内存位置，在 **let** 主体中对表单求值，最后从副本
中恢复原始值。

全局变量总是与命名的符号相关联。全局变量指向的内存中的位置被叫做符号的“符号值”单元格。这与词法
变量形成了直接的对比。词法变量仅在编译时用符号表示。因为词法变量只能从其绑定的词法范围内访问，所
以编译器甚至没有理由记住用来引用词法变量的符号，因此编译器会在编译后的代码中删除这些符号。我们将
在 :ref:`6-7-pandoric-macros` 中来详细的证明这一点。

COMMON LISP的确是提供了动态范围的宝贵特性，但是词法变量是最常见的。动态作用域曾经是lisp中定义
的特性，但在 Common Lisp 之后，动态作用域几乎完全被词法作用域取代。因为词法作用域支持词法闭包
(稍后我们将对此进行讨论)以及更有效的编译器优化，所以动态作用域的取代通常被视为一件好事。然而，
COMMON LISP的设计者给我们留下了一个非常透明的窗口，让我们可以看到动态范围的世界，现在我们知道
动态范围的真正含义：特殊。


.. _2-4-let-it-be-lambda:

2.4 Let It Be Lambda
-----------------------------

**Let** 是 Lisp 中特殊的表单，用于创建并初始化相应表单名称(绑定)的环境。这些名称对于
**let** 主体中有效是，且该求值是连续的，并返回最终表单的结果。虽然 **let** 做了什么很明确，
但具体是怎么做的却没指明。**let** 结果与过程是分离的。在某种程度上， **let** 需要提供一个数
据结构来存储指向值的指针。

正如前面所介绍的那样，cons 的结构适用于存储指针，这点毋庸置疑。同时，还有很多其他的结构可以用来
存储指针。在 lisp 中存储指针的最佳方法之一就是使用 **let** 结构。在 **let** 结构中，你只需
要给指针命名就好，之后 lisp 会计算出怎样最好地存储指针。有时，可以通过声明的形式提供额外的信
息，用来提高编译器的效率，如下代码所示：

.. code-block:: none
    :linenos:

    (defun register-allocated-fixnum ()
      (declare (optimize (speed 3) (safety 0)))
      (let ((acc 0))
        (loop for i from 1 to 100 do
          (incf (the fixnum acc)
                (the fixnum i)))
        acc))

例如，在 **register-allocated-fixnum** 中，我们向编译器提供了一些提示，让其可以高效地将
1 到 100 的整数相加。编译后，此函数将在寄存器中分配数据，完全不需要指针。尽管我们似乎已经要求
lisp 创建一个无限范围的环境来保存 **acc** 和 **i**，但 lisp 编译器将能够通过仅将值存储在
CPU 寄存器中来优化此函数。结果可能是以下的机器代码：

.. code-block:: none
    :linenos:

    ; 090CEB52:       31C9             XOR ECX, ECX
    ;       54:       B804000000       MOV EAX, 4
    ;       59:       EB05             JMP L1
    ;       5B: L0:   01C1             ADD ECX, EAX
    ;       5D:       83C004           ADD EAX, 4
    ;       60: L1:   3D90010000       CMP EAX, 400
    ;       65:       7EF4             JLE L0

注意，地址 **4** 中存储的是 **1**，**400** 中存储的是 **100**，因为在编译后的代码中，
**fixnums** 移动了两位。这与标记有关，这是一种假装某些东西是指针但实际上在其中存储数据的方
法。lisp编译器的标记方案有一个很好的好处，即不需要发生移位来索引字对齐内[DESIGN-OF-CMUCL]。
我们将在 :ref:`chapter07` 中深入了解 lisp 编译器。

但是，如果 lisp 确定之后可能要引用此环境，则它必须使用比寄存器更短暂的东西。在环境中存储指针的
常见结构是数组。如果每个环境都有一个数组，并且包含在该环境中的所有变量引用都只是对该数组的引用，
那么就有一个具有潜在无限范围的高效环境。

如上所述，**let** 将返回其主体中最后一个条语句执行的结果。这对很多 lisp 特殊形式和宏来说很常
见，因此这种模式通常被称为隐式 **progn** ，因为 **progn** 特殊结构设计为除了此之外什么都不
做。有时让 **let** 结构返回最有价值的东西是一个匿名函数，它利用了 **let** 结构提供的词法环
境。为了在 lisp 中创建这些函数，就要用到 *lambda*。

Lambda 是一个简单的概念，但因其灵活性和重要性而令人生畏。lisp 和 scheme 的 lambda 源于
Alonzo Church 的逻辑系统，但已经发展并适应城自己的 lisp 规范。Lambda 是一种简洁的方法，可
以重复地将临时名称（绑定）分配给特定词汇上下文的值，且是 lisp 函数概念的基础。lisp 函数与
Church 心目中的数学函数描述非常不同。这是因为 lambda 在几代 lispers 的手中已经发展成为一种
强大而实用的工具，扩展它已经远远超出了早期逻辑学家所能预见的范围。

尽管 lisp 程序员对 lambda 表示敬意，但该符号本身并没有什么特别之处。正如我们将看到的，
lambda 只是表达这种变量命名的许多可能方式之一。特别是，我们将看到宏可以以其他编程语言实际上不
可能的方式自定义变量的重命名。但是在探索了这一点之后，我们将回到 lambda 并发现它非常接近于表达
这种命名的最佳符号。这绝非偶然。Church 在我们的现代编程环境中可能看起来过时且无关紧要，但他确
实在做某事。他的数学符号，以及它在几代 lisp 专业人士手中的众多改进，已经发展成为一种灵活、通用
的工具。

Lambda 非常有用，就像许多 lisp 的功能一样，大多数现代语言都开始将 lisp 的想法导入到自己的系
统中。一些语言设计者觉得 lambda 太冗长，而使用 **fn** 或其他一些缩写。另一方面，有些人认为
lambda 是个很基本的概念，所以用叫短的名字来掩盖它是异端的邪说。在本书中，虽然我们将描述和探索
lambda 的许多变体，但我们很高兴地称它为 lambda，就像我们之前的几代 lisp 程序员一样。

但是 lisp 的 lambda 是什么？ 首先，与 lisp 中的所有名称一样，lambda 是个符号。我们可以引用
它，比较它，将它存储在列表中。Lambda 仅在作为列表的第一个元素出现时才具有特殊含义。当它出现在
那里时，该列表被称为 lambda 形式或函数指示符。但是这种结构不是函数。这种结构是一个列表数据结
构，可以用 **function** 关键词将其转换为函数：

.. code-block:: console
    :linenos:

    * (function '(lambda (x) (+ 1 x)))

    #<Interpreted Function>

COMMON LISP 使用 **#'** （井号加单引号）读取宏为我们提供了个方便的简写。为了达到同样的效果，
可以使用这个简写，而不是像上面那样编写函数：

.. code-block:: console
    :linenos:

    * #'(lambda (x) (+ 1 x))

    #<Interpreted Function>

作为一个很方便的特性，lambda 也被定义为一个宏，它扩展为对上述特殊形式的函数的调用。COMMON
LISP ANSI 标准需要 [ANSI-CL-ISO-COMPATIBILITY] 一个 **lambda** 宏，定义如下：

.. code-block:: none
    :linenos:

    (defmacro lambda (&whole form &rest body)
      (declare (ignore body))
      **#',form)

先忽略定义的具体代码。这个宏只是一种将函数特殊形式自动应用于函数指示符的简单方法。这个宏允许我们
执行函数指示符以创建函数，因为它们被扩展为 **#'** 形式：

.. code-block:: console
    :linenos:

    * (lambda (x) (+ 1 x))

    #<Interpreted Function>

因为 lambda 宏，几乎没有理由在 lambda 结构前面加上 **#'** 。因为本书不支持 ANSI 之前的
COMMON LISP 环境，所以显而易见是不向下兼容的。但是对象的格式呢？ Paul Graham 在 ANSI
COMMON LISP[GRAHAM-ANSI-CL] 中认为这个宏，连同它的简洁优势，“充其量是一种似是而非的优雅”。
格雷厄姆的反对意见似乎是，由于仍然需要对符号引用的函数进行 **#'** ，因此系统似乎是不对称的。但
是，我相信非 **#'** lambda 结构实际上是一种风格上的改进，因为它突出了第二个命名空间规范中存在
的不对称性。对符号使用 **#'** 是为了引用第二个命名空间，而由 lambda 形式创建的函数当然是无名
的。

甚至无需调用 **lambda** 宏，就可以用 lambda 结构作为函数调用中的第一个参数。就像在这个位置找
到一个符号且 lisp 假设我们正在引用该符号的 **symbol-function** 结构一样，如果找到 lambda
结构，则假设它表示一个匿名函数：

.. code-block:: none
    :linenos:

    * ((lambda (x) (+ 1 x)) 2)

    3

但注意，正如不能调用函数来动态返回要在常规函数调用中使用的符号一样，也不能调用函数以在函数位置返
回 lambda 形式。对于这两种情况，需要使用 **funcall** 或 **apply** 。

lambda 表达式在很大程度上与 C 和其他语言中的函数无关，它的好处是 lisp 编译器通常可以将它们完
全优化为不存在。例如，尽管 **compiler-test** 看起来像是对 **2** 应用了一个递增函数并返回结
果，但正常的编译器会知道该函数总是返回 **3** ，且会直接返回该数字，该过程中不会调用函数。这叫
做 *lambda 折叠*：

.. code-block:: none
    :linenos:

    (defun compiler-test ()
      (funcall
        (lambda (x) (+ 1 x))
        2))

一个重要的有效观察是编译的 lambda 结构是个常量结构。这意味着在程序编译后，对该函数的所有引用都
只是指向一块机器代码的指针。该指针可以从函数返回并嵌入到新环境中，所有这些都没有函数创建开销。编
译程序时吸收了开销。换句话说，返回另一个函数的函数将只是个指针返回函数常量时间：

.. code-block:: none
    :linenos:

    (defun lambda-returner ()
      (lambda (x) (+ 1 x)))

这与 **let** 结构形成鲜明对比，后者旨在在运行时创建一个新环境，因此通常不是个恒定的操作，因为
词法闭包隐含的垃圾收集开销是无限的。

.. code-block:: none
    :linenos:

    (defun let-over-lambda-returner ()
      (let ((y 1))
        (lambda (x)
          (incf y x))))

每次调用 **let-over-lambda-returner** 时，它必须创建一个新环境，将指向 lambda 结构表示的
代码的常量指针嵌入到这个新环境中，然后返回结果闭包。可以用 **time** 来看看这个环境有多小：

.. code-block:: console
    :linenos:

    * (progn
        (compile 'let-over-lambda-returner)
        (time (let-over-lambda-returner)))

    ; Evaluation took:
    ;   ...
    ;   24 bytes consed.
    ;
    #<Closure Over Function>

如果你尝试在闭包上调用 **compile** ，将得到一个错误消息，指出无法编译在非空词法环境中定义的函
数 [CLTL2-P677]。你不能编译闭包，只能编译创建闭包的函数。当编译一个创建闭包的函数时，它创建的
闭包也将被编译[ON-LISP-P25]。

使用包含上述 lambda 的 let 非常重要，我们将在本章的剩余部分中讨论它的模式和变体。


.. _2-5-let-over-lambda:

2.5 Let Over Lambda
-----------------------

*Let over lambda* 是词法闭包的昵称。Let over lambda 比大多数术语更清晰地反映用于创建闭包
的 lisp 代码。在 let over lambda 场景中， **let** 语句返回的最后一个结构是 **lambda**
表达式。它看起来就像 **let** 坐在 **lambda** 之上：

.. code-block:: console
    :linenos:

    * (let ((x 0))
        (lambda () x))

    #<Interpreted Function>

回顾一下，**let** 结构返回计算其主体内最后一个结构的结果，这就是为什么计算这个 let over
lambda 结构会产生一个函数。但是，**let** 中的最后一个结构有特别之处。它是个以 **x** 作为自
由变量的 **lambda** 结构。Lisp 很聪明，可以确定 **x** 应该为这个函数引用什么：来自由
**let** 结构创建的周围词法环境的 **x** 。而且，因为在 lisp 中，默认情况下一切都是无限的，只
要需要，该功能就可以使用该环境。

因此，词法作用域是一种工具，用于准确指定对变量的引用在何处有效，以及引用所指的确切内容。一个简单
的闭包示例是一个计数器，一个在环境中存储整数并在每次调用时递增并返回该值的闭包。下面是带有 let
over lambda 经典实现：

.. code-block:: none
    :linenos:

    (let ((counter 0))
      (lambda () (incf counter)))

这个闭包在第一次被调用时返回 1，随后返回 2，以此类推。考虑闭包的一种方式是它们是具有状态的函
数。这些函数不是数学函数，而是程序，每个都有自己的一点记忆。有时将代码和数据捆绑在一起的数据结构
称为对象。对象是过程和一些相关状态的集合。由于对象与闭包密切相关，因此它们通常可以被认为是一回
事。闭包就像个只有一个方法（ **funcall** ）的对象。一个对象就像一个闭包，你可以通过多种方式调
用它。

尽管闭包总是单个函数及其封闭环境，但对象系统的多个方法、内部类和静态变量都有其对应的闭包。模拟多
个方法的一种可能方法是简单地从同一词法范围内返回多个 **lambda** ：

.. code-block:: none
    :linenos:

    (let ((counter 0))
      (values
        (lambda () (incf counter))
        (lambda () (decf counter))))

这个 *let over two lambdas* 结构将返回两个函数，这两个函数都访问相同的封闭计数器变量。第一
个增加它，第二个减少它。还有许多其他方法可以实现这一点。其中一个便是 :ref:`5-7-dlambda` 中进行了讨
论的 **dlambda** 。基于之后介绍的原因，本书中的代码将使用闭包而不是对象来构造所有数据。提示：
与宏有关。


.. _2-6-lambda-over-letoverlambda:

2.6 Lambda Over Let Over Lambda
--------------------------------

在一些对象系统中，对象、具有关联状态的过程集合和类（用于创建对象的数据结构）之间存在明显区别。闭
包不存在这种区别。我们看到了可以执行以创建闭包的结构示例，其中大多数遵循 lambda 模式，但是程序
如何根据需要创建这些对象？

答案非常简单。如果我们可以在 REPL 中执行它们，也就可以在函数中执行它们。如果创建一个函数，其唯
一目的是执行一个 let over lambda 并返回结果呢？ 因为我们使用 **lambda** 来表示函数，所以它
看起来像这样：

.. code-block:: none
    :linenos:

    (lambda ()
      (let ((counter 0))
        (lambda () (incf counter))))

当调用 *lambda over let over lambda* 时，将创建并返回一个包含计数器绑定的新闭包。记住，
lambda 表达式是常量：仅仅是指向机器代码的指针。这个表达式是一段简单的代码，它创建新的环境来关
闭内部 **lambda** 表达式（它本身是一个常量，编译的形式），就像我们在 REPL 中所做的那样。

对于对象系统，创建对象的一段代码称为类。但是 lambda over let over lambda 与许多语言的类略
有不同。虽然大多数语言都需要命名类，但这种模式完全避免了命名。Lambda over let over lambda
形式可以称为匿名类。

尽管匿名类通常很有用，但我们通常会命名类。给它们命名的最简单方法是认识到这些类是常规函数。我们通
常如何命名函数？ 当然是用 **defun** 关键字了。命名后，上面的匿名类就变成了：

.. code-block:: none
    :linenos:

    (defun counter-class ()
      (let ((counter 0))
        (lambda () (incf counter))))

第一个 **lambda** 去哪儿了？ **defun** 在其主体中的结构周围提供了个隐式 lambda。当用
**defun** 编写常规函数时，实际上还是 lambda 结构，但这隐藏在 **defun** 语法之下。

不幸的是，大多数 lisp 编程书籍都没有提供闭包使用的实际示例，这给读者留下了一种不准确的印象，即
闭包只适用于像计数器这样的玩具示例。这与事实相去甚远。闭包是 lisp 的基石。环境，在这些环境中定
义的函数，以及像 **defun** 这样方便使用的宏，都是建模任何问题所需要的。本书旨在阻止那些习惯于
基于对象的语言的 lisp 程序员开始按照他们的经验去接触如 CLOS 这样的系统。虽然 CLOS 的确为专业
的 lisp 程序员提供一些东西，但在只需用 lambda 时就不要用 CLOS。

.. code-block:: none
    :linenos:

    (defun block-scanner (trigger-string)
      (let* ((trig (coerce trigger-string 'list))
            (curr trig))
        (lambda (data-string)
          (let ((data (coerce data-string 'list)))
            (dolist (c data)
              (if curr
                (setq curr
                      (if (char= (car curr) c)
                        (cdr curr) ; next char
                        trig))))   ; start over
            (not curr))))) ; return t if found

为了鼓励使用闭包，给出了一个现实的例子：**block-scanner** 。**block-scanner** 解决的问题
是，对一些结构的数据传输，数据以大小不定的组（块）形式传递。这些大小通常对底层系统很方便，但对应
用层程序员却不方便，通常由操作系统缓冲区、硬盘驱动器块或网络数据包等因素决定。扫描特定序列的数据
流需要的不仅仅是扫描每个块，因为它带有常规的无状态过程。需要在每个块的扫描之间保持状态，因为我们
正在扫描的序列可能会被分成两个（或更多）块。

在现代语言中实现这种存储状态的最直接、最自然的方法是闭包。基于闭包的块扫描器的初版是给出的
**block-scanner** 。像所有 lisp 开发一样，创建闭包是一个迭代过程。我们可能从
**block-scanner** 中给出的代码开始，并决定通过避免将字符串强制转换为列表来提高其效率，或者可
能通过计算序列出现的次数来改进收集的信息。

尽管 **block-scanner** 是个等待改进的初始实现，但它仍然是使用 lambda over let over
lambda 的很好演示。下面是它使用，假装是某种通信磁带，注意特定的单词，*jihad*：

.. code-block:: console
    :linenos:

    * (defvar scanner
        (block-scanner "jihad"))

    SCANNER
    * (funcall scanner "We will start ")

    NIL
    # (funcall scanner "the ji")

    NIL
    * (funcall scanner "had tomorrow.")

    T


.. _2-7-letoverlambda-over-letoverlambda:

2.7 Let Over Lambda Over Let Over Lambda
--------------------------------------------

对象系统的用户将他们希望在某个类的所有对象之间共享的值存储到所谓的类变量或静态变量中。在 lisp
中，闭包之间共享状态的概念由环境处理，就像闭包本身存储状态一样。由于环境可以无限访问，只要它仍然
可以引用它，就能保证它在需要时可用。

如果想为所有计数器维护一个全局方向， **up** 向上递增， **down** 向下递减，那么可能想要使用
let over lambda over let over lambda 模式：

.. code-block:: none
    :linenos:

    (let ((direction 'up))
      (defun toggle-counter-direction ()
        (setq direction
              (if (eq direction 'up)
                'down
                'up)))

      (defun counter-class ()
        (let ((counter 0))
          (lambda ()
            (if (eq direction 'up)
              (incf counter)
              (decf counter))))))

在上面的例子中，我们扩展了上节的 **counter-class** 。现在调用使用 **counter-class** 创建
的闭包将增加其计数器绑定或减少它，这取决于所有计数器之间共享的方向绑定的值。注意，这里还创建一个
名为 **toggle-counter-direction** 的函数来利用方向环境中的另一个 lambda，该函数更改所有
计数器的当前方向。

虽然 **let** 和 **lambda** 的这种组合非常有用，以至于其他语言以类或静态变量的形式实现它，但
还有其他的 **let** 和 **lambda** 组合允许以没有直接类似物的方式在对象系统中 构造代码和状态
。对象系统是 let 和 lambda 组合子集的形式化，有时带有类似继承的噱头。正因为如此，lisp 程序员
通常不会考虑类和对象。Let 和 lambda 是基本的； 对象和类是衍生物。正如斯蒂尔所说，“对象”不必是
编程语言中的原始概念。一旦可分配的值单元和好的旧 lambda 表达式可用，对象系统充其量只是偶尔有用
的抽象，最坏的情况是特殊情况和冗余。
