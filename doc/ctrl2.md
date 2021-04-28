Refer: ctrl1

# ctrl工具的原理
Ctrl功能主要包括两部分：tool工具给dpvs程序通信及多核之间的通信接口。 本文主要对dpvs中的tools工具dpip, ipvsadmin做介绍。

## 一 dpip介绍
dpip的大部分源码都在tools/dpip下面。main()函数位于文件dpip.c中。这个工具跟linux下的命令ip命令的用法接近。

### 1.1 函数处理过程 
main()函数的第一步解析选项的参数，选项的参数保存在结构体struct dpip_conf中:
```
struct dpip_conf {
    int         af;
    int         verbose;
    int         stats;
    int         interval;
    int         count;
    bool        color;
    char        *obj;
    dpip_cmd_t  cmd;
    int         argc;
    char        **argv;
};
```
其中 obj是object的指针，object需要用到的参数存放在argv中。
接下来调用ojbect的方法来做下一步处理。

**注意**： 源文件中用了如下宏来注册object的方法：
```
#define __init __attribute__((constructor))
#define __exit __attribute__((destructor))
```
被__init宏标记的函数，会在main()调用前被调用。

处理步骤如下：
- help：调用object的help方法
- parse: 解析object的参数
- check：检查参数是否正确
- do_cmd：执行命令

### 1.2 函数处理举例:
以添加一个ip地址来举例说明处理流程。添加ip的命令如下：

./dpip addr add 192.168.7.7/32 dev dpdk1

命令说明：
- Command: add
- Object: addr
- Args: 192.168.7.7/32 dev dpdk1

这个命令会被文件addr.c中的方法处理。Addr没有parse和checke方法，所以直接调用函数**addr_do_cmd()**。该函数首先解析参数，将参数保存到结构体struct inet_addr_param。然后调用函数dpvs_setsockopt(sockoptid_t cmd,)。通过UNIX DOMAIN套接字"/var/run/dpvs_ctrl"，附带如下信息的参数，发送给dpvs。
```
struct inet_addr_param {
    int                 af;					// ipv4 or ipv6
    char                ifname[IFNAMSIZ];	// dpdk1
    union inet_addr     addr;				// 0x0707a8c0
    uint8_t             plen;				// 24
    union inet_addr     bcast;				// 0
    uint32_t            valid_lft;			// 0
    uint32_t            prefered_lft;		// 0
    uint8_t             scope;				// 0
    uint32_t            flags;				// 0

    uint32_t            sa_used;			// 0
    uint32_t            sa_free;			// 0
    uint32_t            sa_miss;			// 0
} __attribute__((__packed__));

struct dpvs_sock_msg *msg；
    msg->version = SOCKOPT_VERSION;
    msg->id = cmd;   // SOCKOPT_SET_IFADDR_ADD
    msg->type = SOCKOPT_SET;
    msg->len = in_len;
    res = sockopt_msg_send(clt_fd, msg, in, in_len);
```
id为SOCKOPT_SET_IFADDR_ADD和type为SOCKOPT_SET的消息会被函数ifa_sockopt_set()处理, 然后调用函数inet_addr_add()。
inet_addr_add()调用函数ifa_add_set()来做具体的处理，处理流程如下：
- 检查参数是否正确，地址是否已经存在
- 分配一个struct inet_ifaddr的结构体（表示inet层的一个ip地址），并初始化
- 如果是IPV6，则进行多播处理
- 添加route table， 添加本cpu的，然后把路由信息用msg发送到其他core（MSG_TYPE_ROUTE_ADD）.


## 二 ipvsadm介绍
dpip的大部分源码都在tools/ipvsadm下面。main()函数位于文件ipvsadm.c中。 这个工具跟linux下的命令ipvsadmin用法接近。还有一部分代码位于keepalived中，当前代码使用的是libipvs-2.6的库。
main函数中只是获取了dpvs的verstion信息，然后调用函数process_options()来处理大部分的工作。

