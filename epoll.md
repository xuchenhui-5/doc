#为什么是epoll 
##定义
按照man手册的说法：是为处理大批量句柄而作了改进的poll。当然，这不是2.6内核才有的，它是在2.5.44内核中被引进的(epoll(4) is a new API introduced in Linux kernel 2.5.44)，它几乎具备了之前所说的一切优点，被公认为Linux2.6下性能最好的多路I/O就绪通知方法。  
##与select/poll的对比
###select/poll的缺点 
* 单个进程能够监视的文件描述符的数量存在最大限制，通常是1024，当然可以更改数量，但由于select采用轮询的方式扫描文件描述符，文件描述符数量越多，性能越差；(在linux内核头文件中，有这样的定义：#define __FD_SETSIZE    1024)
* 内核 / 用户空间内存拷贝问题，select需要复制大量的句柄数据结构，产生巨大的开销；
* select返回的是含有整个句柄的数组，应用程序需要遍历整个数组才能发现哪些句柄发生了事件；
* select的触发方式是水平触发，应用程序如果没有完成对一个已经就绪的文件描述符进行IO操作，那么之后每次select调用还是会将这些文件描述符通知进程。
* 相比select模型，poll使用链表保存文件描述符，因此没有了监视文件数量的限制，但其他三个缺点依然存在。  

>拿select模型为例，假设我们的服务器需要支持100万的并发连接，则在__FD_SETSIZE 为1024的情 况下，则我们至少需要开辟1k个进程才能实现100万的并发连接。除了进程间上下文切换的时间消耗外，从内核/用户空间大量的无脑内存拷贝、数组轮询等，是系统难以承受的。因此，基于select模型的服务器程序，要达到10万级别的并发访问，是一个很难完成的任务。


###epoll优点   
*  **支持一个进程打开大数目的socket描述符(FD):**
    select 最不能忍受的是一个进程所打开的FD是有一定限制的，由FD_SETSIZE设置。对于那些需要支持的上万连接数目的IM服务器来说显然太少了。这时候你一是可以选择修改这个宏然后重新编译内核，不过资料也同时指出这样会带来网络效率的下降，二是可以选择多进程的解决方案(传统的 Apache方案)，不过虽然linux上面创建进程的代价比较小，但仍旧是不可忽视的，加上进程间数据同步远比不上线程间同步的高效，所以也不是一种完美的方案。不过 epoll则没有这个限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于1024,举个例子,在1GB内存的机器上大约是10万左右，具体数目可以cat /proc/sys/fs/file-max察看,一般来说这个数目和系统内存关系很大。
 
* **IO效率不随FD数目增加而线性下降:**
    传统的select/poll另一个致命弱点就是当你拥有一个很大的socket集合，不过由于网络延时，任一时间只有部分的socket是"活跃"的，但是select/poll每次调用都会线性扫描全部的集合，导致效率呈现线性下降。但是epoll不存在这个问题，它只会对"活跃"的socket进行操作---这是因为在内核实现中epoll是根据每个fd上面的callback函数实现的。那么，只有"活跃"的socket才会主动的去调用 callback函数，其他idle状态socket则不会，在这点上，epoll实现了一个"伪"AIO，因为这时候推动力在os内核。在一些 benchmark中，如果所有的socket基本上都是活跃的---比如一个高速LAN环境，epoll并不比select/poll有什么效率，相反，如果过多使用epoll_ctl,效率相比还有稍微的下降。但是一旦使用idle connections模拟WAN环境,epoll的效率就远在select/poll之上了。
 
* **使用mmap加速内核与用户空间的消息传递:**
    这点实际上涉及到epoll的具体实现了。无论是select,poll还是epoll都需要内核把FD消息通知给用户空间，如何避免不必要的内存拷贝就很重要，在这点上，epoll是通过内核于用户空间mmap同一块内存实现的。而如果你想我一样从2.5内核就关注epoll的话，一定不会忘记手工 mmap这一步的。
 
* **内核微调:**
这一点其实不算epoll的优点了，而是整个linux平台的优点。也许你可以怀疑linux平台，但是你无法回避linux平台赋予你微调内核的能力。比如，内核TCP/IP协议栈使用内存池管理sk_buff结构，那么可以在运行时期动态调整这个内存pool(skb_head_pool)的大小--- 通过echo XXXX>/proc/sys/net/core/hot_list_length完成。再比如listen函数的第2个参数(TCP完成3次握手的数据包队列长度)，也可以根据你平台内存大小动态调整。更甚至在一个数据包面数目巨大但同时每个数据包本身大小却很小的特殊系统上尝试最新的NAPI网卡驱动架构。  

##epoll使用  
epoll只有epoll_create,epoll_ctl,epoll_wait 3个系统调用。  

1. **int epoll_create(int size):**
创建一个epoll的句柄。自从linux2.6.8之后，size参数是被忽略的。需要注意的是，当创建好epoll句柄后，它就是会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。
 
2. **int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event):**
epoll的事件注册函数，它不同于select()是在监听事件时告诉内核要监听什么类型的事件，而是在这里先注册要监听的事件类型。第一个参数是epoll_create()的返回值。
第二个参数表示动作，用三个宏来表示：  
EPOLL_CTL_ADD：注册新的fd到epfd中；  
EPOLL_CTL_MOD：修改已经注册的fd的监听事件；
EPOLL_CTL_DEL：从epfd中删除一个fd；  
第三个参数是需要监听的fd。 第四个参数是告诉内核需要监听什么事  
```
struct epoll_event结构如下：
//保存触发事件的某个文件描述符相关的数据（与具体使用方式有关）  
typedef union epoll_data {  
    void *ptr;  
    int fd;  
    __uint32_t u32;  
    __uint64_t u64;  
} epoll_data_t;  
 //感兴趣的事件和被触发的事件  
struct epoll_event {  
    __uint32_t events; /* Epoll events */  
    epoll_data_t data; /* User data variable */  
};  
```  
events可以是以下几个宏的集合：  
EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；  
EPOLLOUT：表示对应的文件描述符可以写；  
EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；  
EPOLLERR：表示对应的文件描述符发生错误；  
EPOLLHUP：表示对应的文件描述符被挂断；  
EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。  
EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里。    
3. int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
收集在epoll监控的事件中已经发送的事件。参数events是分配好的epoll_event结构体数组，epoll将会把发生的事件赋值到events数组中（events不可以是空指针，内核只负责把数据复制到这个events数组中，不会去帮助我们在用户态中分配内存）。maxevents告之内核这个events有多大，这个 maxevents的值不能大于创建epoll_create()时的size，参数timeout是超时时间（毫秒，0会立即返回，-1将不确定，也有说法说是永久阻塞）。如果函数调用成功，返回对应I/O上已准备好的文件描述符数目，如返回0表示已超时。 

