使用Socket接口编程
------------------

什么是Socket
~~~~~~~~~~~~

Socket英文原意是“孔”或者“插座”的意思，在网络编程中，通常将其称之为“套接字”，当前网络中的主流程序设计都是使用Socket进行编程的，因为它简单易用，更是一个标准，能在不同平台很方便移植。本章讲解的是LwIP中的Socket编程接口，因为LwIP作者为了能让更多开发者直接上手LwIP的编程，专门设计了LwIP的第三种编程接口——Socket
API，它兼容BSD Socket。

Socket虽然是能在多平台移植，但是LwIP中的Socket并不完善，因为LwIP设计之初就是为了在嵌入式平台中使用，它只实现了完整Socket的部分功能，不过，在嵌入式平台中，这些功能早已足够。

在Socket中，它使用一个套接字来记录网络的一个连接，套接字是一个整数，就像我们操作文件一样，利用一个文件描述符，可以对它打开、读、写、关闭等操作，类似的，在网络中，我们也可以对Socket套接字进行这样子的操作，比如开启一个网络的连接、读取连接主机发送来的数据、向连接的主机发送数据、终止连接等操作。

LwIP中的Socket
~~~~~~~~~~~~~~

在LwIP中，Socket API是基于NETCONN
API之上来实现的，系统最多提供MEMP_NUM_NETCONN
个netconn连接结构，因此Socket套接字的个数也是那么多个，为了更好对netconn进行封装，LwIP还定义了一个套接字结构体——lwip_sock（我称之为Socket连接结构），每个lwip_sock内部都有一个netconn的指针，实现了对netconn的再次封装，那怎么找到lwip_sock这个结构体呢？LwIP定义了一个lwip_sock
类型的sockets数组，通过套接字就可以直接索引并且访问这个结构体了，这也是为什么套接字是一个整数的原因，lwip_sock结构体是比较简单的，因为基本上全是依赖netconn实现，具体见 代码清单16_1_。

代码清单 16‑1 LwIP中的Socket相关数据结构

.. code-block:: c
   :name: 代码清单16_1

    #define NUM_SOCKETS MEMP_NUM_NETCONN

    /** 全局可用套接字数组（默认是4） */
    static struct lwip_sock sockets[NUM_SOCKETS];

    union lwip_sock_lastdata
    {
        struct netbuf *netbuf;
        struct pbuf *pbuf;
    };

    /** 包含用于套接字的所有内部指针和状态 */
    struct lwip_sock
    {
        /** 套接字当前是在netconn上构建的，每个套接字都有一个netconn */
        struct netconn *conn;
        /** 从上一次读取中留下的数据 */
        union lwip_sock_lastdata lastdata;
        /** 收到数据的次数由event_callback()记录，下面的字段在select机制上使用 */
        s16_t rcvevent;
        /** 发送数据的次数，也是由回调函数记录的 */
        u16_t sendevent;
        /** Socket上的发生的错误次数 */
        u16_t errevent;
        /** 使用select等待此套接字的线程数 */
        SELWAIT_T select_waiting;
    };

Socket API
~~~~~~~~~~

socket()
^^^^^^^^

这个函数的功能是向内核申请一个套接字，在本质上该函数其实就是对netconn_new()函数进行了封装，虽然说不是直接调用它，但是主体完成的工作就做了
netconn_new()函数的事情，而且该函数本质是一个宏定义，具体见 代码清单16_2_。

代码清单 16‑2 socket()

.. code-block:: c
   :name: 代码清单16_2

    #define socket(domain,type,protocol)        \
        lwip_socket(domain,type,protocol)

    int
    lwip_socket(int domain, int type, int protocol);

    #define AF_INET         2

    /* Socket服务类型 (TCP/UDP/RAW) */
    #define SOCK_STREAM     1
    #define SOCK_DGRAM      2
    #define SOCK_RAW        3

参数domain表示该套接字使用的协议簇，对于TCP/IP协议来说，该值始终为AF_INET。

参数type指定了套接字使用的服务类型，可能的类型有3种：

1. SOCK_STREAM：提供可靠的（即能保证数据正确传送到对方）面向连接的Socket服务，多用于资料（如文件）传输，如TCP协议。

2. SOCK_DGRAM：是提供无保障的面向消息的Socket
   服务，主要用于在网络上发广播信息，如UDP协议，提供无连接不可靠的数据报交付服务。

