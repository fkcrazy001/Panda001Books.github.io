# tcpdump 研究

想研究下tcpdump，主要是以下几点：
1. bpf/ebpf 程序加载过程
2. 用户态程序与内核之间的数据流分析
3. 给我的基于DPDK的用户态抓包程序进行修改

## gemini 给的的意见
我先让ai给我生成一下调研的大概方向，这是它给出的。令我意外的是，
dpdk竟然还支持了用户态的ebpf-runtime，这里可以作为额外增加的一点调研方向。
```markdown
当然，这是一个非常专业和有深度的研究目标。tcpdump 的架构是网络分析的基石，而将其思想与 DPDK 结合，则是通向高性能用户态网络处理的关键。

下面我为您设计一份分步、可行的研究方案。

宏观架构概览：tcpdump 是如何工作的？
在深入细节之前，我们先建立一个心智模型。tcpdump 的工作流程可以简化为以下几个核心组件的协作：

用户接口 (CLI): 你输入的命令，如 tcpdump -i eth0 'tcp port 80'。
libpcap 库: 这是 tcpdump 的心脏和大脑。它负责：
解析过滤表达式: 将 'tcp port 80' 这种人类可读的字符串，编译成 BPF (Berkeley Packet Filter) 虚拟机能够理解的字节码。
设置抓包会话: 创建一个特殊的套接字 (AF_PACKET socket)，并将其绑定到指定的网络接口 (eth0)。
加载过滤器: 将编译好的 BPF 字节码通过 setsockopt 系统调用附加到这个套接字上。
读取数据: 从内核为这个套接字准备的环形缓冲区 (Ring Buffer) 中高效地读取数据包。
内核:
AF_PACKET Socket: 提供一个原始接口，允许用户态程序直接访问链路层的数据包。
BPF 虚拟机 (JIT): 在内核中运行 libpcap 提供的 BPF 程序。当网络接口收到一个包后，数据会经过这个虚拟机。
过滤: 如果数据包不匹配 BPF 过滤器，内核会直接丢弃它，根本不会将其复制到用户态，这是性能优化的关键。
数据拷贝: 如果数据包匹配，内核才将其拷贝到 AF_PACKET socket 的共享内存环形缓冲区中。
tcpdump 主程序:
从 libpcap 提供的接口循环读取匹配的数据包。
解析包头（Ethernet, IP, TCP/UDP 等）。
按照用户指定的格式，将解析后的信息打印到标准输出。
可行的研究方案
我们将按照您的三个目标，分阶段进行研究，从理论到实践。

阶段一：深入理解 BPF 加载与内核交互 (目标 1 & 2)
这个阶段的目标是亲手验证上述模型的每一个环节。

1. 理论学习与工具准备

阅读文档:
man pcap-filter: 学习 tcpdump 过滤表达式的语法。这是 BPF 语言的前端。
man tcpdump: 重点关注 -d, -dd, -ddd 选项，它们可以让你看到 libpcap 编译出的 BPF 汇编指令。
工具:
tcpdump: 我们的主要研究对象。
strace: 系统调用追踪神器，用它来观察 tcpdump 和内核之间到底发生了什么。
2. 动手实践：解剖 BPF 加载过程

编译过滤器:

运行命令: tcpdump -i any -d 'tcp and port 80'
观察输出: 你会看到类似 BPF 汇编的指令，例如 ldh [12], jeq #0x800, ...。这证明了 libpcap 确实将字符串编译成了 BPF 程序。
你的任务: 理解这些指令的含义。ldh (load half-word) 是在加载以太网头中的协议类型字段，jeq (jump if equal) 是在判断协议是否为 IP (0x0800)。
追踪系统调用:

运行命令: sudo strace -e trace=socket,setsockopt,recvfrom,mmap tcpdump -i eth0 -c 1 'host 1.1.1.1'
观察输出，并寻找关键调用:
socket(AF_PACKET, SOCK_RAW, ...): 这就是在创建抓包用的原始套接字。
setsockopt(..., SOL_SOCKET, SO_ATTACH_FILTER, ...): 这就是 BPF 程序的加载点！ strace 会显示一个结构体，其中包含了 BPF 指令的数量和指向指令数组的指针。
mmap(...): 这是 libpcap 现代版本中的关键优化。它将内核的环形缓冲区直接内存映射到 tcpdump 的进程空间，避免了每次 read() 都需要从内核空间向用户空间拷贝数据的开销。
recvfrom(...) 或 poll(...): tcpdump 等待内核通知有新数据包到达，并从映射的内存中直接读取。
3. 总结数据流

入向 (Ingress): NIC -> 内核协议栈 -> BPF 过滤器 (在 AF_PACKET socket 上运行) -> (如果匹配) -> 拷贝到 AF_PACKET 的环形缓冲区 -> tcpdump 通过 mmap 的内存区域直接读取。
核心: 过滤发生在内核态，且尽早发生，极大地减少了无用数据拷贝和上下文切换的开销。
阶段二：对比 tcpdump 模型与 DPDK 模型
在修改你的 DPDK 程序之前，必须深刻理解两者在哲学上的根本不同。

特性	tcpdump / libpcap 模型	DPDK (Data Plane Development Kit) 模型
控制平面	内核 (Kernel)	用户态 (Userspace)
驱动	内核标准驱动	用户态驱动 (PMD - Poll Mode Driver)
数据路径	内核旁路，但仍需内核调度和数据拷贝（或 mmap）	内核完全旁路 (Kernel Bypass)
包处理方式	中断驱动 + poll/select (阻塞等待)	轮询 (Polling)，死循环检查网卡队列是否有包
CPU 使用	较低，只在有包到达时被唤醒	极高，通常会独占一个或多个 CPU 核心 100%
数据拷贝	至少一次从 NIC DMA 到内核内存，再到用户态（mmap 优化）	零拷贝 (Zero-Copy)，网卡直接 DMA 到用户态分配的内存
易用性	非常高，一个 apt install 即可用	复杂，需要绑定网卡、配置大页内存、处理 NUMA
性能	数 Gbps，受限于上下文切换和内核处理	数十乃至上百 Gbps，只受限于内存带宽和 CPU 速度
结论: tcpdump 是请求内核帮忙过滤，而 DPDK 是**“踢开”内核，自己直接操作硬件**。

阶段三：为你的 DPDK 程序设计和实现过滤功能 (目标 3)
现在，你的目标是为你的 DPDK 程序增加类似 tcpdump 的强大过滤能力。DPDK 本身不提供类似 'tcp port 80' 这样的高级过滤语言。你有以下几种方案可以实现，从易到难：

方案 A: 硬编码或简单解析 (入门级)

在你的 DPDK 抓包循环中，直接用 C/Rust 代码检查包头。

c复制代码
// DPDK main loop
struct rte_mbuf *pkts[BURST_SIZE];
const uint16_t nb_rx = rte_eth_rx_burst(port_id, 0, pkts, BURST_SIZE);

for (int i = 0; i < nb_rx; i++) {
    struct rte_ether_hdr *eth_hdr = rte_pktmbuf_mtod(pkts[i], struct rte_ether_hdr *);
    if (eth_hdr->ether_type == rte_cpu_to_be_16(RTE_ETHER_TYPE_IPV4)) {
        // Check IP header for protocol, source/dest IP...
        // Check TCP header for source/dest port...
        // 如果匹配，则处理；否则，rte_pktmbuf_free(pkts[i])
    } else {
        rte_pktmbuf_free(pkts[i]);
    }
}
优点: 简单直接，无外部依赖。
缺点: 不灵活，每次修改过滤规则都需要重新编译。
方案 B: 利用硬件过滤卸载 (性能最佳)

许多现代网卡支持硬件过滤 (Flow Director, 5-tuple filter 等)。你可以通过 DPDK 的 rte_flow API 将过滤规则编程到网卡硬件中。

优点: 性能最高，不匹配的包根本不会被 DMA 到主机内存，不消耗 CPU。
缺点: 并非所有网卡都支持，rte_flow API 相对复杂，支持的规则有限。
方案 C: 重用 libpcap 的编译器 + 用户态 BPF 解释器 (最灵活和强大)

这是最接近 tcpdump 思想的方案，强烈推荐。

链接 libpcap: 在你的 DPDK 项目中，包含 pcap.h 并链接 -lpcap。你只使用它的编译功能，不使用它的抓包功能。

获取 BPF 字节码:

c复制代码
#include <pcap/pcap.h>

// 假设 filter_string 来自用户输入，如 "tcp port 80"
const char *filter_string = "tcp port 80"; 
struct bpf_program fp;

// 使用 pcap_compile_nopcap 获取 BPF 程序，它不需要一个活动的 pcap_t 句柄
// DLT_EN10MB 表示链路层是 Ethernet
if (pcap_compile_nopcap(SNAPLEN, DLT_EN10MB, &fp, filter_string, 1, PCAP_NETMASK_UNKNOWN) < 0) {
    // 错误处理
}
// 现在 fp.bf_insns 里就是编译好的 BPF 字节码
在用户态运行 BPF: DPDK 自带一个 BPF 库，可以加载并执行 BPF 程序。

c复制代码
#include <rte_bpf.h>

// 创建 BPF 执行上下文
struct rte_bpf *bpf_ctx = rte_bpf_load(&fp);
pcap_freecode(&fp); // 释放 libpcap 分配的资源

// ... 在你的 DPDK 抓包循环中 ...
for (int i = 0; i < nb_rx; i++) {
    // 从 mbuf 中获取数据包的指针和长度
    void *pkt_data = rte_pktmbuf_mtod(pkts[i], void *);
    uint32_t pkt_len = rte_pktmbuf_data_len(pkts[i]);

    // 在用户态执行 BPF 过滤
    if (rte_bpf_exec(bpf_ctx, pkt_data, pkt_len) > 0) {
        // 包匹配！进行处理...
    } else {
        // 包不匹配，释放
        rte_pktmbuf_free(pkts[i]);
    }
}
rte_bpf_destroy(bpf_ctx);
总结与建议

先完成阶段一，用 strace 和 tcpdump -d 把 libpcap 和内核的交互流程摸得滚瓜烂熟。
深刻理解阶段二的对比表格，明确你的 DPDK 程序在生态位上的不同。
直接上手实现阶段三的方案 C。这是最能体现你学习成果的方案，它完美地将 libpcap 的易用前端与 DPDK 的高性能后端结合在了一起，让你的程序既强大又灵活。
祝你研究顺利！这是一个非常有价值的技术深潜。
```