##实现原理  
* epoll比select/poll的一个优势：select/poll每次调用都要传递所要监控的所有fd给select/poll系统调用（这意味着每次调用都要将fd列表从用户态拷贝到内核态，当fd数目很多时，这会造成低效）。而每次调用epoll_wait时（作用相当于调用select/poll），不需要再传递fd列表给内核，因为已经在epoll_ctl中将需要监控的fd告诉了内核（epoll_ctl不需要每次都拷贝所有的fd，只需要进行增量式操作）。所以，在调用epoll_create之后，内核已经在内核态开始准备数据结构存放要监控的fd了。每次epoll_ctl只是对这个数据结构进行简单的维护。
 
* 此外，内核使用了slab机制，为epoll提供了快速的数据结构：
在内核里，一切皆文件。所以，epoll向内核注册了一个文件系统(文件系统一般用什么数据结构实现？B+树)，用于存储上述的被监控的fd。当你调用epoll_create时，就会在这个虚拟的epoll文件系统里创建一个file结点。当然这个file不是普通文件，它只服务于epoll。epoll在被内核初始化时（操作系统启动），同时会开辟出epoll自己的内核高速cache区，用于安置每一个我们想监控的fd，这些fd会以红黑树的形式保存在内核cache里，以支持快速的查找、插入、删除。这个内核高速cache区，就是建立连续的物理内存页，然后在之上建立slab层，简单的说，就是物理上分配好你想要的size的内存对象，每次使用时都是使用空闲的已分配好的对象。
 
* epoll的第三个优势在于：当我们调用epoll_ctl往里塞入百万个fd时，epoll_wait仍然可以飞快的返回，并有效的将发生事件的fd给我们用户。这是由于我们在调用epoll_create时，内核除了帮我们在epoll文件系统里建了个file结点，在内核cache里建了个红黑树用于存储以后epoll_ctl传来的fd外，还会再建立一个list链表，用于存储准备就绪的事件，当epoll_wait调用时，仅仅观察这个list链表里有没有数据即可。有数据就返回，没有数据就sleep，等到timeout时间到后即使链表没数据也返回。所以，epoll_wait非常高效。而且，通常情况下即使我们要监控百万计的fd，大多一次也只返回很少量的准备就绪fd而已，所以，epoll_wait仅需要从内核态copy少量的fd到用户态而已。那么，这个准备就绪list链表是怎么维护的呢？当我们执行epoll_ctl时，除了把fd放到epoll文件系统里file对象对应的红黑树上之外，还会给内核中断处理程序注册一个回调函数，告诉内核，如果这个fd的中断到了，就把它放到准备就绪list链表里。所以，当一个fd（例如socket）上有数据到了，内核在把设备（例如网卡）上的数据copy到内核中后就来把fd（socket）插入到准备就绪list链表里了。
如此，一颗红黑树，一张准备就绪fd链表，少量的内核cache，就帮我们解决了大并发下的fd（socket）处理问题。
>1. 执行epoll_create时，创建了红黑树和就绪list链表。  
>2. 执行epoll_ctl时，如果增加fd（socket），则检查在红黑树中是否存在，存在立即返回，不存在则添加到红黑树上，然后向内核注册回调函数，用于当中断事件来临时向准备就绪list链表中插入数据。
>3. 执行epoll_wait时立刻返回准备就绪链表里的数据即可。   