### 2.1 功能描述
ipvsadmin命令又两个主要的部分，command和option。command主要有：
- virtual service： add, edit, delete
- local address: add, del, get
- blacklist address: add del get
- real server: add, edit, delete
- connection: set timeout, connection sync

除了以上的操作某个具体的表项的命令外，还有清除全部表项，显示表项等操作。每一个command支持的option都不同，很多option都是特定表项才支持的。

某个命令支持的选项在talbe commands_v_options中可以查到，每个选项对于特定命令可能是一下情况的一种：
- 不支持此选项
- 强制需要此选项
- 多个选项可能存在一个
- 可以多个同时存在的可选项

### 2.2 process_options()介绍
首先调用函数parse_options()来解析所有的command和option。然后检查该命令下的option是否合规。 再之后会对部分command的选项做一些调整，最后不同的命令交个不同的函数来处理。下面给出各个命令的command列表：
- CMD_ADD： **DPVS_SO_SET_ADD**, desc: --add-service, -A, add virtual service
- CMD_EDIT: **DPVS_SO_SET_EDIT**, desc: --edit-service, -E, edit virtual service
- CMD_DEL: **DPVS_SO_SET_DEL**, desc: --delete-service, -D, delete virtual service
- CMD_ADDLADDR: **SOCKOPT_SET_LADDR_ADD**, desc: --add-laddr, -P, add local address
- CMD_DELLADDR： **SOCKOPT_SET_LADDR_DEL**, desc: --del-laddr, -Q, del local address
- CMD_GETLADDR: **SOCKOPT_GET_LADDR_GETALL**, desc: --get-laddr, -G, get local address
- CMD_ADDBLKLST: **SOCKOPT_SET_BLKLST_ADD**, desc: --add-blklst, -U, add blacklist address
- CMD_DELBLKLST: **SOCKOPT_SET_BLKLST_DEL**, desc: --del-blklst, -V, del blacklist address
- CMD_GETBLKLST: **SOCKOPT_GET_BLKLST_GETALL**, desc: --get-blklst, -B, get blacklist address
- CMD_ADDDEST: **DPVS_SO_SET_ADDDEST**, desc: --add-server, -a, add real server
- CMD_EDITDEST: **DPVS_SO_SET_EDITDEST**, desc: --edit-server, -e, edit real server
- CMD_DELDEST: **DPVS_SO_SET_DELDEST**, desc: --delete-server, -d, delete real server
- CMD_FLUSH: desc: --clear, clear the whole table
- CMD_LIST: desc: --list, list the table

上表中第一列是ipvsadmin中队命令的编号的宏，第二列是调用dpvs_setsockopt()或者dpvs_getsockopt()函数的套接字选项，在dpvs中对应选项的处理函数查询ctrl1.md。其中的FLUSH和LIST需要多条命令结合使用，具体请参考源码。

### 2.2.1 add virtual service with scheduling mode
以如下命令为例：  ./ipvsadm -A -u 192.168.7.7:2456 -s rr

这个命令会被dpvs的函数dp_vs_set_svc()-->dp_vs_add_service()处理。

### 2.2.2 add real service with forwarding mode
以如下命令为例： ./ipvsadm -a -u 192.168.7.7:2456 -r 192.168.2.2 -b

这个命令会被dpvs的函数dp_vs_set_svc()-->dp_vs_add_dest()处理。

### 2.2.3 add local addr for FNAT forwarding mode
以如下命令为例： ./ipvsadm --add-laddr -z 192.168.2.11 -u 192.168.7.7:2456 -F dpdk0

在ipvsadm命令中，处理函数是ipvs_add_laddr，此函数分两步：
- 调用函数**ipvs_set_ipaddr(SOCKOPT_SET_IFADDR_ADD)** 添加dpvs的inet laddr。在dpvs程序中，这个命令会被函数ifa_sockopt_set()-->inet_addr_add()处理。
- 调用函数**dpvs_setsockopt(SOCKOPT_SET_LADDR_ADD)** 添加dpvs的real server相关的laddr,在dpvs程序中,这个命令会被dpvs的函数laddr_sockopt_set()-->dp_vs_laddr_add()处理。


