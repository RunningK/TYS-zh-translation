第十四章 不确定性
====================

麦卡锡的非确定运算符`amb`几乎和Lisp一样古老，尽管现在它已经从Lisp中消失了。`amb`接受一个或多个表达式，并在它们中进行一次“非确定”（或者叫“模糊”）选择，这个选择会让程序趋向于有意义。现在我们来探索一下Scheme内置的`amb`过程，该过程会对模糊的选项进行深度优先选择，并使用Scheme的控制操作符`call/cc`来回溯其他的选项。结果是一个优雅的回溯机制，该机制可用于在Scheme中对问题空间进行搜索而不需要另一种扩展了的语言。这种内嵌的恢复续延的机制可以用来实现Prolog风格的逻辑语言，但是更方便（sparer），因为这个操作符更像是Scheme的一个布尔运算符，使用时不需要特殊的上下文（context），而且也不依赖语言学的一些基础元素如逻辑变量和归纳法（unification）。

## 14.1 对amb的描述

最早的Scheme的教程SICP对`amb`进行了易于理解的描述，同时还给出了许多例子。说得直白一些，`amb`接受零个或更多表达式并“不确定”的返回其中“一个”的值。因此：

```scheme
(amb 1 2)
```

的结果可能为1或2。

不带参数调用`amb`则不会有返回值，而且应该会出错。因此：

```scheme
(amb)
-->ERROR!!! amb tree exhausted
```

（我们后面再讨论这个错误信息。）

特别的，如果它的至少一个外层表达式收敛（converges）此时需要`amb`返回一个值，那么就不会出错，因此：
```scheme
(amb 1 (amb))
```
而且：

都返回`1`。

很明显，`amb`不能简单的等同于它的第一个子表达式，因为它必须返回一个“非错误”的值，如果有这种可能的话。然而，仅仅这样还不够：为使程序收敛的选择比单纯选择`amb`的子表达式要更加严格。`amb`应该返回让“整个”程序收敛的值。在这个意义上，`amb`是一个“神”一般的运算符。

比如：
```scheme
(amb #f #t)
```

可以返回`#f`或`#t`，但是在程序：

```scheme
(if (amb #f #t)
    1
    (amb))
```

中，第一个`amb`表达式必须返回`#t`，如果返回`#f`，那就会执行`else`分支，这会导致整个程序挂掉。

## 14.2 用Scheme实现amb

在我们的`amb`实现中，我们令`amb`的子表达式从左向右。也就是说，我们先选择第一个子表达式，如果不论怎样它都失败，那再选择第二个，如此等等。在回溯到前一个`amb`之前，程序控制流中后面出现的`amb`也被搜索以查看所有的可能性。换句话说，我们对`amb`的选择树进行了一个深度优先搜索，当我们碰到失败的情况时，我们就回溯到最近的节点来尝试其他的选择。（这叫做按时间顺序的回溯。）

我们首先定义一个机制来处理基本的错误的续延：
```scheme
(define amb-fail '*)

(define initialize-amb-fail
  (lambda ()
    (set! amb-fail
      (lambda ()
        (error "amb tree exhausted")))))

(initialize-amb-fail)
```

当`amb`出错时，它调用绑定到`amb-fail`的续延。这个续延是在所有`amb`的选择树都被尝试过并且失败的情况下调用的。


我们把`amb`定义为一个宏，接受任意数量的参数。

