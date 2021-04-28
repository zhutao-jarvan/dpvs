# 一 概述
Ctrl功能主要包括两部分：tool工具给dpvs程序通信及多核之间的通信接口。 

## 1.1 多核之间的通信(per-core ring msg)
dpvs底层使用DPDK来实现，为了提高吞吐量，一般会有多个核处理job。多核编程中，为了提高效率，一般尽可能的然每个核的业务数据相互独立，但是有一些信息又必须同步，因此多核之间使用msg来通信。

## 1.2 工具和dpvs程序的通信（UNIX DOMAIN SOCKET）

为了方便控制和查看dpvs程序运行。提供了一些tool，tool需要跟dpvs程序通信，这些tool工具是通过UNIX套接字来通信的。例如dpip和ipvsadmin。 

# 二 初始化
在main中调用函数ctrl_init来完成主要的初始化工作。函数分别调用msg_init()和sockopt_init()，这两个函数都位于文件ctrl.c中。

```
int ctrl_init(void)
{
    int ret;

    ret = msg_init();
    if (unlikely(ret < 0)) {
        RTE_LOG(ERR, MSGMGR, "%s: msg module initialization failed!\n", __func__);
        return ret;
    }
    ret = sockopt_init();
    if (unlikely(ret < 0)) {
        RTE_LOG(ERR, MSGMGR, "%s: sockopt module initialization failed!\n", __func__);
        return ret;
    }
    return EDPVS_OK;
}
```


## 2.1 函数msg_init()
讲这个函数的功能前，先列出函数中用到的重要数据结构：
### 2.1.1 struct dpvs_msg:
dpvs_msg结构体用来表示一个单独的msg消息。其中的type与另一个结构体dpvs_msg_type关联。dpvs_msg是每一个消息的消息头，消息的内容在消息头的结尾。结构体如下：
```
/* inter-lcore msg structure */
struct dpvs_msg {
    struct list_head mq_node;
    msgid_t type;
    uint32_t seq;           /* msg sequence number */
    msg_mode_t mode;        /* msg mode */
    lcoreid_t cid;          /* which lcore the msg from, for multicast always Master */
    uint32_t flags;         /* msg flags */
    rte_atomic16_t refcnt;  /* reference count */
    rte_spinlock_t lock;    /* msg lock */
    struct dpvs_msg_reply reply;
    /* response data, created with rte_malloc... and filled by callback */
    uint32_t len;           /* msg data length */
    char data[0];           /* msg data */
};
```


### 2.1.2 struct dpvs_msg_type:
dpvs_msg_type结构体用来表示一个msg的type。系统的消息类型现在有18种，它们的前缀是宏MSG_TYPE_。消息有两种模式，单播和广播。广播的消息，会在其他所有的core上被接收，单播消息只会发送到特殊的core。结构体的unicast_msg_cb和multicast_msg_cb分别定义了收到该类型的msg的处理函数。

```
/* Unicast only needs UNICAST_MSG_CB, multicast need both UNICAST_MSG_CB and
 * MULTICAST_MSG_CB, and MULTICAST_MSG_CB is set to a default function which does
 * nothing if not set. For mulitcast msg, UNICAST_MSG_CB return a dpvs_msg to
 * Master with the SAME seq number as the msg recieved. */
struct dpvs_msg_type {
    msgid_t type;
    uint8_t prio;
    lcoreid_t cid;          /* on which lcore the callback func registers */
    msg_mode_t mode;        /* distinguish unicast from multicast for the same msg type */
    UNICAST_MSG_CB unicast_msg_cb;     /* call this func if msg is unicast, i.e. 1:1 msg */
    MULTICAST_MSG_CB multicast_msg_cb; /* call this func if msg is multicast, i.e. 1:N msg */
    rte_atomic32_t refcnt;
    struct list_head list;
};

#define MSG_TYPE_REG                        1
#define MSG_TYPE_UNREG                      2
#define MSG_TYPE_HELLO                      3
#define MSG_TYPE_GET_ALL_SLAVE_ID           4
#define MSG_TYPE_MASTER_XMIT                5
#define MSG_TYPE_ROUTE_ADD                  6
#define MSG_TYPE_ROUTE_DEL                  7
#define MSG_TYPE_NETIF_LCORE_STATS          8
#define MSG_TYPE_BLKLST_ADD                 9
#define MSG_TYPE_BLKLST_DEL                 10
#define MSG_TYPE_STATS_GET                  11
#define MSG_TYPE_SAPOOL_STATS               12
#define MSG_TYPE_TC_STATS                   13
#define MSG_TYPE_CONN_GET                   14
#define MSG_TYPE_CONN_GET_ALL               15
#define MSG_TYPE_IPV6_STATS                 16
#define MSG_TYPE_ROUTE6                     17
#define MSG_TYPE_NEIGH_GET                  18
```


