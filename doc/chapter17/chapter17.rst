使用RAW API接口编程
-------------------

RAW
API是基于回调函数实现的API接口，它是很底层的API接口，这需要开发者对LwIP有较深的了解才能很好使用它，RAW
API的核心就是对控制块的处理，因为对于报文数据的处理、注册回调函数等都是需要开发者自己去实现，都是比较麻烦的，但是有一个优点，那就是处理数据效率高。

RAW API的UDP编程
~~~~~~~~~~~~~~~~

学习本小节之前，必须要对UDP控制块熟悉，如果还不熟悉，可以参考第14.4
的内容。

新建控制块udp_new()
^^^^^^^^^^^^^^^^^^^

在使用UDP协议进行通信之前，必须创建一个UDP控制块，然后将控制块与对应的端口号进行绑定，才能发送报文，而在接收UDP报文的时候，这个端口号就是UDP报文唯一识别的标志，否则UDP报文将无法递交到应用层去处理，即无法通过UDP控制块的接收回调函数递交给应用层，新建控制块的函数很简单，就是在内存池中申请一个MEMP_UDP_PCB类型的内存块，用于存放UDP控制块的相关信息，并将其初始化为0，具体见
代码清单17_1_。

代码清单 17‑1 udp_new()源码

.. code-block:: c
   :name: 代码清单17_1

    struct udp_pcb *
    udp_new(void)
    {
        struct udp_pcb *pcb;

        LWIP_ASSERT_CORE_LOCKED();

        pcb = (struct udp_pcb *)memp_malloc(MEMP_UDP_PCB);
        /*  */
        if (pcb != NULL)
        {
            memset(pcb, 0, sizeof(struct udp_pcb));
            pcb->ttl = UDP_TTL;
        }
        return pcb;
    }

绑定控制块udp_bind()
^^^^^^^^^^^^^^^^^^^^

绑定控制块的作用其实就是将本机IP地址与端口号填写在UDP控制块中，以便表示唯一的应用，并且能正常与远端主机进行UDP通信，在这个函数中，它会将UDP控制块的local_ip与local_port字段进行初始化，并且把UDP控制块添加到udp_pcbs链表中，具体见
代码清单17_2_

代码清单 17‑2 udp_bind()源码

.. code-block:: c
   :name: 代码清单17_2

    err_t
    udp_bind(struct udp_pcb *pcb,
            const ip_addr_t *ipaddr,
            u16_t port)
    {
        struct udp_pcb *ipcb;
        u8_t rebind;

        LWIP_ASSERT_CORE_LOCKED();

        if (ipaddr == NULL)
        {
            ipaddr = IP4_ADDR_ANY;
        }

        rebind = 0;
        /* 检查UDP控制块是不是存在udp_pcbs链表中 */
        for (ipcb = udp_pcbs; ipcb != NULL; ipcb = ipcb->next)
        {
            if (pcb == ipcb)
            {
                rebind = 1;
                break;
            }
        }

        if (port == 0)
        {
            //如果端口号为0，则随机选择一个端口号
            port = udp_new_port();
            if (port == 0)
            {
                //没有可用端口号，返回错误
                return ERR_USE;
            }
        }
        else
        {
            for (ipcb = udp_pcbs; ipcb != NULL; ipcb = ipcb->next)
            {
                if (pcb != ipcb)
                {
                    /* 判断端口号是否被占用 */
                    if ((ipcb->local_port == port) &&
                            (ip_addr_cmp(&ipcb->local_ip, ipaddr) ||
                            ip_addr_isany(ipaddr) ||
                            ip_addr_isany(&ipcb->local_ip)))
                    {
                        return ERR_USE;
                    }
                }
            }
        }
        //设置IP地址与端口号
        ip_addr_set_ipaddr(&pcb->local_ip, ipaddr);

        pcb->local_port = port;
        mib2_udp_bind(pcb);

        if (rebind == 0)
        {
            /* 如果控制块没有在 */
            pcb->next = udp_pcbs;
            udp_pcbs = pcb;
        }

        return ERR_OK;
    }

建立会话udp_connect()
^^^^^^^^^^^^^^^^^^^^^

说明：本来是想写建立连接的，但是对于UDP协议来说，建立连接的这种说法并不太准确，因为UDP协议本身就是一个无连接协议，因此，我们就说建立UDP会话好了。

其实udp_connect()这个函数的作用就是设置控制块中的远端IP地址与端口号，然后将UDP控制块的状态设置为会话状态UDP_FLAGS_CONNECTED，
并且将UDP控制块插入udp_pcbs链表中，这样子就是建立会话，虽然是建立会话，但是不会如TCP协议一样，
发送请求连接、应答连接等信息到远端主机中，因为UDP是无连接的协议，只将控制块的远端IP地址与端口号设置，
表示发送数据的时候将发送到这个IP地址与端口号中，具体见 代码清单17_3_

代码清单 17‑3 udp_connect()源码

.. code-block:: c
   :name: 代码清单17_3

    err_t
    udp_connect(struct udp_pcb *pcb,
                const ip_addr_t *ipaddr,
                u16_t port)
    {
        struct udp_pcb *ipcb;

        LWIP_ASSERT_CORE_LOCKED();

        //如果没绑定本地IP地址与端口号，就进行绑定操作
        if (pcb->local_port == 0)
        {
            err_t err = udp_bind(pcb, &pcb->local_ip, pcb->local_port);
            if (err != ERR_OK)
            {
                return err;
            }
        }
        //设置remote_ip字段
        ip_addr_set_ipaddr(&pcb->remote_ip, ipaddr);

        pcb->remote_port = port;            //设置remote_port字段
        pcb->flags |= UDP_FLAGS_CONNECTED;

        /* 变量udp_pcbs链表，查找控制块是否存在链表中 */
        for (ipcb = udp_pcbs; ipcb != NULL; ipcb = ipcb->next)
        {
            if (pcb == ipcb)
            {
                /* 已经存在就无需重复插入，返回成功 */
                return ERR_OK;
            }
        }
        /* 插入udp_pcbs链表首部 */
        pcb->next = udp_pcbs;
        udp_pcbs = pcb;
        return ERR_OK;
    }

断开会话udp_disconnect()
^^^^^^^^^^^^^^^^^^^^^^^^

断开会话udp_disconnect()函数与建立会话udp_connect()函数相反，它主要是清除控制块的远端IP地址与端口号，
并且将UDP控制块的状态清除，当然，断开会话也不会发送任何的信息到对端主机中，具体见 代码清单17_4_。

