
一：背景


1. 讲故事
公司部署在某碟上的项目在9月份压测50并发时，发现某个容器线程、内存非正常的上涨，导致功能出现了异常无法使用。根据所学，自己分析了下线程和内存问题，分析时可以使用lldb或者windbg，但是个人比较倾向于界面化的windbg，所以最终使用windbg开干。


二：WinDbg 分析


1. 到底是哪里的泄漏
在 windows 平台上相信有很多朋友都知道用 !address \-summary 命令看，但这是专属于windows平台的命令，在分析linux上的dump不好使，参考如下输出：



```
0:000> !address -summary
                                     
Mapping file section regions...
Mapping module regions...
Mapping heap regions...

--- Usage Summary ---------------- RgnCount ----------- Total Size -------- %ofBusy %ofTotal
                              4062 ffffffff`f5638600 (  16.000 EB) 100.00%  100.00%
Image                                  1282        0`09fc8a00 ( 159.784 MB)   0.00%    0.00%

--- Type Summary (for busy) ------ RgnCount ----------- Total Size -------- %ofBusy %ofTotal
                                       2431 fffffffe`2b813000 (  16.000 EB)          100.00%
MEM_PRIVATE                            2913        1`d3dee000 (   7.310 GB)   0.00%    0.00%

--- State Summary ---------------- RgnCount ----------- Total Size -------- %ofBusy %ofTotal
                                       2431 fffffffe`2b813000 (  16.000 EB) 100.00%  100.00%
MEM_COMMIT                             2913        1`d3dee000 (   7.310 GB)   0.00%    0.00%

--- Protect Summary (for commit) - RgnCount ----------- Total Size -------- %ofBusy %ofTotal
PAGE_READWRITE                         2115        1`cb683000 (   7.178 GB)   0.00%    0.00%
PAGE_EXECUTE_READ                       175        0`03d49000 (  61.285 MB)   0.00%    0.00%
PAGE_READONLY                           585        0`03ce9000 (  60.910 MB)   0.00%    0.00%
PAGE_EXECUTE_WRITECOPY                   38        0`00d39000 (  13.223 MB)   0.00%    0.00%

--- Largest Region by Usage ----------- Base Address -------- Region Size ----------
                              7ffc`011fa000 ffff8003`fe406000 (  16.000 EB)
Image                                  7f45`fe4e9000        0`01b16000 (  27.086 MB)


```

卦中的内存段分类用处不大，也没有多大的参考价值，那怎么办呢？其实 coreclr 团队也考虑到了这个情况，它提供了一个 maddress 命令来实现跨平台的 !address，更改后输出如下：