嗯，还比较合理。所以先从 strace 开始吧。
看看在输入 `tcpdump -i bond0 'port 8888'` 这个命令之后，整个程序都干啥了。

## strace tcpdump -i bond0 'port 8888'

这个日志会很多，节选了一些我认为关键的部分。然后加了一些注释，即我认为它在干什么。
```c
//...
// 早期启动阶段，说明tcpdump依赖 libpcap 库
openat(AT_FDCWD, "/usr/lib64/libpcap.so.1", O_RDONLY|O_CLOEXEC) = 3
...
// 一个 raw socket，用于后续操作
socket(AF_PACKET, SOCK_RAW, htons(0 /* ETH_P_??? */)) = 4
ioctl(4, SIOCGIFINDEX, {ifr_name="lo", ifr_ifindex=1}) = 0
ioctl(4, SIOCGIFHWADDR, {ifr_name="bond0", ifr_hwaddr={sa_family=ARPHRD_ETHER, sa_data=50:79:73:0d:9b:ed}}) = 0
// 开启混杂模式
setsockopt(4, SOL_PACKET, PACKET_ADD_MEMBERSHIP, {mr_ifindex=if_nametoindex("bond0"), mr_type=PACKET_MR_PROMISC, mr_alen=0, mr_address=50:79:73:0d:9b:ed}, 16) = 0
// 准备ringbuf
setsockopt(4, SOL_PACKET, PACKET_AUXDATA, [1], 4) = 0
getsockopt(4, SOL_SOCKET, SO_BPF_EXTENSIONS, [64], [4]) = 0
mmap(NULL, 266240, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7faf62328000
getsockopt(4, SOL_PACKET, PACKET_HDRLEN, [48], [4]) = 0
setsockopt(4, SOL_PACKET, PACKET_VERSION, [2], 4) = 0
setsockopt(4, SOL_PACKET, PACKET_RESERVE, [4], 4) = 0
setsockopt(4, SOL_PACKET, PACKET_RX_RING, 0x7fff1856bb30, 28) = 0
mmap(NULL, 2097152, PROT_READ|PROT_WRITE, MAP_SHARED, 4, 0) = 0x7faf61a00000

// 开始attach filter
bind(4, {sa_family=AF_PACKET, sll_protocol=htons(ETH_P_ALL), sll_ifindex=if_nametoindex("bond0"), sll_hatype=ARPHRD_NETROM, sll_pkttype=PACKET_HOST, sll_halen=0}, 20) = 0
setsockopt(4, SOL_SOCKET, SO_ATTACH_FILTER, {len=24, filter=0x55ea646ae0d0}, 16) = 0

```