提示：断开会话并不会删除UDP控制块，即不会释放UDP控制块的内存。

代码清单 17‑4 udp_disconnect()源码

.. code-block:: c
   :name: 代码清单17_4

    void
    udp_disconnect(struct udp_pcb *pcb)
    {
        LWIP_ASSERT_CORE_LOCKED();

        LWIP_ERROR("udp_disconnect: invalid pcb", pcb != NULL, return);

        /* 清除remote_ip */
        ip_addr_set_any(IP_IS_V6_VAL(pcb->remote_ip),
                        &pcb->remote_ip);

        pcb->remote_port = 0;
        pcb->netif_idx = NETIF_NO_INDEX;
        /* 清除UDP控制块状态 */
        udp_clear_flags(pcb, UDP_FLAGS_CONNECTED);
    }

接收数据udp_recv()
^^^^^^^^^^^^^^^^^^

这个函数的本质就是设置UDP控制块中的recv与recv_arg字段，这在UDP控制块就已经讲解的内容，recv是一个函数指针，指向一个udp_recv_fn类型的回调函数，它非常重要，是内核与应用程序交互的桥梁，当内核接收到数据的时候，就会调用这个回调函数，进而将数据递交到应用层处理，在recv回调函数中，pcb、p、addr、port等作为参数传递进去，方便用户的处理，其中pcb就是指向UDP控制块的指针，标识一个UDP会话，p是指向pbuf的指针，里面包含着接收到的数据，而addr与port记录着发送数据段的IP地址与端口号，具体见
代码清单17_5_。

代码清单 17‑5 udp_recv()源码

.. code-block:: c
   :name: 代码清单17_5

    typedef void (*udp_recv_fn)(void *arg,
                                struct udp_pcb *pcb,
                                struct pbuf *p,
                                const ip_addr_t *addr,
                                u16_t port);

    void
    udp_recv(struct udp_pcb *pcb, udp_recv_fn recv, void *recv_arg)
    {
        LWIP_ASSERT_CORE_LOCKED();

        LWIP_ERROR("udp_recv: invalid pcb", pcb != NULL, return);

        pcb->recv = recv;
        pcb->recv_arg = recv_arg;
    }

发送数据udp_send()与udp_sendto()
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

UDP发送数据也是依赖IP层，在用户使用发送数据的时候，应该为数据开辟一个pbuf用于存储数据，
并且pbuf中为UDP、IP、以太网首部预留足够的空间，然后用户调用udp_send()或者udp_sendto()函数将pbuf作为参数传递进去，
在发送数据的时候，UDP协议会将UDP首部相关的内容进行填充，形成一个完整的UDP报文递交到IP层，IP层也会为这个数据报添加IP首部，
形成完整的IP数据报递交到链路层中，然后添加以太网首部再发送出去，具体见 代码清单17_6_。

此外，还有一个注意的地方，其实这两个函数的作用是一样的，只不过udp_sendto()需要指定远端远端IP地址与端口号而已，最终都是调用udp_sendto_if()函数进行发送，在udp_sendto_if()函数函数中将调用udp_sendto_if_src()函数进行发送数据，而这个函数在讲解UDP协议的时候就已经讲解过了，此处就不再重复赘述。

代码清单 17‑6 udp_send()与udp_sendto()

.. code-block:: c
   :name: 代码清单17_6

    err_t
    udp_send(struct udp_pcb *pcb, struct pbuf *p)
    {
        if (IP_IS_ANY_TYPE_VAL(pcb->remote_ip))
        {
            return ERR_VAL;
        }

        return udp_sendto(pcb, p, &pcb->remote_ip, pcb->remote_port);
    }

    err_t
    udp_sendto(struct udp_pcb *pcb, struct pbuf *p,
                const ip_addr_t *dst_ip, u16_t dst_port)
    {
        struct netif *netif;

        if (!IP_ADDR_PCB_VERSION_MATCH(pcb, dst_ip))
        {
            return ERR_VAL;
        }

        LWIP_DEBUGF(UDP_DEBUG | LWIP_DBG_TRACE, ("udp_send\n"));

        if (pcb->netif_idx != NETIF_NO_INDEX)
        {
            netif = netif_get_by_index(pcb->netif_idx);
        }
        else
        {
            /* 找到发送报文的网卡 */
            netif = ip_route(&pcb->local_ip, dst_ip);
        }

        /* 没找到返回错误 */
        if (netif == NULL)
        {
            UDP_STATS_INC(udp.rterr);
            return ERR_RTE;
        }

        return udp_sendto_if(pcb, p, dst_ip, dst_port, netif);

    }

删除UDP控制块udp_remove()
^^^^^^^^^^^^^^^^^^^^^^^^^

这个函数的本质就是将UDP控制块从udp_pcbs链表中删除，并且释放UDP控制块的内存空间，这样子一个UDP控制块就彻底从系统中消失，
想要再次使用只能通过创建一个控制块并且将它插入链表中，具体见 代码清单17_7_。

代码清单 17‑7 udp_remove()源码

.. code-block:: c
   :name: 代码清单17_7

    void
    udp_remove(struct udp_pcb *pcb)
    {
        struct udp_pcb *pcb2;

        LWIP_ASSERT_CORE_LOCKED();

        LWIP_ERROR("udp_remove: invalid pcb", pcb != NULL, return);

        mib2_udp_unbind(pcb);
        /* 如果UDP控制块在链表的首部 */
        if (udp_pcbs == pcb)
        {
            /* 删除它 */
            udp_pcbs = udp_pcbs->next;

        }
        /* 如果UDP控制块不在链表的首部 */
        else
        {
            for (pcb2 = udp_pcbs; pcb2 != NULL; pcb2 = pcb2->next)
            {
                /* 变量链表找到要删除的UDP控制块 */
                if (pcb2->next != NULL && pcb2->next == pcb)
                {
                    /* 找到了就删除它 */
                    pcb2->next = pcb->next;
                    break;
                }
            }
        }
        //释放内存
        memp_free(MEMP_UDP_PCB, pcb);
    }

RAW API的TCP编程
~~~~~~~~~~~~~~~~

TCP协议使用RAW API进行编程，本质上也是对TCP控制块的操作。

新建控制块tcp_new()
^^^^^^^^^^^^^^^^^^^