3. SOCK_RAW：表示原始套接字，它允许应用程序访问网络层的原始数据包，这个套接字用得比较少，暂时不用理会它。

参数protocol指定了套接字使用的协议，在IPv4中，只有TCP协议提供SOCK_STREAM这种可靠的服务，只有UDP协议提供SOCK_DGRAM服务，对于这两种协议，protocol的值均为0。

当申请套接字成功的时候，该函数返回一个int类型的值，也是Socket描述符，用户通过这个值可以索引到一个Socket连接结构——lwip_sock，当申请套接字失败时，该函数返回-1。

bind()
^^^^^^

该函数的功能与netconn_bind()函数是一样的，用于服务器端绑定套接字与网卡信息，
实际上就是对netconn_bind()函数进行了封装，可以将一个申请成功的套接字与网卡信息进行绑定，
其函数原型具体见 代码清单16_3_

代码清单 16‑3 bind()

.. code-block:: c
   :name: 代码清单16_3

    #define bind(s,name,namelen)  \
                        lwip_bind(s,name,namelen)

    int lwip_bind(int s,
                const struct sockaddr *name,
                socklen_t namelen);

参数s是表示要绑定的Socket套接字，注意了，这个套机字必须是从socket()函数中返回的索引，否则将无法完成绑定操作。

参数name是一个指向sockaddr结构体的指针，其中包含了网卡的IP地址、端口号等重要的信息，LwIP为了更好描述这些信息，使用了sockaddr结构体来定义了必要的信息的字段，它常被用于Socket
API的很多函数中，我们在使用bind()的时候，只需要直接填写相关字段即可，sockaddr结构体具体见 代码清单16_4_。

参数namelen指定了name结构体的长度。

代码清单 16‑4sockaddr结构体

.. code-block:: c
   :name: 代码清单16_4

    struct sockaddr
    {
        u8_t        sa_len;	/* 长度 */
        sa_family_t sa_family;	/* 协议簇 */
        char        sa_data[14];	/* 连续的14字节信息 */
    };

咋一看这个结构体，好像没啥信息要我们填写的，确实也是这样子，我们需要填写的IP地址与端口号等信息，都在sa_data连续的14字节信息里面，但是这个数据对我们不友好，因此LwIP还定义了另一个对开发者更加友好的结构体——sockaddr_in，我们一般也是用这个结构体，具体见
代码清单16_5_

代码清单 16‑5sockaddr_in结构体

.. code-block:: c
   :name: 代码清单16_5

    struct sockaddr_in
    {
        u8_t            sin_len;
        sa_family_t     sin_family;
        in_port_t       sin_port;
        struct in_addr  sin_addr;
    #define SIN_ZERO_LEN 8
        char            sin_zero[SIN_ZERO_LEN];
    };

这个结构体的前两个字段是与sockaddr结构体的前两个字段一致，而剩下的字段就是sa_data连续的14字节信息里面的内容，只不过从新定义了成员变量而已，sin_port字段是我们需要填写的端口号信息，sin_addr字段是我们需要填写的IP地址信息，剩下sin_zero
区域的8字节保留未用。

那么这个函数应该怎么使用呢？具体见 代码清单16_6_，

代码清单 16‑6 bind()函数的使用方法

.. code-block:: c
   :name: 代码清单16_6

    sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0)
    {
        printf("Socket error\n");
    }
    struct sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(5001);
    memset(&(server_addr.sin_zero), 0, sizeof(server_addr.sin_zero));

    if(bind(sock,(struct sockaddr *)&server_addr,sizeof(struct sockaddr))==-1)
    {
        printf("Unable to bind\n");
    }

connect()
^^^^^^^^^

这个函数的作用与netconn_connect()函数的作用基本一致，因为就是封装了netconn_connect()函数。它用于客户端中，将Socket与远端IP地址、端口号进行绑定，在TCP客户端连接中，调用这个函数将发生握手过程（会发送一个TCP连接请求），并最终建立新的TCP连接，而对于UDP协议来说，调用这个函数只是在UDP控制块中记录远端IP地址与端口号，而不发送任何数据，参数信息与bind()函数是一样的，具体见
代码清单16_7_。

代码清单 16‑7 connect()

.. code-block:: c
   :name: 代码清单16_7

    #define connect(s,name,namelen) \
                    lwip_connect(s,name,namelen)

    int
    lwip_connect(int s,
                const struct sockaddr *name,
                socklen_t namelen);

listen()
^^^^^^^^