## datapath优化
可以看到，tcpdump抓包是用了 [packet mmap](https://www.kernel.org/doc/html/latest/networking/packet_mmap.html) 的方法。
主要是这么几步:
```c
[setup]     socket() -------> creation of the capture socket
            setsockopt() ---> allocation of the circular buffer (ring)
                              option: PACKET_RX_RING
            mmap() ---------> mapping of the allocated buffer to the
                              user process

[capture]   poll() ---------> to wait for incoming packets

[shutdown]  close() --------> destruction of the capture socket and
                              deallocation of all associated
                              resources.
```

它比直接读写fd，就是少了一次从内核态到用户态的数据拷贝。感觉这里带来的性能提升应该还是很可观的。
不知道 rust 的标准库是不是采用的这个方法？我觉得应该不是，不然的话标准库可能得实现一套完整的协议栈了。
这里留个坑，我觉得可以做下这两种情况下的性能对比。
- [性能对比](./packet_mmap_perf.md)

## libpcap 生成过滤条件
由于 libpcap 是用户态的，strace 现在还不能看具体是干了什么。不过我知道好像 `ftrace` 可以? 
anyway, 先尝试看看源码和文档，看看 libpcap 对于这种 'port 8888' 具体会生成什么东西。

### bpf 分析
```shell
[root@localhost ~]# tcpdump -i bond0 -d "port 8888"
(000) ldh      [12]
(001) jeq      #0x86dd          jt 2	jf 10
(002) ldb      [20]
(003) jeq      #0x84            jt 6	jf 4
(004) jeq      #0x6             jt 6	jf 5
(005) jeq      #0x11            jt 6	jf 23
(006) ldh      [54]
(007) jeq      #0x22b8          jt 22	jf 8
(008) ldh      [56]
(009) jeq      #0x22b8          jt 22	jf 23
(010) jeq      #0x800           jt 11	jf 23
(011) ldb      [23]
(012) jeq      #0x84            jt 15	jf 13
(013) jeq      #0x6             jt 15	jf 14
(014) jeq      #0x11            jt 15	jf 23
(015) ldh      [20]
(016) jset     #0x1fff          jt 23	jf 17
(017) ldxb     4*([14]&0xf)
(018) ldh      [x + 14]
(019) jeq      #0x22b8          jt 22	jf 20
(020) ldh      [x + 16]
(021) jeq      #0x22b8          jt 22	jf 23
(022) ret      #262144
(023) ret      #0
```
指令本身还是很好理解的，但是类似于 [12]，这种指向的是哪儿呢？可能还需要去看看BPF调用规范。

- BPF engine and instruction set
    
btw，我之前做过nes的simulator，所以对这三个参数倍感亲切，

  - 首先描述寄存器

    bpf不像epbf，寄存器很简单，只有一个32位的 A(累加) 寄存器，一个32位的 X 寄存器，16个32位的M[i]寄存器

   - 指令 (linux/filter.h以及 linux/bpf_common.h )
    
        就不搬了，常见的 load/store/je/jne， 对于算数有 add/sub/mul/div/neg/mod/xor 等
        filter.h 中还定义了额外的一些linux上的扩展，用于获取 socket buffer 的额外信息等。

   - address mode
        
        一共有 12 种，之前疑惑的 [k] 这样的是mode 4，代表 `BHW at byte offset k in the packet`。
        看看其它访问内存的mode，都是以数据包的地址作为基础地址。这么说起来 cBPF 只是为了抓包而生的，因为别的内存它都访问不到。

        返回支持返回立即数和A寄存器。

参考[kernel文档](https://www.kernel.org/doc/html/latest/networking/filter.html)

- `host 8888` bpf code分析
  
  现在分析这个就很直观了,伪代码大致如下：
  
  ```c
    int filter(char *pkt) {
        if (((u16*)ptr)[12/2] == 0x86dd) {
            // ipv6 case
            if (ptr[20] == 0x11||ptr[20] == 0x6 || ptr[20] == 0x84) {
                // tcp, udp or stcp
                u16 sport = *(u16*)&ptr[54];
                u16 dport = *(u16*)&ptr[56];
                if (sport == 8888 || dport == 8888) {
                    return 262144;
                }
            }
        } else if (((u16*)ptr)[12/2] == 0x0800) {
            // ipv4
            if (ptr[23] == 0x11||ptr[23] == 0x6 || ptr[23] == 0x84) {
                if (*(u16*)&ptr[20]&0x1fff == 0) {
                    // fragment offset == 0
                    u8 hdr_len = ptr[14]*0xf * 4;
                    u16 sport = *(u16*)&ptr[14+hdr_len];
                    u16 dport = *(u16*)&ptr[14+hdr_len];
                    if (sport == 8888 || dport == 8888) {
                        return 262144;
                    }
                }
            }
        }
        return 0
    }
  ```

### kernel bpf attach
`
setsockopt(4, SOL_SOCKET, SO_ATTACH_FILTER, {len=24, filter=0x55ea646ae0d0}, 16) = 0
`

这里，应该就是把上面的BPF code attach到kernel。不过内核似乎不是真正的接受这种 BPF 码。
它接受了一个结构体，描述了这些码的格式。
```c
struct sock_filter {    /* Filter block */
        __u16   code;   /* Actual filter code */
        __u8    jt;     /* Jump true */
        __u8    jf;     /* Jump false */
        __u32   k;      /* Generic multiuse field */
};

struct sock_fprog {                     /* Required for SO_ATTACH_FILTER. */
        unsigned short             len; /* Number of filter blocks */
        struct sock_filter __user *filter;
};
```

` {len=24, filter=0x55ea646ae0d0} ` 看这个参数，说明tcpdump传递给内核的一共是 24 个 sock_filter{}
其中的内容也可以和我们上面的分析对应：一共24条指令，一条指令对应一个结构体。

socket—fd 已经有 网卡的各种信息，那么具体的这个filter是被attach到什么点呢？以及kernel是怎么运行这个code的？
希望能够继续深入研究一下。

- SO_ATTACH_FILTER attach point
        
    这里除了看代码似乎没有什么别的好办法，大概画个流程图吧。

    整个流程最主要的是将用户态传入的程序进行检查，然后选择一个jit runtime进行运行。
    最终把socket的sk_filter置为jit code。

    而这个socket又是什么呢？在 net/socket.c 中可以看到，是 socket 这种fd的 privdata。在sys_socket系统调用时被创建。
    具体的来说，通过 __sock_create 创建。

    通过strace可以看出前面tcpdump创建的net_family为AF_PACKET,由此我们找到对应的create函数为 net / packet / af_packet.c :: packet_create

    create 动作比较复杂，不过我觉得主要关心 `po->prot_hook.func = packet_rcv` 这一行就可以。

    不对，set tx/rx ring的方法会把这个callback改为  tpacket_rcv，或许这才是我们要关注的。
```
setsockopt(syscall) -> __sys_setsockopt -> do_sock_setsockopt -> sock_setsockopt -> sk_setsockopt
        -> sk_attach_filter ->  __get_filter (jit happened)
                            -> __sk_attach_prog
```

- callback point:  tpacket_rcv

    tpacket_rcv 调用 run_filter, 最终一路调用到run bpf code。获取完res值之后，如果res > 0,就说明这个包需要拷贝。

    这里调用了skb_copy_bit(),把 skb 的数据拷贝到了ringbuf中，然后把status置上 TP_STATUS_USER 。这样，用户态就能访问到数据了。

    那么这个tpacket_rcv会在哪里被调用呢？答案在 dev_add_pack, 对于 AF_PACKET/PF_PACKET，会在这种类型下注册一下这个packet_type

    由此当 __netif_receive_skb_core 收到包时，首先进行 do_xdp_generic(通用xdp抓包点，也就是软件模拟的xdp), 
    然后就到了所谓的tc点，也就是这种dev的packet type 进行 deliver_skb
    ```c
    list_for_each_entry_rcu(ptype, &dev_net_rcu(skb->dev)->ptype_all,
				list) {
		if (pt_prev)
			ret = deliver_skb(skb, pt_prev, orig_dev);
		pt_prev = ptype;
	}

    static inline int deliver_skb(struct sk_buff *skb,
			      struct packet_type *pt_prev,
			      struct net_device *orig_dev)
    {
        if (unlikely(skb_orphan_frags_rx(skb, GFP_ATOMIC)))
            return -ENOMEM;
        refcount_inc(&skb->users);
        return pt_prev->func(skb, skb->dev, pt_prev, orig_dev);
    }
    ```


- kernel jit compiler
  从 __get_filter 开始，一路调用到 bpf_jit_compile，这个后面有需要再深入研究吧。

## dpdk ebpf support

源代码位于 `app/test/test_bpf.c` 中，描述了一组 ebpf 测例。
实现代码大部分位于 lib/librte_bpf 下面，目前支持了两种模式，jit 和 interpreter。

但是这个代码目前看起来只是实现了执行的部分，更具体的，比如说想要把这个程序attach到某个netdev，以及ebpf map，似乎没有很好的支持。

我觉得这个可能更多的是做一些抓包工具，可能稍微修改一下就能使得现在的 tcpdump 工作在dpdk上。

[文档]()也比较简陋,在后面看看是否能够作一些相关的有趣的事情。