这个函数用于分配一个TCP控制块，它通过tcp_alloc()函数分配一个TCP控制块结构来存储TCP控制块的数据信息，
如果没有足够的内容分配空间，那么tcp_alloc()函数就会尝试释放一些不太重要的TCP控制块，
比如就会释放处于TIME_WAIT、CLOSING等状态的TCP控制块，或者根据控制块的优先级进行释放，
释放一些不重要的TCP控制块，以完成新TCP控制块的分配，分配完成后，内核会初始化TCP控制块的各个字段内容，具体见 代码清单17_8_。

代码清单 17‑8tcp_new()源码

.. code-block:: c
   :name: 代码清单17_8

    struct tcp_pcb *
    tcp_new(void)
    {
        return tcp_alloc(TCP_PRIO_NORMAL);
    }

绑定控制块tcp_bind()
^^^^^^^^^^^^^^^^^^^^

对应TCP服务器端的程序，一般在创建一个TCP控制块的时候，就会调用tcp_bind()函数将本地的IP地址、端口号与一个控制块进行绑定，它的工作其实很简单，就遍历TCP控制块链表，在讲解13.7
TCP中的数据结构那一小节中，我们知道LwIP使用了4条TCP控制块链表来描述TCP控制块的各种状态，那么肯定是需要遍历所有的TCP控制块链表以便知道要绑定的IP地址与端口号是不重复的，即没有被其他TCP控制块使用，然后再将本地的IP地址、端口号与新创建的控制块进行绑定，最后再将绑定完毕的控制块插入tcp_bound_pcbs链表中，具体见 代码清单17_9_

代码清单 17‑9 tcp_bind()源码

.. code-block:: c
   :name: 代码清单17_9

    struct tcp_pcb *
    tcp_new(void)
    {
        return tcp_alloc(TCP_PRIO_NORMAL);
    }

    err_t
    tcp_bind(struct tcp_pcb *pcb,
            const ip_addr_t *ipaddr,
            u16_t port)
    {
        int i;
        int max_pcb_list = NUM_TCP_PCB_LISTS;
        struct tcp_pcb *cpcb;

        LWIP_ASSERT_CORE_LOCKED();

        if (ipaddr == NULL)
        {
            ipaddr = IP4_ADDR_ANY;
        }

        if (port == 0)
        {
            port = tcp_new_port();
            if (port == 0)
            {
                return ERR_BUF;
            }
        }
        else
        {
            /* 遍历TCP控制块链表 */
            for (i = 0; i < max_pcb_list; i++)
            {
                for (cpcb = *tcp_pcb_lists[i];
                        cpcb != NULL; cpcb = cpcb->next)
                {
                    if (cpcb->local_port == port)
                    {
                        {
                            /* 如果已经使用了IP地址与端口号就返回已使用错误 */
                            if ((IP_IS_V6(ipaddr) ==
                                    IP_IS_V6_VAL(cpcb->local_ip)) &&
                                    (ip_addr_isany(&cpcb->local_ip) ||
                                    ip_addr_isany(ipaddr) ||
                                    ip_addr_cmp(&cpcb->local_ip, ipaddr)))
                            {
                                return ERR_USE;
                            }
                        }
                    }
                }
            }
        }

        //设置IP地址
        if (!ip_addr_isany(ipaddr))
        {
            ip_addr_set(&pcb->local_ip, ipaddr);
        }
        //设置端口号
        pcb->local_port = port;

        //插入tcp_bound_pcbs链表
        TCP_REG(&tcp_bound_pcbs, pcb);
        return ERR_OK;
    }


控制块监听tcp_listen()
^^^^^^^^^^^^^^^^^^^^^^

作为TCP服务器端的程序，TCP监听状态是必须要实现的，它让服务器处于监听状态，等待TCP客户端的连接并且去处理它，它使用的函数就是tcp_listen()。我们也知道，一个TCP控制块对应着一条TCP连接，那么如果处于TCP监听状态的控制块太多，那肯定是需要消耗不少的内存资源，因此LwIP为了节省内存的开销，定义了不完整的TCP控制块——监听TCP控制块tcp_pcb_listen，它是专门应用于监听状态的控制块，里面包含完整TCP控制块的部分字段信息，因为处于监听状态的TCP控制块并不需要使用其他的字段内容，这样子的tcp_pcb_listen结构更小，更适合与嵌入式产品使用。

LwIP是这样子处理这两个TCP控制块的：首先申请一个监听TCP控制块tcp_pcb_listen，将完整的TCP控制块的部分内容拷贝到tcp_pcb_listen中，设置监听TCP控制块tcp_pcb_listen的状态为监听状态LISTEN，然后将完整的TCP控制块从绑定链表tcp_bound_pcbs中删除并且释放TCP控制块的内存空间，最后将监听TCP控制块插入监听链表tcp_listen_pcbs中，完成监听的操作，。

当服务器收到客户端发来的请求连接报文后，内核会遍历TCP监听链表tcp_listen_pcbs，找到和报文中一致的IP地址、目标端口号的控制块，然后内核将新建一个完整的TCP控制块，将监听TCP控制块tcp_pcb_listen的字段内容拷贝到完整的TCP控制块中，然后填写远端IP地址与端口号等字段，最后再将这个完整的TCP控制块挂载到tcp_active_pcbs链表中，当然，监听TCP控制块tcp_pcb_listen并不会被删除，因为它还需等待其他客户端的连接，这正是服务器必须要实现的功能。

提示：因为tcp_listen()函数的本质是一个宏定义，实际调用的函数是tcp_listen_with_backlog_and_err()，具体见
代码清单17_10_。

代码清单 17‑10 tcp_listen_with_backlog_and_err()源码

