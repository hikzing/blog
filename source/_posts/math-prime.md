---
title: 素数生成算法探索
date: 2014-09-21
tags:
---


作为一名数学系学生, 最开心的莫过于用计算机玩数学了:)

这篇文章, 我们来玩一下数学界的宠儿, 素数.


什么是素数?
-----
>[维基百科](http://zh.wikipedia.org/wiki/素数)

在你继续往下看之前, 请**记住**下面这两点.

**算术基本定理:** [`任何大于1的自然数都可以表示成素数的乘积`](http://zh.wikipedia.org/wiki/算术基本定理)

**素数的分解:** `除了2和3, 任何素数都可以表示为6*n + 1或者6*n - 1的形式.`
>不难证明, 因为自然数只能表示成6N±1, 6N±2, ... , 6N±6 六种形式, 而只有6N+1和6N+5这两种形式才**有可能**是素数. 所以, 这个性质是推出一个数是素数的必要不充分条件.

<!-- more -->

如何判定素数?
---
**一般方法**

    def isprime(n):
        for i in range(2, int(n**.5+1)):
            if n%i == 0:
                return False
        return n > 1    # ensure n != 0 or n!= 1

利用素数的定义, 如果数字n能被除了1和自己以外的数整除, 那么n就不是素数. 这是平时用得最多的方法, 也是最简单的方法.

**数学系孩子的方法(maybe)**

    def isprime(n):
        if n < 5:
            return n == 2 or n ==3
        return n%2 and n%3 and all(n%(6*k-1) and n%(6*k+1) for k in
                                range(1, int((n**.5+1)/6)+1))

原理是利用开头提到的两条性质, 依次用素数去除n, 如果存在模为0的情况, 那么说明这个数除了1和它本身, 还能被其它数整除, 不符合素数定义, 因此不是素数.
>遗憾的是, 在我的计算机上, 它的运行速度比第一个方法慢一点 : )

**数学系学霸的方法**

    def isprime(n):
        if n == 561:
            return False
        return all((b**n)%n == b%n for b in [3, 5])

原理: [费马素性测试](http://zh.wikipedia.org/wiki/费马素性检验) + [卡迈克尔数](http://zh.wikipedia.org/wiki/卡邁克爾數)

由上面提到了两条定理, 若有一个数n, 对于x属于[2, n], 如果符合(x**n)%n == x%n , 那么它`有可能`是素数. "有可能"的意思是一个数符合了这个表达式, 但是它不是素数.  唯一的增加可能性的办法是取不同的 x多次验证.

然而, 就算我们判断了全部可能的x, 也不能100%确定某个数一定是素数, 因为还有一种叫`卡迈克尔数`的存在, 因此, 还需要排除掉卡迈克尔数 = _ =

在我们的代码中, 只排除了561, 561是1000以内的唯一一个卡迈克尔数, 所以这个算法只对小于1000的数有效.(能达到百分百准确率).

>你会问, 为什么要那么麻烦呢? 因为它快啊!


不愧是学霸的算法, 类似的"不准确"的算法还有很多, 喜欢的话去搜一下**素性测试**吧 :)
>当然, 就算是这样的学霸算法, 在 ACMer 面前...


素数的生成
----

普通的方法就不说了, 无非是对数字一个一个的判断, 如果是素数就输出, 不是就跳过. 很显然, 这样做的效率是非常低的, 我们来研究下其它方法.

**方法一: 利用任意素数可以表示成关于6*n ± 1的性质**

>这里, 引用下知乎的[Rio](http://www.zhihu.com/people/rio)大神写的代码

    def prime():
        '''Prime sequence generator.'''

        def hexstep():
            '''Generate 6n-1, 6n+1 sequence. '''
            n   = 1
            while True:
                hex = 6*n
                yield hex-1
                yield hex+1
                n   += 1

        def isprime(n):
            for p in primes:
                if n%p==0:  return False
                if p**2>n:  return True     # p**2>n is faster than p>n**.5

        yield 2
        yield 3
        primes  = [3]   # hexstep generates odd numbers, so 2 is unneccessary
                        # this list is interal cache of calcuated primes

        for n in hexstep():    # all primes (except 2 and 3) are 6n-1 or 6n+1
            if isprime(n):
                yield n
                primes.append(n)

原理很简单, `hexstep()`负责生成6*n-1, 6*n+1序列, `isprime()`判断生成的序列是否为素数.之所以要判断, 是因为不是所有的生成数都是素数, 如25 = 6*4+1, 但它不是素数, 如果判定成功, 则返回它. 于此同时, 将这个数加入primes列表

>如果你不能理解这里用到的isprime()函数为什么可以这么写, 那么回过头去看看**算术的基本定理**吧:)

**方法二: [Eratosthenes筛选法](http://zh.wikipedia.org/zh-cn/埃拉托斯特尼筛法)**

代码可以这样写:

    def primes2(n):
        '''Sieve of Eratosthenes

        Return a list of all primes less than n
        '''
        tmp = set(range(2, n))
        i   = 2
        while i < int(n**.5)+1:
            tmp -= set(range(i*2, n+1, i))  # e.g. set([1,2,3, 4]) - set([2, 4])  => {1, 3}
              i += 1

        return tmp

数据结构用的是内置的集合类型, 利用它的性质, 可以很轻松的是实现数据的删除. 虽然看起来易懂, 但它太耗空间和时间了.

改进下.

**方法三: 效率更高, 速度更快的Eratosthenes筛选法**

    def primes3(n):
        """ Sieve of Eratosthenes """
        l = range(n)
        l[1] = 0
        for i in range(2, int(n**.5)+1):
            if l[i]:
                l[i**2::i] == [0] * (n-1-i**2)//i + 1

        return [x for x in l if x]

速度一下子就上来了 : ).
>在我的机器上, 它的运行时间是方法二的一半以下(对于100万以内的数据)

还有更强的吗? 有!

**方法四: 还要快一点的Eratosthenes筛选法**

    def eratosthenes():
        '''Yields the sequence of prime numbers via the Sieve of Eratosthenes.'''
        D = {}  # map composite integers to primes witnessing their compositeness
        q = 2   # first integer to test for primality
        while 1:
            if q not in D:
                yield q        # not marked composite, must be prime
                D[q*q] = [q]   # first multiple of q not already marked
            else:
                for p in D[q]: # move each witness to its next multiple
                    D.setdefault(p+q,[]).append(p)
                del D[q]       # no longer need D[q], free memory
            q += 1

这是一个大神写的, 但是已经记不住出处了, 这是上面介绍的所有方法中速度最快的, 当然代价是是稍微需要多思考一下: )

关于素数, 远不止这些内容, 实在是冰山一脚, 以后再讨论吧.