该函数是对netconn_listen()函数的封装，只能在TCP服务器中使用，让服务器进入监听状态，
等待远端的连接请求，LwIP中可以接收多个客户端的连接，因此参数backlog指定了请求队列的大小，具体见 代码清单16_8_。

代码清单 16‑8 listen()

.. code-block:: c
   :name: 代码清单16_8

    #define listen(s,backlog)         \
                    lwip_listen(s,backlog)

    int
    lwip_listen(int s, int backlog);

accept()
^^^^^^^^

accept()函数与netconn_accept()函数作用一样，用于TCP服务器中，等待着远端主机的连接请求，
并且建立一个新的TCP连接，在调用这个函数之前需要通过调用listen()函数让服务器进入监听状态。
accept()函数的调用会阻塞应用线程直至与远程主机建立TCP连接。参数addr是一个返回结果参数，
它的值由accept()函数设置，其实就是远程主机的地址与端口号等信息，当新的连接已经建立后，
远端主机的信息将保存在连接句柄中，它能够唯一的标识某个连接对象。同时函数返回一个int类型的套接字描述符，
根据它能索引到连接结构，如果连接失败则返回-1，具体见 代码清单16_9_。

代码清单 16‑9 accept()

.. code-block:: c
   :name: 代码清单16_9

    #define accept(s,addr,addrlen)      \
                lwip_accept(s,addr,addrlen)
    int
    lwip_accept(int s,
                struct sockaddr *addr,
                socklen_t *addrlen)

read()、recv()、recvfrom()
^^^^^^^^^^^^^^^^^^^^^^^^^^

read()与recv()函数的核心是调用recvfrom()函数，而recvfrom()函数是基于netconn_recv()函数来实现的，
recv()与read()函数用于从Socket中接收数据，它们可以是TCP协议和UDP协议，
具体见 代码清单16_10_。

代码清单 16‑10 read()、recv()、recvfrom()

.. code-block:: c
   :name: 代码清单16_10

    #define read(s,mem,len)         \
            lwip_read(s,mem,len)
    ssize_t
    lwip_read(int s, void *mem, size_t len)
    {
        return lwip_recvfrom(s, mem, len, 0, NULL, NULL);
    }

    #define recv(s,mem,len,flags)           \
            lwip_recv(s,mem,len,flags)
    ssize_t
    lwip_recv(int s, void *mem, size_t len, int flags)
    {
        return lwip_recvfrom(s, mem, len, flags, NULL, NULL);
    }

    #define recvfrom(s,mem,len,flags,from,fromlen)    \
            lwip_recvfrom(s,mem,len,flags,from,fromlen)
    ssize_t
    lwip_recvfrom(int s, void *mem, size_t len, int flags,
                struct sockaddr *from, socklen_t *fromlen)

men参数记录了接收数据的缓存起始地址，len用于指定接收数据的最大长度，如果函数能正确接收到数据，将会返回一个接收到数据的长度，否则将返回-1，若返回值为0，表示连接已经终止，应用程序可以根据返回的值进行不一样的操作。recv()函数包含一个flags参数，我们暂时可以直接忽略它，设置为0即可。注意，如果接收的数据大于用户提供的缓存区，那么多余的数据会被直接丢弃。

sendto()
^^^^^^^^

这个函数主要是用于UDP协议传输数据中，它向另一端的UDP主机发送一个UDP报文，本质上是对netconn_send()函数的封装，
参数data指定了要发送数据的起始地址，而size则指定数据的长度，参数flag指定了发送时候的一些处理，比如外带数据等，
此时我们不需要理会它，一般设置为0即可，参数to是一个指向sockaddr结构体的指针，
在这里需要我们自己提供远端主机的IP地址与端口号，并且用tolen参数指定这些信息的长度，具体见 代码清单16_11_。

代码清单 16‑11 sendto()

.. code-block:: c
   :name: 代码清单16_11

    #define sendto(s,dataptr,size,flags,to,tolen)     \
    lwip_sendto(s,dataptr,size,flags,to,tolen)

    ssize_t
    lwip_sendto(int s, const void *data, size_t size, int flags,
                const struct sockaddr *to, socklen_t tolen)

send()
^^^^^^