.. code-block:: c
   :name: 代码清单17_10

    struct tcp_pcb *
    tcp_listen_with_backlog_and_err(struct tcp_pcb *pcb,
                                    u8_t backlog,
                                    err_t *err)
    {
        struct tcp_pcb_listen *lpcb = NULL;
        err_t res;

        LWIP_UNUSED_ARG(backlog);

        LWIP_ASSERT_CORE_LOCKED();

        /* 如果已经处于监听状态 */
        if (pcb->state == LISTEN)
        {
            lpcb = (struct tcp_pcb_listen *)pcb;
            res = ERR_ALREADY;
            goto done;
        }
        //申请一个监听状态的TCP控制块tcp_pcb_listen，为了节省内存
        lpcb = (struct tcp_pcb_listen *)memp_malloc(MEMP_TCP_PCB_LISTEN);
        if (lpcb == NULL)
        {
            res = ERR_MEM;
            goto done;
        }
        //在监听TCP控制块中填写完整的TCP控制块的部分字段信息
        lpcb->callback_arg = pcb->callback_arg; //回调函数的传入参数
        lpcb->local_port = pcb->local_port;     //本地端口号
        lpcb->state = LISTEN;       //进入监听状态
        lpcb->prio = pcb->prio;     //优先级
        lpcb->so_options = pcb->so_options; //选项
        lpcb->netif_idx = NETIF_NO_INDEX;
        lpcb->ttl = pcb->ttl;     //生存时间
        lpcb->tos = pcb->tos;     //服务类型
        ip_addr_copy(lpcb->local_ip, pcb->local_ip);  //本地IP地址
        if (pcb->local_port != 0)
        {
            //将完整的TCP控制块从tcp_bound_pcbs链表中删除
            TCP_RMV(&tcp_bound_pcbs, pcb);
        }
        //释放完整的TCP控制块
        tcp_free(pcb);

        lpcb->accept = tcp_accept_null;
        //将监听TCP控制块插入监听链表tcp_listen_pcbs中
        TCP_REG(&tcp_listen_pcbs.pcbs, (struct tcp_pcb *)lpcb);
        res = ERR_OK;
    done:
        if (err != NULL)
        {
            *err = res;
        }
        return (struct tcp_pcb *)lpcb;
    }

处理连接tcp_accept()
^^^^^^^^^^^^^^^^^^^^

在服务器端，处理客户端连接的函数是tcp_accept()，当让服务器进入监听状态后，就需要立即调用这个函数，它向监听TCP控制块中的accept字段注册一个tcp_accept_fn类型的函数，当检测到客户端的连接时，内核就会调用这个回调函数，以完成连接操作，而在accept()函数中，需要用户去处理这些连接，回调函数有3个参数，其中newpcb是新TCP连接对应的控制块，用户需要对这个控制块进行后续处理，具体见
代码清单17_11_。

代码清单 17‑11 tcp_accept()源码

.. code-block:: c
   :name: 代码清单17_11

    typedef err_t (*tcp_accept_fn)(void *arg,
                                    struct tcp_pcb *newpcb,
                                    err_t err);

    void
    tcp_accept(struct tcp_pcb *pcb, tcp_accept_fn accept)
    {
        LWIP_ASSERT_CORE_LOCKED();
        if ((pcb != NULL) && (pcb->state == LISTEN))
        {
            struct tcp_pcb_listen *lpcb = (struct tcp_pcb_listen *)pcb;
            lpcb->accept = accept;
        }
    }

建立连接tcp_connect()
^^^^^^^^^^^^^^^^^^^^^

对于TCP客户端来说，主动建立连接是必不可少的一步，一般在客户端的实现步骤中，都是创建一个TCP控制块，然后绑定本地的IP地址与端口号，然后主动与服务器建立连接。那么建立连接的函数就是tcp_connect()函数，这个函数的最终目的是将TCP控制块从tcp_bound_pcbs绑定链表中取下并且放到tcp_active_pcbs链表中，并且发送一个连接请求报文，不过在处理这些事情之前，它会填写TCP控制块中发送窗口与接收窗口的相关字段，以达到最适的TCP连接，然后调用tcp_enqueue_flags()函数构造一个连接请求报文，将SYN标志置1，具体见
代码清单17_12_。

代码清单 17‑12tcp_connect()源码

.. code-block:: c
   :name: 代码清单17_12

    err_t
    tcp_connect(struct tcp_pcb *pcb,
                const ip_addr_t *ipaddr,
                u16_t port,
                tcp_connected_fn connected)
    {
        struct netif *netif = NULL;
        err_t ret;
        u32_t iss;
        u16_t old_local_port;

        LWIP_ASSERT_CORE_LOCKED();
        //设置远端IP地址（目标IP地址）和端口号
        ip_addr_set(&pcb->remote_ip, ipaddr);
        pcb->remote_port = port;

        if (pcb->netif_idx != NETIF_NO_INDEX)
        {
            netif = netif_get_by_index(pcb->netif_idx);
        }
        else
        {
            /* 找到要发送请求连接报文的网卡 */
            netif = ip_route(&pcb->local_ip, &pcb->remote_ip);
        }

        if (netif == NULL)
        {
            /* 找不到合适的发送网卡，返回失败 */
            return ERR_RTE;
        }

        /* 看看本地IP地址是否绑定了，没绑定就进行绑定 */
        if (ip_addr_isany(&pcb->local_ip))
        {
            const ip_addr_t *local_ip =
                ip_netif_get_local_ip(netif, ipaddr);
            if (local_ip == NULL)
            {
                return ERR_RTE;
            }
            ip_addr_copy(pcb->local_ip, *local_ip);
        }

        old_local_port = pcb->local_port;
        if (pcb->local_port == 0)
        {
            //如果没绑定本地端口号，就进行绑定操作
            pcb->local_port = tcp_new_port();

            if (pcb->local_port == 0)
            {
                return ERR_BUF;
            }
        }
        //设置发送窗口的相关参数
        iss = tcp_next_iss(pcb);
        pcb->rcv_nxt = 0;
        pcb->snd_nxt = iss;
        pcb->lastack = iss - 1;
        pcb->snd_wl2 = iss - 1;
        pcb->snd_lbb = iss - 1;

        /* 设置接收窗口的相关参数 */
        pcb->rcv_wnd = pcb->rcv_ann_wnd = TCPWND_MIN16(TCP_WND);
        pcb->rcv_ann_right_edge = pcb->rcv_nxt;
        pcb->snd_wnd = TCP_WND;
        pcb->mss = INITIAL_MSS;
        pcb->mss = tcp_eff_send_mss_netif(pcb->mss,
                                        netif,
                                        &pcb->remote_ip);
        pcb->cwnd = 1;
        pcb->connected = connected;

        /* 构造一个连接请求报文，将SYN标志置1 */
        ret = tcp_enqueue_flags(pcb, TCP_SYN);
        if (ret == ERR_OK)
        {
            /* 改变TCP控制块的状态为SYN_SENT */
            pcb->state = SYN_SENT;

            if (old_local_port != 0)
            {
                //将控制块从tcp_bound_pcbs链表移除
                TCP_RMV(&tcp_bound_pcbs, pcb);
            }

            //添加到tcp_active_pcbs链表中
            TCP_REG_ACTIVE(pcb);
            MIB2_STATS_INC(mib2.tcpactiveopens);

            //将控制块上的报文发送出去
            tcp_output(pcb);
        }
        return ret;
    }

