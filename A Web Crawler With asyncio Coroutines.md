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
http://www.python.org/~guido/。

### Introduction
Classical computer science emphasizes efficient algorithms that complete computations as quickly as possible. But many networked programs 
spend their time not computing, but holding open many connections that are slow, or have infrequent events. These programs present a very 
different challenge: to wait for a huge number of network events efficiently. A contemporary approach to this problem is asynchronous I/O, 
or "async".

经典计算机科学注重高效的算法,尽可能用最少的时间完成计算。然而很多网络程序的时间并非花在计算上，而是用于保持许多低速接口的连接状态,或处理一些偶然事件。如
何更有效地等待大量的网络事件是过去那些以效率换时间的程序所面临的巨大挑战。现在,我们用异步I/O(异步)来解决这个问题。