send()函数可以用于UDP协议和TCP连接发送数据。在调用send()函数之前，必须使用connect()函数将远端主机的IP地址、
端口号与Socket连接结构进行绑定。对于UDP协议，send()函数将调用lwip_sendto()函数发送数据，而对于TCP协议，
将调用netconn_write_partly()函数发送数据。相对于sendto()函数，参数基本是没啥区别的，但无需我们设置远端主机的信息，
更加方便操作，因此这个函数在实际中使用也是很多的，具体见 代码清单16_12_。

代码清单 16‑12 send()

.. code-block:: c
   :name: 代码清单16_12

    #define send(s,dataptr,size,flags)          \
            lwip_send(s,dataptr,size,flags)
    ssize_t
    lwip_send(int s, const void *data, size_t size, int flags)

write()
^^^^^^^

这个函数一般用于处于稳定的TCP连接中传输数据，当然也能用于UDP协议中，它也是基于lwip_send上实现的，
但是无需我们设置flag参数，具体见 代码清单16_13_。

代码清单 16‑13 write()

.. code-block:: c
   :name: 代码清单16_13

    #define write(s,dataptr,len)         	\
                lwip_write(s,dataptr,len)

    ssize_t
    lwip_write(int s, const void *data, size_t size)
    {
        return lwip_send(s, data, size, 0);
    }

close()
^^^^^^^

close()函数是用于关闭一个指定的套接字，在关闭套接字后，将无法使用对应的套接字描述符索引到连接结构，
该函数的本质是对netconn_delete()函数的封装（真正处理的函数是netconn_prepare_delete()），
如果连接是TCP协议，将产生一个请求终止连接的报文发送到对端主机中，如果是UDP协议，将直接释放UDP控制块的内容，
具体见 代码清单16_14_。

代码清单 16‑14 close()

.. code-block:: c
   :name: 代码清单16_14

    #define close(s)            \
            lwip_close(s)

    int
    lwip_close(int s)

ioctl()、ioctlsocket()
^^^^^^^^^^^^^^^^^^^^^^

这两个函数很有意思（其实是一样的，本质是宏定义，都是调用lwip_ioctl()函数），它用于获取与设置套接字相关的操作参数，参数cmd指明对套接字的操作命令，在LwIP中只支持FIONREAD与FIONBIO命令：

1. FIONREAD命令确定套接字s自动读入的数据量，这些数据已经被接收，但应用线程并未读取的，
   所以可以使用这个函数来获取这些数据的长度，在这个命令状态下，argp参数指向一个无符号长整型，
   用于保存函数的返回值（即未读数据的长度）。如果套接字是SOCK_STREAM类型，则FIONREAD命令会返回recv()函数中所接收的所有数据量，这通常与在套接字接收缓存队列中排队的数据总量相同；而如果套接字是SOCK_DGRAM类型的，则FIONREAD命令将返回在套接字接收缓存队列中排队的第一个数据包大小。

2. FIONBIO命令用于允许或禁止套接字的非阻塞模式。在这个命令下，argp参数指向一个无符号长整型，
   如果该值为0则表示禁止非阻塞模式，而如果该值非0则表示允许非阻塞模式则。当创建一个套接字的时候，
   它就处于阻塞模式，也就是说非阻塞模式被禁止，这种情况下所有的发送、接收函数都会是阻塞的，直至发送、接收成功才得以继续运行；而如果是非阻塞模式下，所有的发送、接收函数都是不阻塞的，如果发送不出去或者接收不到数据，将直接返回错误代码给用户，这就需要用户对这些“意外”情况进行处理，保证代码的健壮性，这与BSD
   Socket是一致的。

其函数原型具体见 代码清单16_15_。

代码清单 16‑15 ioctl()、ioctlsocket()

.. code-block:: c
   :name: 代码清单16_15

    #define ioctl(s,cmd,argp)       \
                lwip_ioctl(s,cmd,argp)

    #define ioctlsocket(s,cmd,argp)      \
                lwip_ioctl(s,cmd,argp)

    int
    lwip_ioctl(int s, long cmd, void *argp)

setsockopt()
^^^^^^^^^^^^

看名字就知道，这个函数是用于设置套接字的一些选项的，参数level有多个常见的选项，如：

-  SOL_SOCKET：表示在Socket层。

-  IPPROTO_TCP：表示在TCP层。

-  IPPROTO_IP： 表示在IP层。

参数optname表示该层的具体选项名称，比如：

1. 对于SOL_SOCKET选项，可以是SO_REUSEADDR（允许重用本地地址和端口）、SO_SNDTIMEO（设置发送数据超时时间）、
   SO_SNDTIMEO（设置接收数据超时时间）、SO_RCVBUF（设置发送数据缓冲区大小）等等。

