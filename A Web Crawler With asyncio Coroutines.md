### A. Jesse Jiryu Davis and Guido van Rossum
A. Jesse Jiryu Davis is a staff engineer at MongoDB in New York. He wrote Motor, the async MongoDB Python driver, and he is the lead 
developer of the MongoDB C Driver and a member of the PyMongo team. He contributes to asyncio and Tornado. He writes at 
http://emptysqua.re.

A. Jesse Jiryu Davis是纽约的一名MongoDB工程师。 他编写了Motor,一个异步的MongoDB Python驱动程序，同时他也是MongoDB C驱动程序的主要开发人员和PyMongo
团队的成员。 他对asyncio和Tornado亦有贡献。 其个人主页是http://emptysqua.re 。

Guido van Rossum is the creator of Python, one of the major programming languages on and off the web. The Python community refers to him as 
the BDFL (Benevolent Dictator For Life), a title straight from a Monty Python skit. Guido's home on the web is 
http://www.python.org/~guido/.

Guido van Rossum是互联网主要编程语言之一--Pytho 的创始人。Python社区将他称为BDFL（仁慈的生活独裁者），该标题取自短剧Monty Python。其个人主页是
http://www.python.org/~guido/ 。

### Introduction
Classical computer science emphasizes efficient algorithms that complete computations as quickly as possible. But many networked programs 
spend their time not computing, but holding open many connections that are slow, or have infrequent events. These programs present a very 
different challenge: to wait for a huge number of network events efficiently. A contemporary approach to this problem is asynchronous I/O, 
or "async".

经典计算机科学注重高效的算法,尽可能用最少的时间完成计算。然而很多网络程序的时间并非花在计算上，而是用于保持许多低速接口的连接状态,或处理一些偶然事件。如
何更有效地等待大量的网络事件是过去那些以效率换时间的程序所面临的巨大挑战。现在,我们用异步I/O(异步)来解决这个问题。

This chapter presents a simple web crawler. The crawler is an archetypal async application because it waits for many responses, but does 
little computation. The more pages it can fetch at once, the sooner it completes. If it devotes a thread to each in-flight request, then as 
the number of concurrent requests rises it will run out of memory or other thread-related resource before it runs out of sockets. It avoids 
the need for threads by using asynchronous I/O.

本章介绍一个简单的Web爬虫。该爬虫是一个典型的异步应用程序，能等待大量响应，但花费的计算量却不多,一次获取的页面越多，它完成得越快。如果我们为每一个活跃的
爬取请求分配一个线程,随着请求数量的增加,在套接字用完之前,程序便会消耗完所有线程资源或导致内存溢出。所以当前爬虫采用了异步I/O而没有使用线程。

We present the example in three stages. First, we show an async event loop and sketch a crawler that uses the event loop with callbacks: 
it is very efficient, but extending it to more complex problems would lead to unmanageable spaghetti code. Second, therefore, we show 
that Python coroutines are both efficient and extensible. We implement simple coroutines in Python using generator functions. In the 
third stage, we use the full-featured coroutines from Python's standard "asyncio" library1, and coordinate them using an async queue.

我们分三步来介绍这个例子。首先，我们展示一个异步循环并编写一个递归使用该循环的爬虫：它非常有效，但将其扩展到更复杂的问题时会导致代码混乱不堪无法管理。 
因此在第二步,我们引出了既高效又可扩展的Python协调器,并用Python生成器实了现简单的协调器。第三步，我们直接使用Python标准库提供的高级协调器，并用异步队
列来协调他们。
