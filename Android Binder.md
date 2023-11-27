

# Android Binder



## 掌握知识点

## 1.虚拟内存

虚拟内存的重要意义是它定义了一个连续的虚拟地址空间,使得程序的编写难度降低。虚拟内存是计算机系统内存管理的一种技术。它使得应用程序认为它拥有连续可用的内存（一个连续完整的地址空间），而实际上，它通常是被分隔成多个物理内存碎片，还有部分暂时存储在外部磁盘存储器上，在需要时进行数据交换。

在没有虚拟内存的时候，程序完全被装载进物理内存中，那么 1G 的存储空间永远只能执行固定个数的应用程序，一旦内存占满，就要触发 swap 和磁盘交换数据。而且每个应用程序大小不一，比如程序 A 占 4M 空间，程序 B 需要 6M 空间，即使 A 被释放了，B 也无法使用这 4M 的空间。
而在虚拟内存中，首先并不是将程序所需所有数据一次性装载进内存，而是需要什么数据才会装载什么数据。其次，虚拟内存以页为单位 (常规 4KB) 同物理内存最小单位一致，那么就可以减少碎片化的问题，可以给程序分配 4KB 为单位的非连续的物理空间，但是虚拟内存中却看起来是连续的。

## 2. Linux进程空间

我们前面提到了虚拟内存的概念，而 Linux 进程空间就是通过虚拟内存来实现的。我们知道现在操作系统都是采用虚拟存储器，那么对32位操作系统而言，它的寻址空间（虚拟存储空间）为4G（2的32次方）。操心系统的核心是内核，独立于普通的应用程序，可以访问受保护的内存空间，也有访问底层硬件设备的所有权限。为了保证用户进程不能直接操作内核，保证内核的安全，操心系统将虚拟空间划分为两部分，一部分为内核空间，一部分为用户空间。
针对 Linux 操作系统而言，将最高的1G字节（从虚拟地址0xC0000000到0xFFFFFFFF），供内核使用，称为内核空间，而将较低的3G字节（从虚拟地址0x00000000到0xBFFFFFFF），供各个进程使用，称为用户空间。每个进程可以通过系统调用进入内核，因此，Linux 内核由系统内的所有进程共享。于是，从具体进程的角度来看，每个进程可以拥有4G字节的虚拟空间，用户进程可以通过系统调用访问到内核空间。

![image-20231128005109661](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20231128005109661.png)

## 3.进程隔离

进程隔离简单的说就是 Linux 操作系统设计的一种机制，使进程之间不能共享数据，保持各自数据的独立性，即A进程不能访问B进程数据，同理B进程也不能访问A进程数据。通过虚拟内存技术，达到 Linux 进程中数据不能共享，从而保持独立的功能。所以，Linux 进程之间要进行数据交互就得采用特殊的通信机制，即 IPC 通信！



## 4. 用户空间向内核空间拷贝数据的方式

+ copy_from_user() 
+ get_user()

![image-20231128005523212](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20231128005523212.png)

## 5. 内核空间向用户空间拷贝数据的方式

+ copy_to_user()
+ put_user()

![image-20231128005742009](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20231128005742009.png)



## 6. 内存映射mmap()系统调用函数

mmap函数可以将一个文件、一段物理内存或者其它对象（也包括内核空间）映射到进程的虚拟内存地址空间。在binder通信中，实际上就是服务端的用户空间某段虚拟内存和内核空间某段虚拟内存同时映射到了同一块物理内存。

## 7. binder驱动

 Binder 驱动并不是 Linux 系统标准内核的一部分，那怎么实现加载和使用呢？这就得益于 Linux 的动态内核可加载模块（Loadable Kernel Module，LKM）的机制；模块是具有独立功能的程序，它可以被单独编译，但是不能独立运行。它在运行时被链接到内核作为内核的一部分运行。这样，Android 系统就可以通过动态添加一个内核模块运行在内核空间，用户进程之间通过这个内核模块作为桥梁来实现通信。

Binder 驱动是一种虚拟的字符设备，重点它是虚拟出来的，尽管名叫“驱动”，实际上 Binder 驱动并没有操作相关的硬件，只是实现方式和设备驱动程序是一样的：它工作于内核态，提供open()，mmap()，poll()，ioctl()等标准文件操作，以字符驱动设备中的misc设备注册在设备目录/dev下，用户通过/dev/binder访问该它。
驱动负责进程之间 Binder 通信的建立，Binder 在进程之间的传递，Binder 引用计数管理，数据包在进程之间的传递和交互等一系列底层支持。驱动和应用程序之间定义了一套接口协议，主要功能由 ioctl() 接口实现，不提供 read()，write() 接口，因为 ioctl() 灵活方便，且能够一次调用实现先写后读以满足同步交互，而不必分别调用 write() 和 read()。

当然 Binder 驱动是 Android 特有的，所以它不是 Linux 标准内核携带的，必须通过动态内核加载进行加载的。既然 Binder 驱动属于内核层，所以应用层对于它访问也是通过系统调用实现的。



## 8. binder机制架构