```

0:000> !sos maddress
Enumerating and tagging the entire address space and caching the result...
Subsequent runs of this command should be faster.
+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+ 
 | Memory Kind            |        StartAddr |        EndAddr-1 |         Size | Type        | State       | Protect                | Image                                                   | 
 +--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+ 
 | Stack                  |     7f42d256e000 |     7f42d2d6e000 |       8.00mb | MEM_PRIVATE | MEM_COMMIT  | PAGE_READWRITE         |                                                         | 
 | Stack                  |     7f42d3570000 |     7f42d3d70000 |       8.00mb | MEM_PRIVATE | MEM_COMMIT  | PAGE_READWRITE         |                                                         | 
 | Stack                  |     7f42d3d71000 |     7f42d4571000 |       8.00mb | MEM_PRIVATE | MEM_COMMIT  | PAGE_READWRITE         |                                                         | 
 | Stack                  |     7f42d4572000 |     7f42d4d72000 |       8.00mb | MEM_PRIVATE | MEM_COMMIT  | PAGE_READWRITE         |                                                         | 
 | Stack                  |     7f42d4d73000 |     7f42d5573000 |       8.00mb | MEM_PRIVATE | MEM_COMMIT  | PAGE_READWRITE         |                                                         | 
 | Stack                  |     7f42d5574000 |     7f42d5d74000 |       8.00mb | MEM_PRIVATE | MEM_COMMIT  | PAGE_READWRITE         |                                                         | 
 | Stack                  |     7f42d5d75000 |     7f42d6575000 |       8.00mb | MEM_PRIVATE | MEM_COMMIT  | PAGE_READWRITE         |                                                         | 
 | Stack                  |     7f42d6d77000 |     7f42d7577000 |       8.00mb | MEM_PRIVATE | MEM_COMMIT  | PAGE_READWRITE         |                                                         | 
 | Stack                  |     7f42d7578000 |     7f42d7d78000 |       8.00mb | MEM_PRIVATE | MEM_COMMIT  | PAGE_READWRITE         |                                                         | 
 | Stack                  |     7f42d7d79000 |     7f42d8579000 |       8.00mb | MEM_PRIVATE | MEM_COMMIT  | PAGE_READWRITE         |                                                         | 
 | Stack                  |     7f42d857a000 |     7f42d8d7a000 |       8.00mb | MEM_PRIVATE | MEM_COMMIT  | PAGE_READWRITE         |                                                         | 
 ...
 +-------------------------------------------------------------------------+ 
 | Memory Type            |          Count |         Size |   Size (bytes) | 
 +-------------------------------------------------------------------------+ 
 | Stack                  |            788 |       6.28gb |  6,743,269,376 | 
 | GCHeap                 |             48 |     688.98mb |    722,448,384 | 
 | PAGE_READWRITE         |            930 |     180.22mb |    188,977,152 | 
 | Image                  |          1,278 |     159.69mb |    167,447,040 | 
 | HighFrequencyHeap      |            327 |      20.35mb |     21,336,064 | 
 | LowFrequencyHeap       |            259 |      18.31mb |     19,202,048 | 
 | LoaderCodeHeap         |             15 |      17.53mb |     18,378,752 | 
 | HostCodeHeap           |             11 |       1.51mb |      1,581,056 | 
 | ResolveHeap            |              1 |     348.00kb |        356,352 | 
 | PAGE_READONLY          |            123 |     261.50kb |        267,776 | 
 | DispatchHeap           |              1 |     196.00kb |        200,704 | 
 | IndirectionCellHeap    |              3 |     152.00kb |        155,648 | 
 | LookupHeap             |              3 |     144.00kb |        147,456 | 
 | CacheEntryHeap         |              2 |     100.00kb |        102,400 | 
 | PAGE_EXECUTE_WRITECOPY |              5 |      96.00kb |         98,304 | 
 | StubHeap               |              2 |      76.00kb |         77,824 | 
 | PAGE_EXECUTE_READ      |              2 |       8.00kb |          8,192 | 
 +-------------------------------------------------------------------------+ 
 | [TOTAL]                |          3,798 |       7.34gb |  7,884,054,528 | 
 +-------------------------------------------------------------------------+ 

```

从卦中可以看到当前程序总计 6\.28gb 内存占用，基本上都被线程栈给吃掉了，更让人意想不到的是这个线程栈居然占用 8M 的内存空间，这个着实有点大了，而且 linux 不像 windows 有一个 reserved 的概念，这里的 8M 是实实在在的预占，可以观察这 8M 的内存地址即可，都是初始化的 0, 这就说不过去了。



```
0:000> dp 7f42d256e000 7f42d2d6e000
...
00007f42`d2d6dfa0  00000000`00000000 00000000`00000000
00007f42`d2d6dfb0  00000000`00000000 00000000`00000000
00007f42`d2d6dfc0  00000000`00000000 00000000`00000000
00007f42`d2d6dfd0  00000000`00000000 00000000`00000000
00007f42`d2d6dfe0  00000000`00000000 00000000`00000000
00007f42`d2d6dff0  00000000`00000000 00000000`00000000
00007f42`d2d6e000  ????????`????????

```

2. 如何修改栈空间大小
一般来说不同的操作系统发行版有不同的默认栈空间配置，可以先到内存搜一下当前是哪一个发行版，做法就是搜索操作系统名称主要关键字。



```

0:000> s-a 0 L?0xffffffffffffffff "centos"
...
00005570`9cddbc18  63 65 6e 74 6f 73 2e 37-2d 78 36 34 00 00 00 00  centos.7-x64....
...

```

从卦中可以看到当前操作系统是 centos7\-x64，在 windows 平台上修改栈空间大小可以修改 PE 头，在 linux 上有两种做法。


修改 ulimit \-s 参数（不建议）



```

root@ubuntu:/data# ulimit -s
8192
root@ubuntu:/data# ulimit -s 2048
root@ubuntu:/data# ulimit -s
2048

```

修改 DOTNET\_DefaultStackSize 环境变量（建议，针对异常容器在环境变量配置）


DOTNET\_DefaultStackSize\=180000


更多可以参考文章： [https://www.alexander\-koepke.de/post/2023\-10\-18\-til\-dotnet\-stack\-size/](https://github.com)


上面是解决问题的第一个方向，接下来我们说另一个方向，为什么会产生总计 888 个线程呢？


3. 为什么会有那么多线程
要找到这个答案，需要去看每一个线程此时都在干嘛，这个可以使用 windbg 专属命令。



```

