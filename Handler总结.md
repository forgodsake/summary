
#Message
一般来说，消息可以分为三类，分别是同步消息，异步消息，屏障消息
其中异步消息是系统绘制屏幕等时机发送的消息；
屏障消息没有target，用来保障异步消息的执行；
而我们平时使用的就是同步消息；

#MessageQueue
消息队列处理消息时，会判断当前是否需要立即唤醒epoll_wait
如果当前是首条消息或者是屏障消息或者当前消息比队头的消息触发时机早，
并且mBlocked为true，会使用nativeWake()唤醒。
如果当前消息不符合上述条件，但是当前消息是异步的，并且队头是屏障消息，
那么也有可能立即唤醒，会再进行一定判断。
唤醒的话，会从Looper的loop函数的循环里，从queue.next()阻塞处返回，接着
往下执行。判断是否到达执行时间，来觉得进入下次循环或者执行消息。

#Looper
Looper在准备好之后，会进入loop函数开启死循环。并不断从queue.next()取出消息
进行处理，queue.next()是一个阻塞函数，它只有在时间到了或者别的地方使用了
nativeWake()函数才会唤醒继续执行。

#Handler
无论是Handler的post类型函数还是send类型函数，最终都会调用到sendMessageAtTime函数，
而执行时间，则是系统开机时间加上指定的时间。其中post类型函数，会把要post的Runnable包装
为Message，指定其callback为我们发送的Runnable，最终回调是判断该callback存在，则执行
其run方法。

