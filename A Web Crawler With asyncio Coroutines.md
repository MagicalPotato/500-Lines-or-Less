### A. Jesse Jiryu Davis and Guido van Rossum(作者简介)
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

### Introduction(概述)
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

### The Task(任务概况)
A web crawler finds and downloads all pages on a website, perhaps to archive or index them. Beginning with a root URL, it fetches each 
page, parses it for links to unseen pages, and adds these to a queue. It stops when it fetches a page with no unseen links and the queue 
is empty.

爬虫查找和下载所有页面,是为了存档或索引它们。从根URL开始抓取并分析每个页面，如果该页面的有未被抓取过的链接,就将其添加到队列中。当页面上所有链接都被抓
取完或者队列为空时,爬虫将会停止。

We can hasten this process by downloading many pages concurrently. As the crawler finds new links, it launches simultaneous fetch 
operations for the new pages on separate sockets. It parses responses as they arrive, adding new links to the queue. There may come some 
point of diminishing returns where too much concurrency degrades performance, so we cap the number of concurrent requests, and leave the 
remaining links in the queue until some in-flight requests complete.

我们可以通过同时下载多个页面来加速爬取过程。当爬虫发现新的链接时，它会在一个单独的套接字程序上启动抓取新页面的操作,解析新链接的响应,而后将新链接加入抓
取队列.在并发量大到产生性能问题时会出现间歇地卡顿,所以我们限制了并发请求的数量,将获取到的链接保留在队列中直到那些正在进行的任务完成后才继续新任务。

### The Traditional Approach(传统方法)
How do we make the crawler concurrent? Traditionally we would create a thread pool. Each thread would be in charge of downloading one 
page at a time over a socket. For example, to download a page from xkcd.com:

爬虫如何并发?传统的方式是创建一个线程池,每个线程负责通过套接字一次下载一个页面。例如,要从xkcd.com下载页面,传统方式如下：
```
def fetch(url):
    sock = socket.socket()
    sock.connect(('xkcd.com', 80))
    request = 'GET {} HTTP/1.0\r\nHost: xkcd.com\r\n\r\n'.format(url)
    sock.send(request.encode('ascii'))
    response = b''
    chunk = sock.recv(4096)
    while chunk:
        response += chunk
        chunk = sock.recv(4096)

    # Page is now downloaded.
    links = parse_links(response)
    q.add(links)
```
By default, socket operations are blocking: when the thread calls a method like connect or recv, it pauses until the operation 
completes. Consequently to download many pages at once, we need many threads. A sophisticated application amortizes the cost of thread-
creation by keeping idle threads in a thread pool, then checking them out to reuse them for subsequent tasks; it does the same with 
sockets in a connection pool.

默认情况下，套接字出于阻塞状态：当线程调用connect或recv之类的方法时，它会暂停直到操作完成。因此，要一次下载多个页面，我们需要很多线程。复杂的应用程序
为了节省频繁创建线程的开销,会在线程池中保留一些空闲线程，并在需要时将其分配给后续任务; 连接池中的套接字也是如此。

And yet, threads are expensive, and operating systems enforce a variety of hard caps on the number of threads a process, user, or machine 
may have. On Jesse's system, a Python thread costs around 50k of memory, and starting tens of thousands of threads causes failures. If we 
scale up to tens of thousands of simultaneous operations on concurrent sockets, we run out of threads before we run out of sockets. Per-
thread overhead or system limits on threads are the bottleneck.

然而，线程的开销很大，并且操作系统对系统中的进程数，用户或机器能拥有的线程数量有各种硬性限制。 在Jesse的系统上，一个Python线程大约占用50k的内存，当线
程数量上万时会导致新线程启动失败。如果在并发套接字上存在数以万计的并发操作，那么在套接字还没用完之前我们会先用完线程。线程自身的开销或系统限制是传统爬
虫并发的瓶颈。

In his influential article "The C10K problem", Dan Kegel outlines the limitations of multithreading for I/O concurrency. He says:It's 
time for web servers to handle ten thousand clients simultaneously, don't you think? After all, the web is a big place now.

Dan Kegel就曾在他著名的文章“The C10K problem”中阐了多线程对I/O并发的局限性。他说:是时候让服务器同时处理一万个请求了。你不觉得吗？毕竟网络现在可是一个大地方。

Kegel coined the term "C10K" in 1999. Ten thousand connections sounds dainty now, but the problem has changed only in size, not in kind. 
Back then, using a thread per connection for C10K was impractical. Now the cap is orders of magnitude higher. Indeed, our toy web crawler 
would work just fine with threads. Yet for very large scale applications, with hundreds of thousands of connections, the cap remains: 
there is a limit beyond which most systems can still create sockets, but have run out of threads. How can we overcome this?

Kegel在1999年提出了“一万个连接的限制”这个概念。一万个连接现在听起来不算什么，但这并不是说现在数量问题已经得到解决,事实上本质并未改变。那时候，单纯为了
去契合"C10K"而为每个连接创建一个线程是不切实际的。如今这个上限高出了好几个数量级,我们这个示例爬虫也的确可以很好地处理线程。然而,对于具有数十万个连接
的超大规模应用程序，这个限制依然存在：超过这个限制，大多数系统仍可以创建套接字，但却没法再创建线程了。我们该如何解决这个问题？

### Async(异步)
Asynchronous I/O frameworks do concurrent operations on a single thread using non-blocking sockets. In our async crawler, we set the 
socket non-blocking before we begin to connect to the server:

异步I/O框架使用非阻塞套接字在单个线程上执行并发操作。这个异步爬虫，在连接到服务器之前，我们会先将套接字设置为非阻塞：
```
sock = socket.socket()
sock.setblocking(False)
try:
    sock.connect(('xkcd.com', 80))
except BlockingIOError:
    pass
```
Irritatingly, a non-blocking socket throws an exception from connect, even when it is working normally. This exception replicates the 
irritating behavior of the underlying C function, which sets errno to EINPROGRESS to tell you it has begun.

这里有个问题，即便非阻塞套接字在正常工作状况下,它也可能会抛出异常。这个异常是由底层的C函数导致的，该函数将一个变量errno设置为EINPROGRESS(个人编程
习惯,将其拆成e in progress,大意是什么什么正在进行中)，以告诉您它已经开始工作了。

Now our crawler needs a way to know when the connection is established, so it can send the HTTP request. We could simply keep trying in a 
tight loop:

现在，我们的爬虫需要通过一种方式来判断何时建立连接，以便发送HTTP请求。一个简单的循环即可：
```
request = 'GET {} HTTP/1.0\r\nHost: xkcd.com\r\n\r\n'.format(url)
encoded = request.encode('ascii')

while True:
    try:
        sock.send(encoded)
        break  # Done.
    except OSError as e:
        pass

print('sent')
```

