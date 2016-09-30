---
title: Python yield 的黑魔法
date: 2015-09-30 01:25:23
tags:
---

在开始之前, 我们先来熟悉下和 yield 非常相关的协程的概念

>### 什么是协程

>维基: [定义][1]

>知乎: [协程的好处是什么][2]

>总的来说, 协程的出现是为了充分利用并发的优势, 同时又减少了线程(进程)进行切换时的系统开销而出现的一种技术, 相当于程序本身承担起调度, 切换等功能, 而在用户层面上实现的"微线程", "用户级线程"

>典型的和协程相关的 Python 库有`gevent` `twisted` `greenlet`等...

<!-- more -->

Python用 yield 关键字实现了不完全的协程

利用协程的特性, 我们可以做许多好玩的事, 比如:

### 生产者-消费者模型


``` python
    def consumer():
        thanks = "Thanks"
        while 1:
            food = yield thanks
            print 'consumer: get: {}'.format(food)


    def product():
        c = consumer()
        c.send(None)  # init coroutine
        for food in ['cola', 'apple', 'meat']:
            response = c.send(food)
            print 'product: get response: {}'.format(response)

    product()
```

``` shell
output:
    consumer: get: cola
    product: get response: Thanks
    consumer: get: apple
    product: get response: Thanks
    consumer: get: meat
    product: get response: Thanks
```

其中, `x = yield y` 这样的语法意思是:

*   x: 从发送者那里接收到的值
*   y: 返回给发送者的值


上面的代码展示了, 生产者通过 send 发送食物给消费者, 消费者返回感谢的过程. 这就是一个简单的利用 yield 的协程实现. 控制权在不同函数间自动切换,而不用操作系统的介入.



再来个复杂一点的例子

### Unix的Pipe 模型

类Unix系统中, 管道是进程间通信的一种手段. 如找出当前目录下包含"Python"字符的的文件, 再进行排序, 可以这样写

``` bash
$ ls -l | grep "Python" | sort
```
很优雅对吧, 我们用 yield 来模仿一下类似的实现.

先来看一个来自<编程珠玑>的变位词问题:
>给定一个英语字典，找出其中的所有变位词集合。例如，“pots”，“stop”和“tops”互为变位词，因为每一个单词都可以通过改变其他单词中的字母顺序来得到。

作者给出的思路是:

1. 对每个单词进行 sign(标记), 标记的方法为将单词按字典序排序. 如: pans的标记为anps, stop 的标记为opst

2. 对标记进行排序(也就是将相同标记的词放在一起)

3. 整理出具有相同标记的词.

举个例子吧, 对于数据 [pans, pots, opt, snap, stop, tops], 这个过程是这样的.


    pans            anps  pans             anps  pans
    pots            opst  pots             anps  snap              [pans, snap]
    opt   =>sign=>  opt   opt   =>sort=>   opt   opt   =>squash=>  [opt]
    snap            anps  snap             opst  pots              [pots, stop,
    stop            opst  stop             opst  stop               tops]
    tops            opst  tops             opst  tops


好了.扯得好远了= _ =.

模拟 Unix pipe 的写法, 可以这样写

```python
    from collections import defaultdict


    def find_anagram(input_data):
        """ find all the anagram from a dictionary
        """

        # initize a cotoutine
        def start(func):
            def _(*args, **kwargs):
                g = func(*args, **kwargs)
                g.send(None)
                return g
            return _

        @start
        def sign(target):
            while True:
                words = yield
                for w in words:
                    target.send([''.join(sorted(w)), w])
                target.send(None)  # flag incicates that all data have been seen

        @start
        def sort(target):
            sign_words = []
            while True:
                word = yield
                if word:
                    sign_words.append(word)
                else:  # all word have sort
                    target.send(sorted(sign_words))

        @start
        def squash():
            nonlocal dictionary  # python3 only: use the variable define in outside
            while True:
                word_list = yield
                for x in word_list:
                    dictionary[x[0]].add(x[1])

        dictionary = defaultdict(set)

        sign(sort(squash())).send(input_data)

        # filter the word has no anagram
        return filter(lambda x: len(x[1]) > 1, dictionary.items())


    if __name__ == "__main__":

        test_data = ['abc', 'acb', 'bca', 'iterl', 'liter', 'hello',
                     'subessential', 'suitableness', 'hello']
        result = find_anagram(test_data)
        for each in result:
            print(each[0], ':', each[1])
```

有点长. 但基本上就这一句话关键.

`sign(sort(squash())).send(input_data)`

意思是, 发送一个初始数据集给 sign, 然后 sign 处理数据后传给 sort, sort 处理后再传给 squash.

这和 pipe 的思想是一样的, 如果用 pipe 的写法, 那会是这样的.

`sign < input_data | sort | suqash`

是不是很神奇呢? 强大的yield.
>我第一次接触到这样的用法时那是惊呆了: )

类似的运用在一些 Web 框架中也有出现.

比较出名的如 Tornado, 它就利用 yield 和 装饰器 实现了`用同步的方式写异步的代码`这一功能.

像这样:

``` python
    class Index(RequestHandler):

        @gen.coroutine
        def get(self):

            response = yield AsyncHTTPClient().fetch(url)
            # some_thing_else
```
 没有回调函数! 异步 异步调用! 看起来就像是同步的!太酷了是不是!

>下一篇文章我们来分析下它是怎么实现的吧: )


总结: 文章写得有点乱= _ =. 大家不要介意: D, 有问题我们研究研究吧.




[1]: http://zh.wikipedia.org/wiki/%E5%8D%8F%E7%A8%8B

[2]: http://www.zhihu.com/question/20511233/answer/24260355
