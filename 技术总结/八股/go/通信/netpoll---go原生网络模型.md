前面学习了 Linux 的 IO 多路复用 select/poll/epoll 的实现原理，最近学习了下 Go 语言的 netpoll 网络轮询器，在学习的过程中，产生了下面这些疑问，相信对这块内容有所了解的同学都会比较关心：

1. Go语言的 TCP 连接怎样建立？C 客户端调用的 socket()、connect()，和服务端调用的 socket()、bind()、listen()、accept() 等系统函数是怎样被封装在 Go 语言中？这些函数的本质是什么？
2.  Go 服务端建立 TCP 连接用到的 Listen 和 Accept 函数创建的两个 socket fd 是一样的么？为什么这两个 socket fd 都要加入 epoll 对象中监听呢？
3. Go 运行时通过什么样的机制构建高效的 Go 网络轮询器？
4. Go 原生的网络模型 netpoll 是否存在单个 Goroutine 处理单个连接即 goroutine per connection 的问题？在大量连接的情况下，岂不是内存和调度开销都很大？是否有更高效的 Go 网络模型呢？

本文结合 Go1.18 相关源码，和 Linux 网络基础知识，对以上问题给出了自己的结论。

另外，关于怎样写好技术文章，最近也有了新的感悟，可以多些图片，而不要贴太多源码，尽量用准确精炼的语言将事情的底层原理解释清楚，尤其是不能光谈实现，而不解释原因，“为什么” 比 “怎样做” 更加重要，如果不能用几句话把事情的本质解释清楚，就不是真的理解。秉承着“把复杂留给自己，把简单留给读者”的写作思想，希望能帮助大家半小时搞定 Go 原生网络模型netpoll。

# 结论

本文的其他内容主要是得出了下面几个结论：

1. C 客户端通过 socket 函数创建 socket，通过 connect 函数发起三次握手和服务器端建立 TCP 连接；C服务器端调用 socket 函数建立 socket 对象，bind 函数将本地监听端口绑定到该**监听 socket** 中，listen函数会传入参数确定**监听 socket** 能建立的最大连接数，并且内核中启动循环函数，检查客户端的 TCP 连接是否可以建立，如果可以，则会通过三次握手、基于**该监听 socket** 创建一个个**子 socket**，以便和多个客户端建立不同的 **TCP 连接**，即 **连接socket**，这些建立好的**连接 socket** 都放在 **监听 socket** 的全连接队列中，等待 accept 函数获取。

2. Go 的 net.Listen() 函数的主要动作是：调用 socket()系统函数创建 socket 并设置非阻塞；调用bind() 系统函数绑定并监听本地的一个端口；调用 listen() 系统函数开始 **监听socket** 上的 TCP 连接是否到达；调用 epoll_create()函数创建一个 epoll 对象；调用epoll_etl() 将 listen 的**监听 socket** 添加到 epoll 对象中等待和客户端 TCP 连接的建立，在**连接 socket** 建立时得到通知，然后调用 Accept 函数返回这个**连接socket**。

3. Go 的 Accept() 函数的主要逻辑是：调用 accept() 系统函数从 **监听socket** 的全连接队列头一个TCP连接；如果连接没有到达，就把当前 Goroutine 阻塞掉；有客户端连接到来的话，将建立的 **连接socket** 添加到 epoll 对象中管理起来，监听该**连接socket** 的数据到达情况。  

4. 服务器端维护了两种 socket，父socket 可以叫做 **监听 socket （父socket，socket、bind、listen函数的操作对象）**，子socket 是指和客户端建立的 **连接 socket （子socket，listen函数中建立的、accept函数返回的）**，两种 socket 的文件描述符fd 都要添加到 IO 多路复用对象 epoll 中管理起来，前者监听**客户端连接的到达**，后者监听**连接 socket** 上**数据的到达**，注意这个差别**。**

5. Go 原生网络轮询器 netpoll 是通过 IO 多路复用模型 epoll 和 Goroutine 调度器两种机制结合起来形成的：G 监听 TCP 连接，当数据没有到达时会park住，不会陷入内核态，也不会让和它一起绑定的 M 陷入阻塞，而只是在用户态进行协程 G 的切换，而且 G 切换的开销大约只有线程 M 切换的三十分之一，极大的提高了网络监听的效率。