0:000> ~*e !clrstack
...
OS Thread Id: 0x1b82 (225)
        Child SP               IP Call Site
00007F441B7FD660 00007f4cdbb69ad8 [HelperMethodFrame_1OBJ: 00007f441b7fd660] System.Threading.Monitor.ObjWait(Int32, System.Object)
00007F441B7FD790 00007f4c676318cd System.Threading.ManualResetEventSlim.Wait(Int32, System.Threading.CancellationToken) [/_/src/libraries/System.Private.CoreLib/src/System/Threading/ManualResetEventSlim.cs @ 570]
00007F441B7FD810 00007f4c676312e1 System.Net.Sockets.SocketAsyncContext.PerformSyncOperation[[System.__Canon, System.Private.CoreLib]](OperationQueue`1 ByRef, System.__Canon, Int32, Int32) [/_/src/libraries/System.Net.Sockets/src/System/Net/Sockets/SocketAsyncContext.Unix.cs @ 1330]
00007F441B7FD8A0 00007f4c67e26ff1 System.Net.Sockets.SocketAsyncContext.ReceiveFrom(System.Memory`1, System.Net.Sockets.SocketFlags ByRef, Byte[], Int32 ByRef, Int32, Int32 ByRef) [/_/src/libraries/System.Net.Sockets/src/System/Net/Sockets/SocketAsyncContext.Unix.cs @ 1557]
00007F441B7FD920 00007f4c67e2ea6b System.Net.Sockets.SocketPal.Receive(System.Net.Sockets.SafeSocketHandle, Byte[], Int32, Int32, System.Net.Sockets.SocketFlags, Int32 ByRef)
00007F441B7FD9A0 00007f4c67e26c37 System.Net.Sockets.Socket.Receive(Byte[], Int32, Int32, System.Net.Sockets.SocketFlags, System.Net.Sockets.SocketError ByRef)
00007F441B7FDA20 00007f4c67e26929 System.Net.Sockets.NetworkStream.Read(Byte[], Int32, Int32) [/_/src/libraries/System.Net.Sockets/src/System/Net/Sockets/NetworkStream.cs @ 231]
00007F441B7FDA70 00007f4c69b85757 System.IO.BufferedStream.ReadByteSlow() [/_/src/libraries/System.Private.CoreLib/src/System/IO/BufferedStream.cs @ 771]
00007F441B7FDA90 00007f4c69b774e8 System.IO.BinaryReader.ReadByte() [/_/src/libraries/System.Private.CoreLib/src/System/IO/BinaryReader.cs @ 207]
00007F441B7FDAA0 00007f4c69b853ee RabbitMQ.Client.Impl.InboundFrame.ReadFrom(RabbitMQ.Util.NetworkBinaryReader)
00007F441B7FDAF0 00007f4c69b852c6 RabbitMQ.Client.Framing.Impl.Connection.MainLoopIteration()
00007F441B7FDB10 00007f4c69b57068 RabbitMQ.Client.Framing.Impl.Connection.MainLoop()
00007F441B7FDB50 00007f4c67590d19 System.Threading.ExecutionContext.RunInternal(System.Threading.ExecutionContext, System.Threading.ContextCallback, System.Object) [/_/src/libraries/System.Private.CoreLib/src/System/Threading/ExecutionContext.cs @ 183]
00007F441B7FDCF0 00007f4cdb1e3aa7 [DebuggerU2MCatchHandlerFrame: 00007f441b7fdcf0] 
...

```

可以使用正规的 dotnet\-dump 或者 procdump抓取，根据上面卦象展示，可以看到大量的和 RabbitMQ.Client.Framing.Impl 有关的链接库，猜测大量线程卡在 RabbitMQ.Client.Framing.Impl 中。


有了这些知识，最后给到朋友的建议如下：


修改 DOTNET\_DefaultStackSize 参数
可以仿照 windows 上的 .netcore 默认 1\.5M 的栈空间设置，因为8M真的太大了，扛不住，也和 Linux 的低内存使用不符。修改后压测读取dump观察发现配置已生效



```