### 2.1.3 函数msg_init()的初始化任务
再讲几个重要全局变量：
- **msg_pool**: type(struct dpvs_mempool), msg的内存池，所有核共享。
- **msg_ring**: type(struct rte_ring), msg的队列，每个核都有一个。
- **ctrl_lcore_job**：type(struct netif_lcore_loop_job)，lcore loop job，会注册到每个core上去

函数的初始化过程也是围绕着上诉的几个重要全局变量展开的。首先创建一个msg_pool，当后续有msg发送或者应答msg的时候，首先在msg_pool中分配一个msg的内存，使用完再返回msg_pool。然后在给每个lcore分配msg队列，这个队列使用的是dpdk的ring机制。再之后在所有的**slave lcore**上注册一个快速loop job（**也许应该换成慢速的job**）。job的处理函数是**msg_slave_process**，这个函数会以此队列中取出消息并处理（注意：此函数只处理单播消息）。 

最后调用函数register_built_in_msg()来将register和unregister的msg消息类型注册到所有的slave lcore的msg类型中。此后就可以处理其他模块的register和unregister的请求了。最后给每个slave lcore注册一个MSG_TYPE_MASTER_XMIT类型的msg处理类型，此类型的消息是用于master core通过slave core发送数据包的。自此，初始化任务完成。

## 2.2 sockopt_init()
此函数的处理逻辑相对简单，只是创建了一个UNIX套接字（"/var/run/dpvs_ctrl"），并开始监听他。




# 三 消息处理

初始化完成后，在main函数中的无限循环中分别调用函数sockopt_ctl()处理tools的消息和msg_master_process()来处理master core收到的消息。

## 3.1 函数sockopt_ctl()
sockopt_ctl是用来接收tool工具发来的对dpvs网络进行配置的set或者get请求的，每个请求都是通过一个msg来封装的，msg的header如下：
```
struct dpvs_sock_msg {
    uint32_t version;
    sockoptid_t id;
    enum sockopt_type type;
    size_t len;
    char data[0];
};
```
收到消息后，查询sockopt的type[GET or SET], ,然后根据id找到对于的处理函数，处理完成后，返回给客户端。
sockopt的id在系统中有多处定义，分别如下(括号中的值为value：count)：
- inet_addr_init: SOCKOPT_SET_IFADDR_ADD(400:3), set: ifa_sockopt_set
- ipset_init：SOCKOPT_SET_IPSET_ADD(3300:2), set: ipset_sockopt_set
- ipv6_ctrl_init：SOCKOPT_IP6_SET(1100:1), set: ip6_sockopt_set
- ip_tunnel_init: SOCKOPT_TUNNEL_ADD(1000:4), set: tunnel_so_set
- dp_vs_blklst_init: SOCKOPT_SET_BLKLST_ADD(700), set: blklst_sockopt_set
- conn_ctrl_init: 0,0, set: NULL
- dp_vs_laddr_init: SOCKOPT_SET_LADDR_ADD(100:3), set: laddr_sockopt_set 
- dp_vs_service_init: SOCKOPT_SVC_BASE(200: 9), set: dp_vs_set_svc
- arp_init: SOCKOPT_SET_NEIGH_ADD(601: 2), set: neigh_sockopt_set
- netif_ctrl_init: SOCKOPT_NETIF_SET_LCORE(500: 3), set: netif_sockopt_set
- route_init: SOCKOPT_SET_ROUTE_ADD(300:4), set: route_sockopt_set
- route6_init: SOCKOPT_SET_ROUTE6_ADD_DEL(6300:2), set: rt6_sockopt_set
- tc_ctrl_init: SOCKOPT_TC_ADD(900:4), set: tc_sockopt_set
- vlan_init: SOCKOPT_SET_VLAN_ADD(800:2), set: vlan_sockopt_set



## 3.2 函数msg_master_process()
从master core的ring中读取消息并处理。