6. Go 原生网络模型 netpoll 通过一个 Goroutine 监听一个 TCP 连接，Goroutine-per-connection ，在海量连接的业务场景下， G 数量以及消耗的资源会呈线性趋势暴涨，优化方案可以是社区的Netpoll(字节开源)、gnet等。

# 1. Go程序怎样处理网络请求

要弄懂 Go 原生网络模型netpoll，需要先分析清楚 Go 语言怎样建立一个TCP连接，并弄明白 Go 客户端程序怎样封装 socket()、connect()，和服务端封装的 socket()、bind()、listen()、accept() 等系统函数的本质动作是什么，如果没有这些函数，会有什么问题。

服务端和客户端要发送数据，需要先通过一些函数建立 TCP 连接，不同的语言有不同的实现方式，不过本质逻辑是一样的。先看 C 的连接，再看 Go 的。

C 语言客户端建立 TCP 连接的代码是：

```cpp
// 客户端核心代码
int main(){
    fd = socket(AF_INET,SOCK_STREAM, 0);
    connect(fd, ...);
    ...
}
```

主要逻辑是创建 socket，然后调用 connect 连接 server。

C语言的服务端核心代码是：

```cpp
// C 语言的服务端核心代码
int main(int argc, char const *argv[])
{
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    bind(fd, ...);
    listen(fd, 128);
    accept(fd, ...);
    ...
}
```

主要逻辑是创建 socket，绑定端口，listen 监听，最后 accept 接收客户端的请求。 