在使用这个函数的时候，除了需要传递IP地址与端口号以外，还需要传入一个tcp_connected_fn类型的回调函数，
内核会自动注册在TCP控制块中，当建立连接之后，就会调用connected()这个回调函数，具体见 代码清单17_13_

代码清单 17‑13tcp_connected_fn类型的回调函数

.. code-block:: c
   :name: 代码清单17_13

    typedef err_t (*tcp_connected_fn)(void *arg,
                                    struct tcp_pcb *tpcb,
                                    err_t err);

终止连接tcp_close()
^^^^^^^^^^^^^^^^^^^

在任意时候，应用程序都可以主动调用tcp_close()函数终止一个TCP连接，终止连接的这个过程建议结合TCP状态转移图来学习，
内核会根据处于不同状态的TCP控制块有不一样的处理，当TCP控制块处于关闭状态CLOSED的时候，
会将TCP控制块从绑定链表tcp_bound_pcbs中移除，并且释放TCP控制块的内存空间；当TCP控制块处于监听状态的时候，
那么会将TCP控制块从监听链表tcp_listen_pcbs中移除，并且释放控制块的内存空间；当TCP控制块处于SYN_SENT状态时，
就将TCP控制块从tcp_active_pcbs链表中删除，并且释放控制块的内存空间；而对于处于其他状态的TCP控制块，直接通过tcp_close_shutdown_fin()函数来处理，主动关闭TCP连接，具体见 代码清单17_14_。

代码清单 17‑14 tcp_close()源码

.. code-block:: c
   :name: 代码清单17_14

    err_t
    tcp_close(struct tcp_pcb *pcb)
    {
        if (pcb->state != LISTEN)
        {
            tcp_set_flags(pcb, TF_RXCLOSED);
        }

        return tcp_close_shutdown(pcb, 1);
    }

    static err_t
    tcp_close_shutdown(struct tcp_pcb *pcb,
                    u8_t rst_on_unacked_data)
    {
        if (rst_on_unacked_data && ((pcb->state == ESTABLISHED)
                                    || (pcb->state == CLOSE_WAIT)))
        {
            if ((pcb->refused_data != NULL)
                || (pcb->rcv_wnd != TCP_WND_MAX(pcb)))
        {
                /* 发送TCP RESET数据包（设置了RST标志） */
                tcp_rst(pcb, pcb->snd_nxt,
                        pcb->rcv_nxt,
                        &pcb->local_ip,
                        &pcb->remote_ip,
                        pcb->local_port,
                        pcb->remote_port);

                tcp_pcb_purge(pcb);
                TCP_RMV_ACTIVE(pcb);
                /* 因为已经为它发送了一个RST，所以取消分配pcb */
                if (tcp_input_pcb == pcb)
                {
                    tcp_trigger_input_pcb_close();
                }
                else
                {
                    tcp_free(pcb);
                }
                return ERR_OK;
            }
        }

        /* 根据不同的状态进行不同的处理 */
        switch (pcb->state)
        {
        //关闭状态
        case CLOSED:
            if (pcb->local_port != 0)
            {
                TCP_RMV(&tcp_bound_pcbs, pcb);
            }
            tcp_free(pcb);
            break;
        //监听状态
        case LISTEN:
            tcp_listen_closed(pcb);
            tcp_pcb_remove(&tcp_listen_pcbs.pcbs, pcb);
            tcp_free_listen(pcb);
            break;
        //握手状态
        case SYN_SENT:
            TCP_PCB_REMOVE_ACTIVE(pcb);
            tcp_free(pcb);
            MIB2_STATS_INC(mib2.tcpattemptfails);
            break;
        //其他状态
        default:
            return tcp_close_shutdown_fin(pcb);
        }
        return ERR_OK;
    }

接收数据tcp_recv()
^^^^^^^^^^^^^^^^^^

这个函数的功能就是想控制块中的recv字段注册一个回调函数，当内核收到数据的时候就会调用这个回调函数，
进而让数据递交到应用层中。回调函数的传入参数有4个，其中主要的是tpcb，它是TCP控制块，表示了哪个TCP连接；
p是pbuf指针，指向接收到数据的pbuf，当内核检测到对方主动终止TCP连接的时候，也会触发回调函数，此时的pbuf为空，
而对于这种情况，用户就需要进行处理，也需要调用tcp_close()函数来终止本地到远端方向上的TCP连接。一般来说，
这个函数在connected()函数中调用，具体见 代码清单17_15_。

代码清单 17‑15 tcp_recv()源码

.. code-block:: c
   :name: 代码清单17_15

    typedef err_t (*tcp_recv_fn)(void *arg,
                                struct tcp_pcb *tpcb,
                                struct pbuf *p,
                                err_t err);

    void
    tcp_recv(struct tcp_pcb *pcb, tcp_recv_fn recv)
    {
        LWIP_ASSERT_CORE_LOCKED();
        if (pcb != NULL)
        {
            pcb->recv = recv;
        }
    }

发送数据tcp_sent()
^^^^^^^^^^^^^^^^^^

这个函数是用于注册一个发送的回调函数，即将一个tcp_sent_fn类型的函数注册到TCP控制块的sent字段中，
当发送的数据被对方确认接收后，内核会将发送窗口向后移动，并且调用这个注册的回调函数告诉应用，数据已经被对方接收了，
那么用户就可以根据这个函数来将那些已经发送的数据删除掉或者发送新的数据，具体见 代码清单17_16_.

代码清单 17‑16tcp_sent()源码

.. code-block:: c
   :name: 代码清单17_16

    typedef err_t (*tcp_sent_fn)(void *arg,
                                struct tcp_pcb *tpcb,
                                u16_t len);

    void
    tcp_sent(struct tcp_pcb *pcb, tcp_sent_fn sent)
    {
        LWIP_ASSERT_CORE_LOCKED();
        if (pcb != NULL)
        {
            pcb->sent = sent;
        }
    }

异常处理tcp_err()
^^^^^^^^^^^^^^^^^

这个函数是用于注册一个异常处理的函数，它向TCP控制块的err字段中注册一个tcp_err_fn类型的异常处理函数，
用户需要自行编写这个函数，可以拥有完成在连接异常的一些处理，比如连接失败的时候，我们可以释放TCP控制块的内存空间、
或者选择重连等等，具体见 代码清单17_17_。