```scheme
(define-macro amb
  (lambda alts...
    `(let ((+prev-amb-fail amb-fail))
       (call/cc
        (lambda (+sk)

          ,@(map (lambda (alt)
                   `(call/cc
                     (lambda (+fk)
                       (set! amb-fail
                         (lambda ()
                           (set! amb-fail +prev-amb-fail)
                           (+fk 'fail)))
                       (+sk ,alt))))
                 alts...)

          (+prev-amb-fail))))))
```

对`amb`的调用被首先存储到`+prev-amb-fail`中，`amb-fail`的值是此时的入口。这是因为`amb-fail`变量会被随着对可能选项的遍历被设置为不同的失败续延。

我们然后捕获`amb`的入口续延`+sk`，这样当求出一个“非失败”的值时，它可以马上退出`amb`。

每个序列中的选择`alt`都被尝试（Scheme中隐式的`begin`序列）。

首先，我们捕获当前续延`+fk`，把它包在一个过程中并把该过程赋给`amb-fail`。接着替换物被求值`(+sk alt)`。如果`alt`的求值没有失败，那么把它的返回值作为参数给续延`+sk`，这样马上就退出了`amb`的调用。如果`alt`失败了，就调用`amb-fail`。`amb-fail`做的第一件事是重新设置`amb-fail`为之前入口时的值。它接下来调用失败续延`+fk`，这个续延会尝试下个可能的选择（如果存在的话）。

如果所有选择都失败了，`amb`入口的`amb-fail`（我们之前把它存放在`+prev-amb-fail`中）会被调用。

## 14.3 在Scheme中使用amb

选择一个1到10之间的数字，我们可以这样写：
```scheme
(amb 1 2 3 4 5 6 7 8 9 10)
```
毫无疑问这个程序会返回1（根据我们之前实现的策略），但这个与它的上下文有关，它完全可能返回给定的任何数字。

过程`number-between`是一种生成给定`lo`到`hi`（包括`lo`和`hi`在内）之间数字的抽象方法：
```scheme
(define number-between
  (lambda (lo hi)
    (let loop ((i lo))
      (if (> i hi) (amb)
          (amb i (loop (+ i 1)))))))
```

因此`(number-between 1 6)`会首先生成1。如果失败了，继续循环，生成2。如果还是失败，我们就得到3，这样一直到6。6以后，`loop`以参数7被调用，这比6要大，调用`(amb)`。这会产生一个最终的错误（回忆之前我们所说的，单独的`(amb)`肯定会出现错误）这时，这个包含`(number-between 1 6)`的程序会按时间顺序依次回溯之前的`amb`调用，用另一种方式来满足这个调用。

`(amb)`一定失败的特点可以用于程序的 _断言_ 中。
```scheme
(define assert
  (lambda (pred)
    (if (not pred) (amb))))
```

调用`(assert pred)`确保了`pred`为真，否则它会让当前的`amb`选择点失败。

下面的程序用`assert`来生成一个小于等于其参数`hi`的素数：
```scheme
(define gen-prime
  (lambda (hi)
    (let ((i (number-between 2 hi)))
      (assert (prime? i))
      i)))
```

这看起来也太简单了，只是当不论以任何数字（如20）调用这个过程，它永远会给出第一个解：2。

我们当然希望得到所有的解，而不是只有第一个。这种情况下，我们会希望得到所有比20小的素数。一种方法是在该过程输出了第一个解后，显式地调用失败续延。因此：
```scheme
(amb)
=> 3
```

这样又会产生另一个失败续延，我们还可以继续调用它来得到另一个解。
```scheme
(amb)
=> 5
```

这种方式的问题是程序首先在Scheme的命令提示符后面被调用，并且在Scheme的命令行上调用`(amb)`也可以得到成功的解。实际上，我们正在使用不同的程序（我们无法预计到底有多少！），并把信息从前一个传递到下一个。相反的，我们希望可以在任意上下文中调用某种形式然后返回这些解。为此我们定义了`bag-of`宏，该宏返回其参数的所有成功实例。（如果参数永远不能成功，就返回空列表）因此我们可以这样写：
```scheme
(bag-of
  (gen-prime 20))
```

这样会返回：
```scheme
(2 3 5 7 11 13 17 19)
```
宏`bag-of`定义如下：
```scheme
(define-macro bag-of
  (lambda (e)
    `(let ((+prev-amb-fail amb-fail)
           (+results '()))
       (if (call/cc
            (lambda (+k)
              (set! amb-fail (lambda () (+k #f)))
              (let ((+v ,e))
                (set! +results (cons +v +results))
                (+k #t))))
           (amb-fail))
       (set! amb-fail +prev-amb-fail)
       (reverse! +results))))
```

`bag-of`首先保存它的入口到`amb-fail`。它重新定义了`amb-fail`为一个在`if`测试中创建的本地续延。在这个测试中，`bag-of`的参数`e`被求值，如果成功，它的结果被收集到一个叫`+results`的列表，并且以`#t`为参数调用本地续延。这会让`if`测试成功，导致`e`会在它的下一个回溯点被重新尝试。`e`的其他结果也通过这种方法获得并放进`+results`里。

最后，当`e`失败时，它会调用基本的`amb-fail`，即以`#f`为参数调用本地续延。这就把控制从`if`中转移出来。我们把`amb-fail`恢复到它上一个入口的值，并返回`+results`。（过程`reverse!`只是用来把结果以他们生成的顺序展现出来）

## 14.4 逻辑谜题

在解决逻辑谜题时，这种深度优先搜索与回溯相结合的方法的强大才能明显体现出来。这些问题用过程式的方式非常难以解决，但是可以用`amb`简洁、直截了当的解决，而且不会减少解决问题的魅力。

### 14.4.1 Kalotan谜题

Kalotan是一个奇特的部落。这个部落里所有男人都总是讲真话。所有的女人从来不会连续2句讲真话，也不会连续2句都讲假话。

一个哲学家（Worf）开始研究这些人。Worf不懂Kalotan的语言。一天他碰到一对Kalotan夫妻和他们的孩子Kibi。Worf问Kibi：“你是男孩吗？”Kibi用Kalotan语回答，Worf没听懂。

Wrof又问孩子的父母（他们都会说英语），其中一个人说：“Kibi说：‘我是个男孩。’”，另外一个人说：“Kibi是个女孩，Kibi撒谎了”。

请问这三个Kalotan人的性别。

解决的方法包括引进一堆变量，给它们赋上各种可能的值，把所有情况列举为一系列`assert`表达式。

变量：`parent1`,`parent2`,`kibi`分别是父母（按照说话的顺序）和Kibi的性别。`kibi-self-desc`是Kibi用Kalotan语说的自己的性别。`kibi-lied?`表示Kibi是否说谎。

```scheme
(define solve-kalotan-puzzle
  (lambda ()
    (let ((parent1 (amb 'm 'f))
          (parent2 (amb 'm 'f))
          (kibi (amb 'm 'f))
          (kibi-self-desc (amb 'm 'f))
          (kibi-lied? (amb #t #f)))
      (assert
       (distinct? (list parent1 parent2)))
      (assert
       (if (eqv? kibi 'm)
           (not kibi-lied?)))
      (assert
       (if kibi-lied?
           (xor
            (and (eqv? kibi-self-desc 'm)
                 (eqv? kibi 'f))
            (and (eqv? kibi-self-desc 'f)
                 (eqv? kibi 'm)))))
      (assert
       (if (not kibi-lied?)
           (xor
            (and (eqv? kibi-self-desc 'm)
                 (eqv? kibi 'm))
            (and (eqv? kibi-self-desc 'f)
                 (eqv? kibi 'f)))))
      (assert
       (if (eqv? parent1 'm)
           (and
            (eqv? kibi-self-desc 'm)
            (xor
             (and (eqv? kibi 'f)
                  (eqv? kibi-lied? #f))
             (and (eqv? kibi 'm)
                  (eqv? kibi-lied? #t))))))
      (assert
       (if (eqv? parent1 'f)
           (and
            (eqv? kibi 'f)
            (eqv? kibi-lied? #t))))
      (list parent1 parent2 kibi))))
```

对于辅助过程的一些说明：`distinct?`过程返回`true`，如果其参数列表里所有参数都是不同的，否则返回`false`。过程`xor`只有当它的两个参数一个真一个假时才返回`true`，否则返回`false`。

输入`(solve-kalotan-puzzle)`会解决这个谜题。

### 14.4.2 地图着色

人们很早以前就知道（但知道1976年才证明）至少用四种颜色就可以给地球的地图着色，也就是说给所有国家着色并保证相邻的国家的颜色是不同的。为了验证确实是这样的，我们编写下面的程序，并指出非确定性编程是如何为之提供便利的。

下面的这段程序解决了西欧的地图着色问题。这个问题和其用Prolog语言的解法在《the Art of Prolog》中给出。（如果你能比较我们与那本书里的解法应该很有益处）

过程`choose-color`非确定的返回四种颜色之一：

```scheme
(define choose-color
  (lambda ()
    (amb 'red 'yellow 'blue 'white)))
```

在我们的解法中，我们为每个国家建立了一个数据结构。该结构是一个三元素的列表：第一个元素表示国家名，第二个元素是颜色，第三个元素是它相邻国家的颜色。注意我们用国家的首字母作为颜色的变量，即比利时（Belgium）的列表是`(list 'belgium b (list f h l g))`，因为——按照这个问题列表——比利时的邻国是法国(France)，荷兰(Holland)，卢森堡(Luxembourg)，德国(Germany)。

一旦我们给每个国家创建了列表，我们 *仅仅* 需要陈述他们应该满足的条件，即每个国家不能与邻国有相同的颜色。换句话说，对每个国家的列表，第二个元素的值应该不在第三个元素（列表）中。

```scheme
(define color-europe
  (lambda ()

    ;choose colors for each country
    (let ((p (choose-color)) ;Portugal
          (e (choose-color)) ;Spain
          (f (choose-color)) ;France
          (b (choose-color)) ;Belgium
          (h (choose-color)) ;Holland
          (g (choose-color)) ;Germany
          (l (choose-color)) ;Luxemb
          (i (choose-color)) ;Italy
          (s (choose-color)) ;Switz
          (a (choose-color)) ;Austria
          )

      ;construct the adjacency list for
      ;each country: the 1st element is
      ;the name of the country; the 2nd
      ;element is its color; the 3rd
      ;element is the list of its
      ;neighbors' colors
      (let ((portugal
             (list 'portugal p
                   (list e)))
            (spain
             (list 'spain e
                   (list f p)))
            (france
             (list 'france f
                   (list e i s b g l)))
            (belgium
             (list 'belgium b
                   (list f h l g)))
            (holland
             (list 'holland h
                   (list b g)))
            (germany
             (list 'germany g
                   (list f a s h b l)))
            (luxembourg
             (list 'luxembourg l
                   (list f b g)))
            (italy
             (list 'italy i
                   (list f a s)))
            (switzerland
             (list 'switzerland s
                   (list f i a g)))
            (austria
             (list 'austria a
                   (list i s g))))
        (let ((countries
               (list portugal spain
                     france belgium
                     holland germany
                     luxembourg
                     italy switzerland
                     austria)))

          ;the color of a country
          ;should not be the color of
          ;any of its neighbors
          (for-each
           (lambda (c)
             (assert
              (not (memq (cadr c)
                         (caddr c)))))
           countries)

          ;output the color
          ;assignment
          (for-each
           (lambda (c)
             (display (car c))
             (display " ")
             (display (cadr c))
             (newline))
           countries))))))
```

输入`(color-europe)`来得到一个颜色-国家对应表。

----------------------------
1. SICP把这个过程命名为`require`。我们使用`assert`标识符是为了避免与用来从其他文件中加载代码的`require`标识符混淆。