![](https://km.woa.com/asset/b3f93ebd57af4bfa83f9bcf230693c29?height=1226&width=2396)

_图1.1 C程序 TCP 连接的建立_

如图1.1所示，是 C 程序 TCP 连接的建立过程，主要有下面几个步骤：

1）C 客户端通过 socket() 函数创建和服务端通信的 socket 对象，然后调用 connect() 函数传入 IP 和端口号，试图通过三次握手建立和服务端的 TCP 连接，这里先设置 socket 状态为 TCP_SYN_SENT （意思是发出第一次握手请求），动态选择一个客户端的端口，同时发出第一次握手请求 SYN；

2）C 服务端先通过 socket() 函数创建和客户端通信的 socket 对象（**监听 socket**），然后调用 bind() 函数绑定 socket 对象和服务器的 IP 及端口号，接着调用 listen() 函数启动一个内核的循环函数，检查客户端的握手请求是否到达，通过传入的第二个参数来设置本 socket 能建立的最大连接数，根据当前 socket 的半连接队列和全连接队列是否满了，来判断新的连接是否可以建立，如果可以，则会回应客户端的第一次握手请求 SYN，响应 SYN ACK 给客户端，并将当前握手信息添加到半连接队列（意思是 TCP 连接还未完全建立，但是可以建立）； —— 注意：这里的 socket 可以叫做 **监听 socket （父socket，socket、bind、listen函数的操作对象）**，以区分下面和客户端建立的 **连接 socket （子socket，listen函数中建立的、accept函数返回的）**；

3）客户端收到服务器端发来 SYN ACK 请求的时候，connect启动的一个处理函数会修改自己的 socket 状态为 ESTABLISHED （意思是连接已经建立），并向服务端发送 ACK请求，进行第三次握手；

4）服务器端收到第三次握手的 ACK 请求， 由 listen() 函数启动的循环函数函数，在第 2）步的半连接队列里找到刚才建立的一个半连接，并建立完整的 TCP **连接 socket**，然后将连接请求从半连接队列中删除，并添加 **连接 socket** 到全连接队列，并设置服务器这边的**连接 socket** 状态为 ESTABLISHED；

5）经过上面几步，客户端和服务器端的 TCP 连接就已经建立了，客户端维护了一个连接 socket 对象，服务器端维护了一个**监听 socket （父socket）**和多个 **连接socket（子socket）**对象，至于服务器端的 accept 函数，主要是将全连接队列队头的 **连接 socket** 返回给服务器使用而已。

需要说明的是，服务器端维护了两种socket，**监听 socket** 和 **连接 socket**，前者是服务器端调用 socket 函数建立的 socket 对象，bind 函数将本地监听端口绑定到该**监听 socket** 中，listen函数会在内核中启动循环函数，检查客户端的 TCP 连接是否可以建立，如果可以，则会基于该监听 socket 创建一个个**子 socket**，以便和多个客户端建立不同的 **TCP 连接**，即 **连接socket**，这些建立好的**连接 socket** 都放在 父socket的全连接队列中，等待 accept函数获取。  

Linux中万物皆文件，一个 socket 对象在内核中由一个文件表示，每个文件都有一个文件描述符 file description （简称 fd），是指在进程中打开的文件列表的序号。这里，为了方便理解，可以认为 socket对象 = fd 文件描述符 = TCP连接，后面要说到的 epoll 多路复用，需要将 socket 的 fd 写入红黑树结构，方便监控客户端连接的建立和连接 socket 上数据的到达。

每个socket 由五个字段确定的，即所谓的五元组，由（源IP地址，源端口，目的IP地址，目的端口，传输层协议）确定一个 socket 对象，简单理解，如果确定是TCP连接的话，**四元组****（源IP地址，源端口，目的IP地址，目的端口）可以确定一个socket**。服务器的 IP 和端口号相同，可以接收不同的客户端发来的请求，只要客户端的 IP 或端口号不同，都是不同的 TCP 连接。 这里，**监听 socket** 可以认为是**四元组****（服务器IP地址，服务器端口，0，0）**，**连接 socket** 是 **四元组****（服务器IP地址，服务器端口，客户端IP地址，客户端端口）**。

弄明白了 C 程序 TCP 连接的建立过程，我们看看 Go 语言的 TCP 连接的实现。

Go语言的客户端一般通过下面代码发送网络请求：

```go
func main() {
    conn, err := net.Dial("tcp", "127.0.0.1:3000")
    ......
    for {
        .....
        _, err := conn.Write([]byte(data)) // 向服务端发送数据
        if err != nil {
            // handle error
        }
        buf := make([]byte,512)
        n,err := conn.Read(buf)            //读取服务端端数据
    }
}
```

如果检查 net.Dial() 函数的源码，会发现它封装了通过各种协议（TCP、UDP、ICMP等）和服务器的IP、端口跟服务器建立连接的过程，底层调用的也是 socket 和 connect 系统函数，如图1.2所示。

![](https://km.woa.com/asset/37ac592311634effaa6a116aae997920?height=1140&width=1728)

_图1.2 Go客户端的net.Dial() 底层也是通过socket 和 connect 函数建立 TCP 连接_

Go语言开发的服务端接收请求的代码逻辑是：

```go
func main()  {
    listen,err := net.Listen("tcp",":8080") //创建监听
    if err != nil{
         // handle error
    }
    for{
        conn,errs := listen.Accept()  //接受客户端连接
        if errs != nil{
            // handle error
        }
        go handle(conn) // 一个Goroutine处理一个连接
    }
}

func handle(conn net.Conn) {
    // 结束时关闭连接
    defer conn.Close()
    // 读取连接上的数据
    var buf [1024]byte
    len, err := conn.Read(buf[:])
    // 发送数据
    _, err = conn.Write([]byte("I am server!"))
    ...
}
```

Go语言的服务端代码中，主要调用了 net.Listen() 函数和客户端建立了 TCP 连接，此外，还封装并启动了 epoll，将建立的 TCP 连接放入 epoll对象，以监听数据到达情况。

net.Listen() 函数的主要动作是：

1. 创建 socket 并设置非阻塞；
2. bind 绑定并监听本地的一个端口；
3. 调用 listen 开始监听；
4. epoll_create 创建一个 epoll 对象；
5. epoll_etl 将 listen 的监听 socket 添加到 epoll 中等待 TCP 连接的建立；

一次 Go 的 Listen 调用，相当于在 C 语言中的 socket、bind、listen、epoll_create、epoll_etl 等多次函数调用的效果。

![](https://km.woa.com/asset/a747eb82ca914ec1a7f071aabf0c612e?height=1978&width=1450)

_图1.3 Go的Listen函数逻辑_

如图1.3所示，是 Go 的 Listen 函数调用逻辑，从net.Listen() 到 net/sock_posix.go 文件的 socket() 函数调用之间的逻辑，跟客户端的 net.Dial() 逻辑类似，Listen 的 socket() 函数依然会通过 sysSocket() 函数调用 socket() 系统函数创建 socket对象，不同之处是，会通过 netFD.listenStream() 函数分别通过 syscall.Bind 和 listenFunc 调用 bind 和 listen 系统函数，实现服务端的端口绑定和监听，此外，netFD.listenStream() 还会通过 poll/fd_poll_runtime.go 包的 pollDesc.init() 分别调用 netpollinit() 和 netpollopen() 函数创建 epoll对象，并将 listen的 **监听socket** fd 加入到 epoll 对象进行监听，以便在和客户端的 **连接 socket** 建立时得到通知，然后调用 Accept函数获取建立的 **连接socket**。

Go服务器程序在调用完 net.Listen() 函数之后，接着就是调用 Accept() 函数，该函数的主要逻辑是：

1. 调用 accept() 系统函数接收一个TCP连接；
2. 如果连接没有到达，就把当前 Goroutine 阻塞掉；
3. 有客户端连接到来的话，将建立的 **连接socket** 添加到 epoll 对象中管理起来，监听该连接socket 的数据到达情况。

![](https://km.woa.com/asset/0c75ffdd3d78468a8dd3fc609aeed83f?height=1148&width=1284)

_图1.4 Go服务端的 Accept() 函数获取连接逻辑_

如图1.4 所示，是Go服务器端调用 Accept() 函数的主要逻辑。主要是通过 TCPListener.Accept() 函数分别调用 net/fd_unix.go 包的 FD.Accept() 函数和 netFD.init() 函数，FD.Accept()  获取客户端的连接，如果获取不到，则阻塞当前Goroutine，如果获取到了，netFD.init() 将获取的连接放入 epoll对象中管理起来，监控数据的到达。

最后，从 for 循环中的 go handle(conn) 可以看到，服务器端接收客户端请求的逻辑，是**一个 Goroutine 处理一个 TCP 连接**。

# 2. Go原生网络模型netpoll怎样实现对epoll的封装？

把 Go 的 Listen() 和 Accept 函数的内在原理剖析清楚了，再结合epoll的底层原理，很容易弄明白Go原生网络模型netpoll的底层原理。

## 2.1 epoll的基本原理

epoll的底层原理可以参考本人的上一篇文章 [深入学习IO多路复用select/poll/epoll实现原理](https://cloud.tencent.com/developer/article/2176655 "深入学习IO多路复用select/poll/epoll实现原理")。

如图2.1所示，通过 **epoll_create() 函数**创建一个 epoll 对象，通过 **epoll_etl() 函数**将要监听的 socket fd 添加到epoll 对象中，通过 **epoll_wait() 函数**监听 epoll 对象中的socket fd上是否有数据到达，当有数据到达 socket 的等待队列时，**epoll_wait() 函数**通过**回调函数 ep_poll_callback** 找到 eventpoll 对象中红黑树的 epitem 节点，并将其加入就绪列队 rdllist，然后通过**回调函数 default_wake_function** 唤醒用户进程 ，并将 rdllist 传递给用户进程，让用户进程准确读取就绪的 socket 的数据。这种回调机制能够定向准确的通知程序要处理的事件，而不需要每次都循环遍历检查 socket 中的数据是否到达以及数据该由哪个进程处理。

![](https://km.woa.com/asset/b01989dd2a0b4b19bd9dc85d77a9293c?height=2198&width=2914)

_图2.1 epoll实现原理_

简单说明就是，**epoll 多路复用机制，能够通过一个进程高效的监听多个连接 socket 的数据是否到达，到了的就处理，没到的就等待**。  

## 2.2 netpoll的实现原理

如果我们花时间弄明白了 epoll的原理，再去学习 netpoll 的原理，会发现其实很简单，因为 netpoll 的主要操作就是对 epoll 相关函数的封装，只是加上了 Go 特有的高效的 Goroutine 调度器机制，即一个 Goroutine 监听一个 TCP 连接，获取到了数据就执行，否则就阻塞，直到连接的数据到达，再通过 epoll 通知 Goroutine 解除阻塞进入运行状态。

### **2.2.1 netpoll的平台兼容性**

为了提高 I/O 多路复用的性能，不同的操作系统也都实现了自己的 I/O 多路复用函数，例如：epoll、kqueue 和 evport 等。Go 语言为了提高在不同操作系统上的 I/O 操作性能，使用平台特定的函数实现了多个版本的网络轮询模块：

src/runtime/netpoll_epoll.go  
src/runtime/netpoll_kqueue.go  
src/runtime/netpoll_solaris.go  
src/runtime/netpoll_windows.go  
src/runtime/netpoll_aix.go  
src/runtime/netpoll_fake.go

这些模块在不同平台上实现了相同的功能，构成了一个常见的树形结构。编译器在编译 Go 语言程序时，会根据目标平台选择树中特定的分支进行编译：

![](https://km.woa.com/asset/287de24275f14edea6502e7f7f8e50c5?height=830&width=1020)

_图2.2 Go的netpoll网络模型兼容了多个操作系统的IO多路复用的实现_

如果目标平台是 Linux，那么就会根据文件中的 // +build linux 编译指令选择 src/runtime/netpoll_epoll.go 并使用 epoll 函数处理用户的 I/O 操作。

### 2.2.2 netpoll的主要接口

Go中提供了几个接口表达多路复用的基本功能：

```go
func netpollinit()       //  初始化网络轮询器，epoll模块的该函数会调用epoll_create创建epoll对象，通过 sync.Once 和 netpollInited 变量保证函数只会调用一次；
func netpollopen(fd uintptr, pd *pollDesc) int32   // 监听socket fd文件描述符上的数据到达事件；
func netpoll(delay int64) gList    // 阻塞等待返回一组已经准备就绪的 Goroutine
```

要掌握netpoll，只需要搞懂这三个函数的实现机制即可，Go 的netpoll_epoll、netpoll_kqueue 等 IO 多路复用子模块都提供了这几个接口的具体实现。  

对于 netpoll_epoll 子模块的具体实现，可以将 netpollinit() 、netpollopen()、netpollclose()、epoll_wait等函数和 epoll 本身的epoll_create()、epoll_ctl()等函数一一对应上，前者实现了对后者的调用：

![](https://km.woa.com/asset/c7e176671b364df7be98657150cd1f6f?height=542&width=1558)

_图2.3 netpoll_epoll 子模块的函数实现了对epoll本身的函数的调用_

从第1节分析 Go 的 net.Listen() 函数和 Accept() 函数的执行过程可知，在这两个函数中，都会调用 **netpollinit() 函数**创建 epoll对象，由于 netpollinit() 通过 sync.Once 和 netpollInited 变量保证函数只会调用一次，因此 **net.Listen() 函数和 Accept() 函数创建的 epoll 对象是同一个**。然后net.Listen() 函数和 Accept() 函数会调用 **netpollopen()函数**分别将**监听 socket** 和**连接 socket** 通过 **epoll_ctl() 系统函数**加入 epoll 对象中监听起来，**一个监听客户端连接 socket 的建立，一个监听连接 socket 上数据的到达**。 

然后，Go 在多种场景下都可能会调用 **netpoll() 函数**检查 socket 连接的 fd 文件描述符状态，主要是通过 epoll_wait() 函数监听获取数据到达的 socket fd，并找到这些 socket fd 对应的轮询器中附带的信息，根据这些信息将之前等待这些 socket fd 就绪的 Goroutine 状态修改为 _Grunnable ，**执行完 netpoll() 函数之后，会返回一个就绪 fd 列表对应的 Goroutine 列表**，接下来将就绪的 Goroutine 加入到调度队列中，等待调度运行。

Go运行时中调用runtime.netpoll()的地方主要有两处：

1. 在Go调度器中执行 runtime.schedule()，该方法中会执行 runtime.findrunable()，在 runtime.findrunable() 中调用了runtime.netpoll() 获取待执行的 Goroutine；
2. Go runtime 在程序启动的时候会创建一个独立的 sysmon 监控线程，sysmon 每 20us~10ms 运行一次，每次运行会检查距离上一次执行 netpoll() 函数是否超过10ms，如果是则会调用一次 runtime.netpoll()。

### 2.2.3 epoll 和轻量级的Go协程调度器的结合

Go 语言的 netpoll 网络模型比 Linux 的 epoll 多路复用机制更高效的地方在于，和用户态的Go协程调度器结合，实现了高效的网络轮询。

每个 Goroutine 监听一个 TCP 连接，当这个 TCP 连接上没有数据到达时，该 Goroutine 会被 Go运行时调用 gopark 函数给阻塞住，从而进入netpoll 的结构体 pollDesc 的  wait queue 等待队列，G 所在的 M 会尝试获取下一个 _Grunnable 状态的 G 运行。而当G监听的 TCP 连接上有数据到达时，会通过 runtime.netpoll() 调用相关函数解除阻塞，并进入运行队列，等待某个 M 运行它。

![](https://km.woa.com/asset/470ca21075ff45a680dee1e8f7464bb8?height=2270&width=3092)

_图2.4 netpoll网络模型和Go调度器的结合_

这里，netpoll有两个高性能的点：一是 G 监听 TCP 连接，当数据没有到达时不会陷入内核态，也不会让和它一起绑定的 M 陷入阻塞，而只是在用户态进行协程 G 的切换，二是 Goroutine 协程的切换开销大约只有线程 M 切换的三十分之一，极大的提高了网络监听的效率。  

# 3. netpoll的不足

Go原生网络模型 netpoll 通过一个 Goroutine 监听一个 TCP 连接，Goroutine-per-connection 这种模式虽然简单高效，但是在某些极端的场景下也会暴露出问题：Goroutine 虽然非常轻量，它的自定义栈内存初始值仅为 2KB，后面按需扩容；海量连接的业务场景下， Goroutine-per-connection ，此时 Goroutine 数量以及消耗的资源就会呈线性趋势暴涨，首先给 Go 运行时调度器造成极大的压力和侵占系统资源，然后资源被侵占又反过来影响Go 运行时的调度，导致性能大幅下降；此外，我们通过源码可以知道，Go netpoll 会通过 sync.Once 确保只初始化一个 epoll 实例，也就是说它是 single event-loop 模式，接受新连接和处理 I/O 事件是全部放在一个线程里的，所以在海量连接同时又高频创建和销毁连接的业务场景下有可能会导致性能瓶颈。

能够优化这一问题的解决方案，目前了解到的是Go开源社区的网络框架gnet、Netpoll(字节开源)等几个，这里先不多赘述，有空咱们再学习下。

# 4. 总结

本文主要分析了，Go 怎样通过 Listen() 函数和 Accept() 函数实现对底层 socket、bind、listen、accept 函数及 epoll操作函数 epoll_create、epoll_ctl 等的封装，以建立socket连接，并将 socket fd加入 epoll对象中监听起来。

此外，要弄懂 Go 原生网络模型netpoll，需要先了解 epoll 底层原理和 Go 调度器，netpoll 只是二者的结合。

# References

在 golang 中是如何对 epoll 进行封装的？       [https://mp.weixin.qq.com/s/hjWhh_zHfxmH1yZFfvu_zA](https://mp.weixin.qq.com/s/hjWhh_zHfxmH1yZFfvu_zA)

图解 | 深入揭秘 epoll 是如何实现 IO 多路复用的！ [https://mp.weixin.qq.com/s/OmRdUgO1guMX76EdZn11UQ](https://mp.weixin.qq.com/s/OmRdUgO1guMX76EdZn11UQ)

Go 语言设计与实现 6.6 网络轮询器    [https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-netpoller/](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-netpoller/)

Go netpoll I/O 多路复用构建原生网络模型之源码深度解析    [https://www.infoq.cn/article/boeavgkiqmvcj8qjnbxk](https://www.infoq.cn/article/boeavgkiqmvcj8qjnbxk)

详解Go语言I/O多路复用netpoller模型   [https://www.luozhiyun.com/archives/439](https://www.luozhiyun.com/archives/439)

能将三次握手理解到这个深度，面试官拍案叫绝！  [https://mp.weixin.qq.com/s/vlrzGc5bFrPIr9a7HIr2eA](https://mp.weixin.qq.com/s/vlrzGc5bFrPIr9a7HIr2eA)

go源码解析之TCP连接（一）——Listen   [https://www.jianshu.com/p/8e41a7aa5f07](https://www.jianshu.com/p/8e41a7aa5f07)