##epoll工作模式  
* **LT模式：**Level Triggered水平触发这个是缺省的工作模式。同时支持block socket和non-block socket。内核会告诉程序员一个文件描述符是否就绪了。如果程序员不作任何操作，内核仍会通知。
 
* **ET模式：**Edge Triggered 边缘触发是一种高速模式。仅当状态发生变化的时候才获得通知。这种模式假定程序员在收到一次通知后能够完整地处理事件，于是内核不再通知这一事件。注意：缓冲区中还有未处理的数据不算状态变化，所以ET模式下程序员只读取了一部分数据就再也得不到通知了，正确的用法是程序员自己确认读完了所有的字节（一直调用read/write直到出错EAGAIN为止）。 
如下图： ![](2.jpg)
0：表示文件描述符未准备就绪
1：表示文件描述符准备就绪.  
*对于水平触发模式(LT)：*在1处，如果你不做任何操作，内核依旧会不断的通知进程文件描述符准备就绪。  
*对于边缘出发模式(ET):*只有在0变化到1处的时候，内核才会通知进程文件描述符准备就绪。之后如果不在发生文件描述符状态变化，内核就不会再通知进程文件描述符已准备就绪。  
Nginx 默认采用的就是ET。
##实例  
```
 1: #include <stdio.h>
   2: #include <stdlib.h>
   3: #include <unistd.h>
   4: #include <sys/socket.h>
   5: #include <errno.h>
   6: #include <sys/epoll.h>
   7: #include <netinet/in.h>
   8: #include <fcntl.h>
   9: #include <string.h>
  10:  #include <netdb.h>
  11:  
  12:  
  13:  
  14: struct epoll_event  *events = NULL;
  15: int epollFd = -1;
  16:  
  17: const int MAX_SOCK_NUM = 1024;
  18:  
  19:  
  20: int epoll_init();
  21: int epoll_socket(int domain, int type, int protocol);
  22: int epoll_cleanup();
  23: int epoll_new_conn(int sfd);
  24:  
  25:  
  26: int main()
  27: {
  28:       struct sockaddr_in listenAddr;
  29:       int listenFd = -1;
  30:  
  31:       if(-1 == epoll_init())
  32:       {
  33:           printf("epoll_init err\n");
  34:           return -1;
  35:       }
  36:  
  37:       if((listenFd = epoll_socket(AF_INET,SOCK_STREAM,0)) == -1)
  38:       {
  39:           printf("epoll_socket err\n");
  40:           epoll_cleanup();
  41:           return -1;
  42:       }
  43:  
  44:       listenAddr.sin_family = AF_INET;
  45:       listenAddr.sin_port = htons(999);
  46:       listenAddr.sin_addr.s_addr = htonl(INADDR_ANY);
  47:  
  48:       if(-1 == bind(listenFd,(struct sockaddr*)&listenAddr,sizeof(listenAddr)))
  49:       {
  50:           printf("bind err %d\n",errno);
  51:           epoll_cleanup();
  52:           return -1;
  53:       }
  54:  
  55:       if(-1 == listen(listenFd,1024))
  56:       {
  57:           printf("listen err\n");
  58:           epoll_cleanup();
  59:           return -1;
  60:       }
  61:  
  62:       //Add ListenFd into epoll
  63:       if(-1 == epoll_new_conn(listenFd))
  64:       {
  65:           printf("eph_new_conn err\n");
  66:           close(listenFd);
  67:         epoll_cleanup();
  68:         return -1;
  69:       }
  70:  
  71:  
  72:       //LOOP
  73:       while(1)
  74:       {
  75:           int n;
  76:           n = epoll_wait(listenFd,events,MAX_SOCK_NUM,-1);
  77:           for (int i = 0; i < n; i++)
  78:           {
  79:                if( (events[i].events & EPOLLERR) || ( events[i].events & EPOLLHUP ) || !(events[i].events & EPOLLIN) )
  80:                {
  81:                    printf("epoll err\n");
  82:                    close(events[i].data.fd);
  83:                    continue;
  84:                }
  85:                else if(events[i].data.fd == listenFd)
  86:                {
  87:                    while(1)
  88:                    {
  89:                        struct sockaddr inAddr;
  90:                        char hbuf[1024],sbuf[NI_MAXSERV];
  91:                        socklen_t inLen = -1;
  92:                        int inFd = -1;
  93:                        int s = 0;
  94:                        int flag = 0;
  95:  
  96:                        inLen = sizeof(inAddr);
  97:                        inFd = accept(listenFd,&inAddr,&inLen);
  98:  
  99:                        if(inFd == -1)
 100:                        {
 101:                            if( errno == EAGAIN || errno == EWOULDBLOCK )
 102:                            {
 103:                                break;
 104:                            }
 105:                            else
 106:                            {
 107:                                printf("accept error\n");
 108:                                break;
 109:                            }
 110:                        }
 111:  
 112:                     if (s ==  getnameinfo (&inAddr, inLen, hbuf, sizeof(hbuf), sbuf, sizeof(sbuf), NI_NUMERICHOST | NI_NUMERICSERV)) 
 113:                     {
 114:                         printf("Accepted connection on descriptor %d (host=%s, port=%s)\n", inFd, hbuf, sbuf);
 115:                     }
 116:  
 117:                     //Set Socket to non-block
 118:                     if((flag = fcntl(inFd,F_GETFL,0)) < 0 || fcntl(inFd,F_SETFL,flag | O_NONBLOCK) < 0)
 119:                     {
 120:                         close(inFd);
 121:                         return -1;
 122:                     }
 123:  
 124:                     epoll_new_conn(inFd);
 125:                    }
 126:                }
 127:                else
 128:                {
 129:                         while (1) 
 130:                         {
 131:                         ssize_t count;
 132:                         char buf[512];
 133:  
 134:                         count = read (events[i].data.fd, buf, sizeof buf);
 135:  
 136:                         if (count == -1) 
 137:                         {
 138:                             if (errno != EAGAIN)
 139:                              { 
 140:                                 printf("read err\n");
 141:                                 }
 142:  
 143:                             break;
 144:  
 145:                         } 
 146:                         else if (count == 0) 
 147:                         {  
 148:                             break;
 149:                         }
 150:  
 151:                         write (1, buf, count); 
 152:                     }
 153:                 }
 154:           }
 155:  
 156:       }
 157:  
 158:       epoll_cleanup();
 159: }
 160:  
 161:  
 162: int epoll_init()
 163: {
 164:     if(!(events = (struct epoll_event* ) malloc ( MAX_SOCK_NUM * sizeof(struct epoll_event))))
 165:     {
 166:         return -1;
 167:     }
 168:  
 169:     if( (epollFd = epoll_create(MAX_SOCK_NUM)) < 0 )
 170:     {
 171:         return -1;
 172:     }
 173:  
 174:     return 0;
 175: }
 176:  
 177: int epoll_socket(int domain, int type, int protocol)
 178: {
 179:     int sockFd = -1;
 180:     int flag = -1;
 181:  
 182:     if ((sockFd = socket(domain,type,protocol)) < 0)
 183:     {
 184:         return -1;
 185:     }
 186:  
 187:     //Set Socket to non-block
 188:     if((flag = fcntl(sockFd,F_GETFL,0)) < 0 || fcntl(sockFd,F_SETFL,flag | O_NONBLOCK) < 0)
 189:     {
 190:         close(sockFd);
 191:         return -1;
 192:     }
 193:  
 194:     return sockFd;
 195: }
 196:  
 197: int epoll_cleanup()
 198: {
 199:     free(events);
 200:     close(epollFd);
 201:     return 0;
 202: }
 203:  
 204: int epoll_new_conn(int sfd)
 205: {
 206:  
 207:       struct epoll_event  epollEvent;
 208:       memset(&epollEvent, 0, sizeof(struct epoll_event));
 209:       epollEvent.events = EPOLLIN | EPOLLERR | EPOLLHUP | EPOLLET;
 210:       epollEvent.data.ptr = NULL;
 211:       epollEvent.data.fd  = sfd;
 212:  
 213:       if (epoll_ctl(epollFd, EPOLL_CTL_ADD, sfd, &epollEvent) < 0)
 214:       {
 215:         return -1;
 216:       }
 217:  
 218:     epollEvent.data.fd  = sfd;
 219:  
 220:     return 0;
 221: }
```  
---
此文来自于：  
<http://www.open-open.com/lib/view/open1410403215664.html>  
<http://blog.csdn.net/xiajun07061225/article/details/9250579>   
<http://blog.csdn.net/chen19870707/article/details/42525887>