代码清单 17‑17 tcp_err()源码

.. code-block:: c
   :name: 代码清单17_17

    typedef void  (*tcp_err_fn)(void *arg, err_t err);

    void
    tcp_err(struct tcp_pcb *pcb, tcp_err_fn err)
    {
        LWIP_ASSERT_CORE_LOCKED();
        if (pcb != NULL)
        {
            pcb->errf = err;
        }
    }

周期性回调tcp_poll()
^^^^^^^^^^^^^^^^^^^^

该函数用于在TCP控制块的poll字段注册一个类型为tcp_poll_fn的回调函数（函数的名字也是poll），内核会周期性调用控制块中的poll回调函数，调用的周期为interval*0.5s，因为0.5s是内核定时器的处理周期，用户可以适当使用poll回调函数完成一些周期性的事件，比如检测连接的情况、周期性发送一些数据等等，因为使用裸机编程（RAW
API），确实是不太好处理这些周期性的事件，所以LwIP为用户考虑了很多，该函数源码具体见
代码清单17_18_。

代码清单 17‑18 tcp_poll()源码

.. code-block:: c
   :name: 代码清单17_18

    typedef err_t (*tcp_poll_fn)(void *arg, struct tcp_pcb *tpcb);

    void
    tcp_poll(struct tcp_pcb *pcb, tcp_poll_fn poll, u8_t interval)
    {
        LWIP_ASSERT_CORE_LOCKED();

        pcb->poll = poll;

        pcb->pollinterval = interval;
    }

构建报文段tcp_write()
^^^^^^^^^^^^^^^^^^^^^

在一个稳定的TCP连接中，我们可以调用tcp_write()函数来构建一个TCP报文段，这个函数我们在TCP发送讲解的时候稍作介绍过，因为如果使用NETCONN
API与Socket
API的话，其实不用了解这个函数，即使这两种API最终都是调用这个函数来构建TCP报文段的，但是上层API全部给我们封装好了，但是在使用RAW
API的时候，我们需要调用这个函数直接构建报文段，这就需要我们对这个函数有一定了解，不过这个函数终究还是太复杂（代码多达400行），我就不讲解它里面是怎么实现的，只简单介绍一下整个函数的运作过程就好了，其函数原型具体见
代码清单17_19_。

代码清单 17‑19 tcp_write()函数原型

.. code-block:: c
   :name: 代码清单17_19

    err_t        tcp_write(struct tcp_pcb *pcb,
                        const void *dataptr,
                        u16_t len,
                        u8_t apiflags);

该函数有4个参数，其中pcb是相应的TCP连接，dataptr是数据指针，len是数据长度，以字节为单位，apiflags是表示应用程序期望内核对报文段的额外操作，如是否拷贝数据，是否设置首部中的PSH标志等等。

该函数首先会调用tcp_write_checks()函数检查一下是否允许构建报文段，看一下数据是否能挂载到发送缓冲区中；接下来内核会将可发送的数据组成TCP报文段并且挂载到TCP控制块的unsent字段中，注意了，tcp_write()函数只是构建TCP报文段并缓存在unsent字段中，真正发送TCP数据的函数是tcp_output()，这个函数在讲解TCP发送数据的时候就讲解过了，此处不再重复赘述。那么很多同学就有疑问了，为什么这个函数不是发送数据还要调用它呢？其实这个函数在构建好一个TCP报文段之后，内核会在超时处理中调用tcp_output()函数进行发送，而后者是根据TCP控制块unsent字段的内容进行发送数据的，因此，我们只需要把数据挂载到unsent字段中即可，内核会处理剩下的事情，当然啦，如果想要立即发送数据，也是可以在tcp_write()函数后调用tcp_output()函数，就可以立即发送数据。

更新接收窗口tcp_recved()
^^^^^^^^^^^^^^^^^^^^^^^^

其实在用户接收到数据之后，应该调用一下这个函数来更新接收窗口，因为内核不知道应用层是否真正接收到数据，
如果不调用这个函数，就没法进行确认，而发送的一方会认为对方没有接收到，因此会重发数据。在这个函数中，
它会调用tcp_update_rcv_ann_wnd()函数进行更新接收窗口，以告知发送方能发送多大的数据，参数pcb是对应的TCP连接控制块，
len表示应用程序已经处理完的数据长度，那么接收窗口也会增大len字节的长度，具体见 代码清单17_20_。

顺便提一下，如果发现应用程序中无法接收数据，但是能发送数据，那么很可能就是在接收到数据之后没调用tcp_recved()函数来更新接收窗口，导致接收窗口为0，无法接收数据。

代码清单 17‑20tcp_recved()源码

.. code-block:: c
   :name: 代码清单17_20

    void
    tcp_recved(struct tcp_pcb *pcb, u16_t len)
    {
        u32_t wnd_inflation;
        tcpwnd_size_t rcv_wnd;

        LWIP_ASSERT_CORE_LOCKED();

        rcv_wnd = (tcpwnd_size_t)(pcb->rcv_wnd + len);
        if ((rcv_wnd > TCP_WND_MAX(pcb)) || (rcv_wnd < pcb->rcv_wnd))
        {
            /* 窗口太大或tcpwnd_size_t溢出 */
            pcb->rcv_wnd = TCP_WND_MAX(pcb);
        }
        else
        {
            pcb->rcv_wnd = rcv_wnd;
        }

        wnd_inflation = tcp_update_rcv_ann_wnd(pcb);

        /* 如果窗口编号大于 TCP_WND/4 ，立即发送一个ack*/
        if (wnd_inflation >= TCP_WND_UPDATE_THRESHOLD)
        {
            tcp_ack_now(pcb);
            tcp_output(pcb);
        }

    }

实验
~~~~

TCP Client
^^^^^^^^^^

在这个实验中，我们使用RAW API实现一个TCP客户端，我们整个流程如下：

1. 创建一个TCP控制块。

2. 调用tcp_connect()函数与服务器建立连接。

3. 注册一个异常处理函数，在连接失败的时候进行合适的处理。

4. 建立连接之后就注册一个周期性发送数据的函数与接收处理函数，接收处理函数主要是用于在服务器主动断开的时候进行重连操作。

5. 周期性发送数据到服务器中。

    首先我们拿到一个移植好的LwIP裸机例程，然后添加两个文件，分别为tcpclient.c与tcpclient.h，
    然后添加以下代码，具体见 代码清单17_21_ 与 代码清单17_21_。

    实验的现象是与NETCONN API的实验现象是一致的，此处就不重复赘述。