0:000> !sos maddress
Enumerating and tagging the entire address space and caching the result...
Subsequent runs of this command should be faster.
*** WARNING: Unable to verify timestamp for lttng-ust-wait-8-0
*** WARNING: Unable to verify timestamp for lttng-ust-wait-8
 +--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+ 
 | Memory Kind            |        StartAddr |        EndAddr-1 |         Size | Type        | State       | Protect                | Image                                                   | 
 +--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+ 
 .......
 | Stack                  |     7fabe4e8c000 |     7fabe500c000 |       1.50mb | MEM_PRIVATE | MEM_COMMIT  | PAGE_READWRITE         |                                                         | 
 | Stack                  |     7fabe500d000 |     7fabe518d000 |       1.50mb | MEM_PRIVATE | MEM_COMMIT  | PAGE_READWRITE         |                                                         | 
 | Stack                  |     7fabe518e000 |     7fabe530e000 |       1.50mb | MEM_PRIVATE | MEM_COMMIT  | PAGE_READWRITE         |                                                         | 
 | Stack                  |     7fabe530f000 |     7fabe548f000 |       1.50mb | MEM_PRIVATE | MEM_COMMIT  | PAGE_READWRITE         |                                                         | 
 | Stack                  |     7fabe5490000 |     7fabe5610000 |       1.50mb | MEM_PRIVATE | MEM_COMMIT  | PAGE_READWRITE         |                                                         | 
 | Stack                  |     7fabe5611000 |     7fabe5791000 |       1.50mb | MEM_PRIVATE | MEM_COMMIT  | PAGE_READWRITE         |                                                         | 
 | Stack                  |     7fabe5792000 |     7fabe5912000 |       1.50mb | MEM_PRIVATE | MEM_COMMIT  | PAGE_READWRITE         |                                                         | 
 | Stack                  |     7fabe5913000 |     7fabe5a93000 |       1.50mb | MEM_PRIVATE | MEM_COMMIT  | PAGE_READWRITE         |                                                         | 
 | Stack                  |     7fabe5a94000 |     7fabe5c14000 |       1.50mb | MEM_PRIVATE | MEM_COMMIT  | PAGE_READWRITE         |                                                         | 
 | Stack                  |     7fabe5c15000 |     7fabe5d95000 |       1.50mb | MEM_PRIVATE | MEM_COMMIT  | PAGE_READWRITE         |                                                         | 
 .......
 +-------------------------------------------------------------------------+ 
 | Memory Type            |          Count |         Size |   Size (bytes) | 
 +-------------------------------------------------------------------------+ 
 | Stack                  |            766 |       1.41gb |  1,518,571,520 | 
 | GCHeap                 |             48 |     702.39mb |    736,509,952 | 
 | PAGE_READWRITE         |            931 |     186.31mb |    195,358,720 | 
 | Image                  |          1,283 |     158.77mb |    166,480,384 | 
 | HighFrequencyHeap      |            336 |      20.97mb |     21,991,424 | 
 | LowFrequencyHeap       |            256 |      18.32mb |     19,214,336 | 
 | LoaderCodeHeap         |             15 |      17.53mb |     18,378,752 | 
 | HostCodeHeap           |             11 |       1.63mb |      1,703,936 | 
 | ResolveHeap            |              1 |     348.00kb |        356,352 | 
 | PAGE_READONLY          |            123 |     261.50kb |        267,776 | 
 | DispatchHeap           |              1 |     196.00kb |        200,704 | 
 | IndirectionCellHeap    |              3 |     152.00kb |        155,648 | 
 | LookupHeap             |              3 |     144.00kb |        147,456 | 
 | PAGE_EXECUTE_WRITECOPY |              5 |     132.00kb |        135,168 | 
 | CacheEntryHeap         |              2 |     100.00kb |        102,400 | 
 | StubHeap               |              2 |      76.00kb |         77,824 | 
 | PAGE_EXECUTE_READ      |              2 |       8.00kb |          8,192 | 
 +-------------------------------------------------------------------------+ 
 | [TOTAL]                |          3,788 |       2.50gb |  2,679,660,544 | 
 +-------------------------------------------------------------------------+ 


```

观察项目代码中RabbitMQ.Client.Framing.Impl 的相关逻辑
发现该引用其实在代码中属于无效引用，将该引用删除压测观察，发现线程正常。


三：总结
Linux 上的 .NET 调试生态在日渐丰富，这是一件让人很兴奋的事情，最后给我的老师[《一线码农》](https://github.com):[豆荚加速器](https://yirou.org)和 WinDbg 点个赞


