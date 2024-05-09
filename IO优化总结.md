# I/O优化总结

## 线上监控

### 1，I/O 跟踪

方法一： Java Hook 根据java文件读写流的系统函数调用链，找到合适的hook点，采用反射或反射加动态代理的方式实现I/O跟踪。无法监控native代码，兼容性差，需根据Android版本做适配。

方法二：Native Hook 采用PLT/GOT Hook的方式，hook掉native层的系统io函数，挑选一些包含系统io方式的动态库，保证覆盖到所有java层的I/O调用。

监控内容：open ：文件名、fd、文件原始大小、堆栈、线程
		  read（write）：类型、读写次数、读写总大小、使用buffer大小、读写总耗时
		  close：打开文件总耗时、最大连续读写时间
通过测试，native层的hook对io调用性能损耗很小。

### 2，线上监控

1，主线程I/O
监控主线程连续读写超过100ms的操作

2，读写buffer过小
buffer size小于block size，一般为4KB
read、write次数超过一定次数，如5次

3，重复读 
重复读取次数超过3次，并且读取内容相同
读取期间内容没有更新，没有发生过write

4，资源泄露
文件、cursor没有及时close导致。利用strictMode监控资源泄露。（CloseGuard.java）
利用反射，把 CloseGuard 中的 ENABLED 值设为 true。
利用动态代理，把 REPORTER 替换成我们定义的 proxy。
对其他希望监控的资源，手动增加埋点，编写MyCloseGuard。

### 3，I/O与启动优化

通过跟踪拿到整个启动过程的io操作详细列表。检查是否必要。
优化：
1，对大文件使用mmap或者NIO方式。

2，安装包不压缩。启动需要的文件，不要压缩。

3，Buffer复用。Okio开源库的ByteString和Buffer通过重用，减少cpu和内存消耗。

4，存储结构和算法的优化。通过算法和数据结构优化减少IO，如配置文件从启动完全解析改为读取时才解析。替换一些性能差的数据结构（xml、json）。