代码清单 17‑21tcpclient.c文件内容

.. code-block:: c
   :name: 代码清单17_21

    #include "tcpclient.h"
    #include "lwip/netif.h"
    #include "lwip/ip.h"
    #include "lwip/tcp.h"
    #include "lwip/init.h"
    #include "netif/etharp.h"
    #include "lwip/udp.h"
    #include "lwip/pbuf.h"
    #include <stdio.h>
    #include <string.h>


    static struct tcp_pcb *client_pcb = NULL;

    static void client_err(void *arg, err_t err)
    {
        printf("connect error! closed by core!!\n");
        printf("try to connect to server again!!\n");

        //连接失败的时候释放TCP控制块的内存
        tcp_close(client_pcb);

        //重新连接
        TCP_Client_Init();
    }


    static err_t client_send(void *arg, struct tcp_pcb *tpcb)
    {
        uint8_t send_buf[]= "This is a TCP Client test...\n";

        //发送数据到服务器
        tcp_write(tpcb, send_buf, sizeof(send_buf), 1);

        return ERR_OK;
    }

    static err_t client_recv(void *arg,
                            struct tcp_pcb *tpcb,
                            struct pbuf *p,
                            err_t err)
    {
        if (p != NULL)
        {
            /* 更新窗口 */
            tcp_recved(tpcb, p->tot_len);

            /* 返回接收到的数据*/
            tcp_write(tpcb, p->payload, p->tot_len, 1);

            memset(p->payload, 0 , p->tot_len);
            pbuf_free(p);
        }
        else if (err == ERR_OK)
        {
            //服务器断开连接
            printf("server has been disconnected!\n");
            tcp_close(tpcb);

            //重新连接
            TCP_Client_Init();
        }
        return ERR_OK;
    }

    static err_t client_connected(void *arg,
                                struct tcp_pcb *pcb,
                                err_t err)
    {
        printf("connected ok!\n");

        //注册一个周期性回调函数
        tcp_poll(pcb,client_send,2);

        //注册一个接收函数
        tcp_recv(pcb,client_recv);

        return ERR_OK;
    }


    void TCP_Client_Init(void)
    {
        ip4_addr_t server_ip;
        /* 创建一个TCP控制块  */
        client_pcb = tcp_new();

        IP4_ADDR(&server_ip, 192,168,0,181);

        printf("client start connect!\n");

        //开始连接
        tcp_connect(client_pcb,
                    &server_ip,
                    TCP_CLIENT_PORT,
                    client_connected);

        //注册异常处理
        tcp_err(client_pcb, client_err);
    }

代码清单 17‑22tcpclient.h文件内容

.. code-block:: c
   :name: 代码清单17_22

    #ifndef _TCPCLIENT_H_
    #define _TCPCLIENT_H_

    #define TCP_CLIENT_PORT 5001

    void TCP_Client_Init(void);

    #endif

然后根据自己开发板所处的网络环境，修改main.h的网络相关的配置信息，注意，此工程是裸机工程，
因此我放置的网络配置信息所处的位置是不一样的，此处在main.h文件中，具体见 代码清单17_23_。

代码清单 17‑23main.h的网络信息

.. code-block:: c
   :name: 代码清单17_23

    /* USER CODE BEGIN 0 */
    #define DEST_IP_ADDR0               192
    #define DEST_IP_ADDR1               168
    #define DEST_IP_ADDR2                 0
    #define DEST_IP_ADDR3               102

    #define DEST_PORT                  6000

    #define UDP_SERVER_PORT            5002
    #define UDP_CLIENT_PORT            5002

    #define LOCAL_PORT                 5001

    /*Static IP ADDRESS: IP_ADDR0.IP_ADDR1.IP_ADDR2.IP_ADDR3 */
    #define IP_ADDR0                    192
    #define IP_ADDR1                    168
    #define IP_ADDR2                      0
    #define IP_ADDR3                    122

    /*NETMASK*/
    #define NETMASK_ADDR0               255
    #define NETMASK_ADDR1               255
    #define NETMASK_ADDR2               255
    #define NETMASK_ADDR3                 0

    /*Gateway Address*/
    #define GW_ADDR0                    192
    #define GW_ADDR1                    168
    #define GW_ADDR2                      0
    #define GW_ADDR3                      1
    /* USER CODE END 0 */

然后在main函数中调用一下TCP_Client_Init()函数，进行开发客户端的初始化即可，main.c文件内容具体见
代码清单17_24_。

代码清单 17‑24main.c文件内容

.. code-block:: c
   :name: 代码清单17_24

    #include "main.h"

    #include <lwip/opt.h>
    #include <lwip/arch.h>
    #include "tcpip.h"
    #include "lwip/init.h"
    #include "lwip/netif.h"
    #include "ethernetif.h"
    #include "netif/ethernet.h"
    #include "lwip/def.h"
    #include "lwip/stats.h"
    #include "lwip/etharp.h"
    #include "lwip/ip.h"
    #include "lwip/snmp.h"
    #include "lwip/timeouts.h"

    #include "tcpclient.h"

    struct netif gnetif;
    ip4_addr_t ipaddr;
    ip4_addr_t netmask;
    ip4_addr_t gw;
    uint8_t IP_ADDRESS[4];
    uint8_t NETMASK_ADDRESS[4];
    uint8_t GATEWAY_ADDRESS[4];

    void LwIP_Init(void)
    {
        /* IP addresses initialization */
        /* USER CODE BEGIN 0 */
    #ifdef USE_DHCP
        ip_addr_set_zero_ip4(&ipaddr);
        ip_addr_set_zero_ip4(&netmask);
        ip_addr_set_zero_ip4(&gw);
    #else
        IP4_ADDR(&ipaddr,IP_ADDR0,IP_ADDR1,IP_ADDR2,IP_ADDR3);
        IP4_ADDR(&netmask,NETMASK_ADDR0,NETMASK_ADDR1,
                NETMASK_ADDR2,NETMASK_ADDR3);
        IP4_ADDR(&gw,GW_ADDR0,GW_ADDR1,GW_ADDR2,GW_ADDR3);
    #endif /* USE_DHCP */
        /* USER CODE END 0 */

        /* Initilialize the LwIP stack without RTOS */
        lwip_init();

        /* add the network interface (IPv4/IPv6) without RTOS */
        netif_add(&gnetif, &ipaddr, &netmask,
                &gw, NULL, &ethernetif_init, &ethernet_input);

        /* Registers the default network interface */
        netif_set_default(&gnetif);

        if (netif_is_link_up(&gnetif))
        {
    /* When the netif is fully configured this function must be called */
            netif_set_up(&gnetif);
        }
        else
        {
            /* When the netif link is down this function must be called */
            netif_set_down(&gnetif);
        }

        /* USER CODE BEGIN 3 */

        /* USER CODE END 3 */
    }


    int flag = 0;
    int main(void)
    {
        //板级外设初始化
        BSP_Init();

        //LwIP协议栈初始化
        LwIP_Init();
        TCP_Client_Init();	//开发板客户端初始化
        while (1)
        {
            if (flag)
            {
                flag = 0;
                //调用网卡接收函数
                ethernetif_input(&gnetif);
            }
            //处理LwIP中定时事件
            sys_check_timeouts();
        }
    }