2. 对于IPPROTO_TCP选项，可以是TCP_NODELAY（不使用Nagle算法）、TCP_KEEPALIVE（设置TCP保活时间）等等。

3. 对于IPPROTO_IP选项，可以是IP_TTL（设置生存时间）、IP_TOS（设置服务类型）等等。

代码清单 16‑16 setsockopt()

.. code-block:: c
   :name: 代码清单16_16

    #define setsockopt(s,level,optname,opval,optlen)  \
    lwip_setsockopt(s,level,optname,opval,optlen)

    int
    lwip_setsockopt(int s,
                    int level,
                    int optname,
                    const void *optval,
                    socklen_t optlen)

getsockopt()
^^^^^^^^^^^^

这个函数与setsockopt()函数的选项参数及名称都是差不多的，只不过是作用是获得这些选项信息在这里就不过多讲解。

实验
~~~~

TCP Client
^^^^^^^^^^

这个实验现象与NETCONN
API中实验的是一样的，我们直接把上次的工程拷贝过来，然后将NETCONN
API替换成Socket
API就基本差不多了，我们首先在lwipopts.h文件中将宏LWIP_SOCKET配置为1，在文件中添加以下代码，注意，不要删除LWIP_NETCONN宏定义。

.. code-block:: c

    #define LWIP_SOCKET 1

在client.c文件中添加 代码清单16_17_ 所示代码，当然，端口号等信息根据你们自己的网络环境修改即可，
然后编译工程，下载到开发板上，电脑端的操作步骤与NETCONN
API中实验操作步骤是一样的，就不再过多赘述了。

代码清单 16‑17client.c文件内容

.. code-block:: c
   :name: 代码清单16_17

    #include "client.h"

    #include "lwip/opt.h"

    #include "lwip/sys.h"
    #include "lwip/api.h"

    #include <lwip/sockets.h>

    #define PORT              5001
    #define IP_ADDR        "192.168.0.181"

    static void client(void *thread_param)
    {
        int sock = -1;
        struct sockaddr_in client_addr;

        uint8_t send_buf[]= "This is a TCP Client test...\n";

        while (1)
        {
            sock = socket(AF_INET, SOCK_STREAM, 0);
            if (sock < 0)
            {
                printf("Socket error\n");
                vTaskDelay(10);
                continue;
            }

            client_addr.sin_family = AF_INET;
            client_addr.sin_port = htons(PORT);
            client_addr.sin_addr.s_addr = inet_addr(IP_ADDR);
            memset(&(client_addr.sin_zero), 0, sizeof(client_addr.sin_zero));

            if (connect(sock,
                        (struct sockaddr *)&client_addr,
                        sizeof(struct sockaddr)) == -1)
            {
                printf("Connect failed!\n");
                closesocket(sock);
                vTaskDelay(10);
                continue;
            }

            printf("Connect to iperf server successful!\n");

            while (1)
            {
                if (write(sock,send_buf,sizeof(send_buf)) < 0)
                    break;

                vTaskDelay(1000);
            }

            closesocket(sock);
        }

    }

    void
    client_init(void)
    {
        sys_thread_new("client", client, NULL, 512, 4);
    }

TCP Server
^^^^^^^^^^

同理，这个实验也只需把NETCONN
API中的实验拷贝过来，然后在lwipopts.h文件中将宏LWIP_SOCKET配置为1，再将tcpecho.c文件的内容替换为 代码清单16_18_
的内容即可，操作步骤也是一样的，然后编译工程并且下载到开发板上即可看到实验现象。

代码清单 16‑18tcpecho.c文件内容