+ 对于binder服务端(Server)：
  1. Binder 服务端(Service)在启动之后，通过系统调用 Binde 驱动在内核空间创建一个数据接收缓存区（调用 binder_oepn 方法执行相关的操作）。
  2. 接着 Binder 服务端进程空间的内核接收到系统调用的指令，进而调用 mmap 函数进行对应的处理。首先申请一块物理内存，然后建立 Binder 服务端(Service端)的用户空间和内核空间一块区域的映射关系(这样上述两块区域就映射在一起了)。
+ 对于请求端(Client):
  1. Client 向服务端发送通信发送请求，这个请求数据打包完毕之后通过系统调用先到驱动中，然后在驱动中通过 copy_from_user() 将数据从用户空间拷贝到内核空间的缓存区中(注意这块内核空间和 Binder 服务端的用户空间存在映射关系)。
  2. 由于内核缓存区和接收进程的用户空间存在内存映射，因此也就相当于把数据发送到了接收进程的用户空间（只需要进行一定的偏移操作即可），这样便完成了一次进程间的通信，从而省去了一次拷贝的操作。

![image-20231128011209937](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20231128011209937.png)



Binder是Android提供的一种IPC通信方式，在基于C/S架构体系中，除了架构所包含的Client端和Server端外，Android还有一个全局的ServiceManager，它的作用是管理系统中各种服务(Service)。

![image-20231128011413258](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20231128011413258.png)

​	

三者关系：
+ ServiceManager是Server的服务端：Server调用ServiceManager接口向ServiceManager注册服务；
+ ServiceManager是Client的服务端
+ Server是Client的服务端



```c++
/frameworks/native/libs/binder/ProcessState.cpp
    
```



![在这里插入图片描述](https://img-blog.csdnimg.cn/a86ce68c70ad4202891514b69befc009.png)

**以MediaServer为切入点，介绍如何将MediaPlayerService服务注册到ServerManager中。**

## 一、业务层：将数据打包

​	首先看main_mediaserver.cpp内的main函数：

```c++
int main(int argc __unused, char **argv __unused)
{
    //1.获得一个processState实例
    sp<ProcessState> proc(ProcessState::self());
    
    //2.调用defaultServiceManager函数，获得一个IServrceManger对象，用于向服务端ServiceManger注册MediaPlayerService
    sp<IServiceManager> sm(defaultServiceManager());
    
    //3.向ServiceManager注册MediaPlayerService
    MediaPlayerService::instantiate();
    
    
    ProcessState::self()->startThreadPool();
    IPCThreadState::self()->joinThreadPool();
}
```



通过以上注释几点，逐一分析，理解binder：

### 1.每个进程只存在一个PorcessState对象

​	每个进程只存在一个PorcessState对象，main函数开始处执行了以下代码：

```c++
//1.获得一个processState实例
    sp<ProcessState> proc(ProcessState::self());
```

进入ProcessState::self()函数看看具体做了什么：

```c++
sp<ProcessState> ProcessState::self()
{
    // gProcess在Static.cpp中定义的全局变量
    //程序刚开始执行，gProcess一定为空
    if (gProcess != NULL) {
        return gProcess;
    }
    
    //创建一个ProcessState对象，并赋值给gProcess。下面接着看ProcessState的构造函数
    gProcess = new ProcessState("/dev/binder");
    return gProcess;
}
```

ProcessState::self()函数用了单例模式，所以一个进程只会有一个Process对象。

现在开始看ProcessState的构造函数:

```c++
ProcessState::ProcessState(const char *driver)
    : mDriverName(String8(driver))
      //打开“dev/binder”设备,driver是在调用ProcessState构造函数传进来的
      //下面接着分析open_driver(driver)函数
    , mDriverFD(open_driver(driver))
    , mVMStart(MAP_FAILED)//映射内存的起始位置
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mStarvationStartTimeMs(0)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
{
    if (mDriverFD >= 0) {
        //调用mmap函数内存映射
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        
    }
}
```



上面的ProcessState的构造函数调用了open_driver(driver)函数，看下里面做了什么：

```c++
//ProcessState.cpp
static int open_driver(const char *driver)
{
    //driver = “/dev/binder”,这里打开了/dev/binder设备
    int fd = open(driver, O_RDWR | O_CLOEXEC);
    if (fd >= 0) 
        //通过ioctl方式告诉binder驱动，这个fd支持的最大线程数为DEFAULT_MAX_BINDER_THREADS
        size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;
        result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
       
    } 
//返回fd文件描述符，将fd作为参数调用mmap()使数据接收方用户空间(服务端、当前进程)与内核空间同时映射到一块物理内存
    return fd;
}
```



#### 总结：

ProcessState::self()函数总结完了，看下具体做了什么：

1. 打开/dev/binder设备，返回binder驱动的fd文件描述符，为内存映射做准备。
2. 对返回的fd使用mmap，使服务端的某段虚拟内存和binder驱动程序的某段虚拟内存映射到同一块物理内存。

注意事项：

+ open_driver()放到了最后说明，但它是在mmap()执行的。
+ ProcessState是单例的所以一个进程只会打开一次binder设备和内存映射。
+ 发生内存映射的是在服务端，上诉只是说明当前进程是可以作为服务端的。而下面将讲解的是该进程注册服务，说明当前进程是要传递数据注册服务，该场景是说明当前进程作为客户端。