TCP Server
^^^^^^^^^^

在这个实验中，我们使用开发板做一个TCP回显服务器，等待着上位机软件的连接，建立连接之后，将客户端发送的数据回显到客户端中，其步骤如下：

1. 创建一个TCP控制块。

2. 绑定本地IP地址与端口号。

3. 进入监听状态。

4. 处理来自客户端的连接请求。

5. 连接完成后注册一个接收数据回调函数。

6. 在回调函数中更新接收窗口，并且将接收到的数据返回到客户端中。

在LwIP的裸机工程中创建两个文件，分别为tcpecho.c与tcpecho，并且添加以下代码，
具体见 代码清单17_25_ 与 代码清单17_26_，然后在main函数中调用TCP_Echo_Init()函数即可。

此实验的现象是与NETCONN API的实验现象是一致的，此处就不重复赘述。

代码清单 17‑25tcpecho.c文件内容

.. code-block:: c
   :name: 代码清单17_25

    #include "tcpecho.h"
    #include "lwip/netif.h"
    #include "lwip/ip.h"
    #include "lwip/tcp.h"
    #include "lwip/init.h"
    #include "netif/etharp.h"
    #include "lwip/udp.h"
    #include "lwip/pbuf.h"
    #include <stdio.h>
    #include <string.h>


    static err_t tcpecho_recv(void *arg,
                            struct tcp_pcb *tpcb,
                            struct pbuf *p,
                            err_t err)
    {
        if (p != NULL)
        {
            /* 更新窗口*/
            tcp_recved(tpcb, p->tot_len);

            /* 返回接收到的数据*/
            tcp_write(tpcb, p->payload, p->tot_len, 1);

            memset(p->payload, 0 , p->tot_len);
            pbuf_free(p);

        }
        else if (err == ERR_OK)
        {
            return tcp_close(tpcb);
        }
        return ERR_OK;
    }

    static err_t tcpecho_accept(void *arg,
                                struct tcp_pcb *newpcb,
                                err_t err)
    {
        tcp_recv(newpcb, tcpecho_recv);
        return ERR_OK;
    }

    void TCP_Echo_Init(void)
    {
        struct tcp_pcb *pcb = NULL;

        /* 创建一个TCP控制块  */
        pcb = tcp_new();

        /* 绑定TCP控制块 */
        tcp_bind(pcb, IP_ADDR_ANY, TCP_ECHO_PORT);


        /* 进入监听状态 */
        pcb = tcp_listen(pcb);

        /* 处理连接 */
        tcp_accept(pcb, tcpecho_accept);
    }

代码清单 17‑26tcpecho.h文件内容

.. code-block:: c
   :name: 代码清单17_26

    #ifndef _TCPECHO_H_
    #define _TCPECHO_H_

    #define TCP_ECHO_PORT 5001

    void TCP_Echo_Init(void);

    #endif

UDP
^^^

在这个实验中，我们用开发板与电脑上位机软件建立一个UDP会话，并且将上位机软件发送给开发板的数据返回去，做一个UDP回显的实验，其实验步骤如下：

1. 新建一个UDP控制块

2. 将控制块与本地IP地址、端口号进行绑定。

3. 注册一个接收数据的回调函数

4. 在接收回调函数中将接收到的数据发送到对端主机中。

5. 释放pbuf。

首先将LwIP的裸机工程拷贝一份，然后添加两个文件，分别为udpecho.c与udpecho.h，然后添加以下代码，
然后在main函数中调用UDP_Echo_Init()函数即可，具体见 代码清单17_27_ 与 代码清单17_28_。

这个实验现象是与NETCONN API的实验现象是一致的，此处就不重复赘述。

代码清单 17‑27udpecho.c文件内容

.. code-block:: c
   :name: 代码清单17_27

    #include "udpecho.h"
    #include "lwip/netif.h"
    #include "lwip/ip.h"
    #include "lwip/tcp.h"
    #include "lwip/init.h"
    #include "netif/etharp.h"
    #include "lwip/udp.h"
    #include "lwip/pbuf.h"
    #include <stdio.h>


    static void udp_demo_callback(void *arg,
                                struct udp_pcb *upcb,
                                struct pbuf *p,
                                const ip_addr_t *addr,
                                u16_t port)
    {
        struct pbuf *q = NULL;
        const char* reply = "This is reply!\n";

        if (arg)
        {
            printf("%s",(char *)arg);
        }

        pbuf_free(p);

        q = pbuf_alloc(PBUF_TRANSPORT, strlen(reply)+1, PBUF_RAM);
        if (!q)
        {
            printf("out of PBUF_RAM\n");
            return;
        }

        memset(q->payload, 0 , q->len);
        memcpy(q->payload, reply, strlen(reply));
        udp_sendto(upcb, q, addr, port);
        pbuf_free(q);
    }

    static char * st_buffer= "We get a data\n";
    void UDP_Echo_Init(void)
    {
        struct udp_pcb *udpecho_pcb;
        /* 新建一个控制块*/
        udpecho_pcb = udp_new();

        /* 绑定端口号 */
        udp_bind(udpecho_pcb, IP_ADDR_ANY, UDP_ECHO_PORT);

        /* 注册接收数据回调函数 */
        udp_recv(udpecho_pcb, udp_demo_callback, (void *)st_buffer);
    }

代码清单 17‑28 udpecho.h文件内容

.. code-block:: c
   :name: 代码清单17_28

    #ifndef _UDPECHO_H_
    #define _UDPECHO_H_

    #define UDP_ECHO_PORT 5001

    void UDP_Echo_Init(void);

    #endif