.. code-block:: c
   :name: 代码清单16_18

    #include "tcpecho.h"

    #include "lwip/opt.h"

    #if LWIP_SOCKET
    #include <lwip/sockets.h>

    #include "lwip/sys.h"
    #include "lwip/api.h"
    /*--------------------------------------------------------------------*/

    #define PORT              5001
    #define RECV_DATA         (1024)


    static void
    tcpecho_thread(void *arg)
    {
        int sock = -1,connected;
        char *recv_data;
        struct sockaddr_in server_addr,client_addr;
        socklen_t sin_size;
        int recv_data_len;

        recv_data = (char *)pvPortMalloc(RECV_DATA);
        if (recv_data == NULL)
        {
            printf("No memory\n");
            goto __exit;
        }

        sock = socket(AF_INET, SOCK_STREAM, 0);
        if (sock < 0)
        {
            printf("Socket error\n");
            goto __exit;
        }

        server_addr.sin_family = AF_INET;
        server_addr.sin_addr.s_addr = INADDR_ANY;
        server_addr.sin_port = htons(PORT);
        memset(&(server_addr.sin_zero), 0, sizeof(server_addr.sin_zero));

        if (bind(sock, (struct sockaddr *)&server_addr, sizeof(struct sockaddr)) == -1)
        {
            printf("Unable to bind\n");
            goto __exit;
        }

        if (listen(sock, 5) == -1)
        {
            printf("Listen error\n");
            goto __exit;
        }

        while (1)
        {
            sin_size = sizeof(struct sockaddr_in);

            connected = accept(sock, (struct sockaddr *)&client_addr, &sin_size);

            printf("new client connected from (%s, %d)\n",
                inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port));
            {
                int flag = 1;

                setsockopt(connected,
                        IPPROTO_TCP,     /* set option at TCP level */
                        TCP_NODELAY,     /* name of option */
                        (void *) &flag, /* the cast is historical cruft */
                        sizeof(int));    /* length of option value */
            }

            while (1)
            {
                recv_data_len = recv(connected, recv_data, RECV_DATA, 0);

                if (recv_data_len <= 0)
                    break;

                printf("recv %d len data\n",recv_data_len);

                write(connected,recv_data,recv_data_len);

            }
            if (connected >= 0)
                closesocket(connected);

            connected = -1;
        }
    __exit:
        if (sock >= 0) closesocket(sock);
        if (recv_data) free(recv_data);
    }

    void
    tcpecho_init(void)
    {
        sys_thread_new("tcpecho_thread", tcpecho_thread, NULL, 512, 4);
    }

UDP
^^^

同理，这个实验也只需把NETCONN
API中的实验拷贝过来，然后在lwipopts.h文件中将宏LWIP_SOCKET配置为1，再将udpecho.c文件的内容替换为 代码清单16_19_
的内容即可，操作步骤也是一样的，然后编译工程并且下载到开发板上即可看到实验现象。

代码清单 16‑19udpecho.c文件内容

.. code-block:: c
   :name: 代码清单16_19

    #include "udpecho.h"

    #include "lwip/opt.h"

    #include <lwip/sockets.h>
    #include "lwip/api.h"
    #include "lwip/sys.h"

    #define PORT              5001
    #define RECV_DATA         (1024)

    /*-------------------------------------------------------------*/
    static void
    udpecho_thread(void *arg)
    {
        int sock = -1;
        char *recv_data;
        struct sockaddr_in udp_addr,seraddr;
        int recv_data_len;
        socklen_t addrlen;

        while (1)
        {
            recv_data = (char *)pvPortMalloc(RECV_DATA);
            if (recv_data == NULL)
            {
                printf("No memory\n");
                goto __exit;
            }

            sock = socket(AF_INET, SOCK_DGRAM, 0);
            if (sock < 0)
            {
                printf("Socket error\n");
                goto __exit;
            }

            udp_addr.sin_family = AF_INET;
            udp_addr.sin_addr.s_addr = INADDR_ANY;
            udp_addr.sin_port = htons(PORT);
            memset(&(udp_addr.sin_zero), 0, sizeof(udp_addr.sin_zero));

            if (bind(sock, (struct sockaddr *)&udp_addr, sizeof(struct sockaddr)) == -1)
            {
                printf("Unable to bind\n");
                goto __exit;
            }
            while (1)
            {
                recv_data_len=recvfrom(sock,recv_data,
                                    RECV_DATA,0,
                                    (struct sockaddr*)&seraddr,
                                    &addrlen);

                /*显示发送端的IP地址*/
                printf("receive from %s\n",inet_ntoa(seraddr.sin_addr));

                /*显示发送端发来的字串*/
                printf("recevce:%s",recv_data);

                /*将字串返回给发送端*/
                sendto(sock,recv_data,
                    recv_data_len,0,
                    (struct sockaddr*)&seraddr,
                    addrlen);
            }

    __exit:
            if (sock >= 0) closesocket(sock);
            if (recv_data) free(recv_data);
        }
    }
    /*---------------------------------------------------------------------*/
    void
    udpecho_init(void)
    {
        sys_thread_new("udpecho_thread", udpecho_thread, NULL, 2048, 4);
    }
