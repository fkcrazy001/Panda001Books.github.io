# ERDMA

实现一个自己的 RDMA 虚拟网卡，纯软件模拟，基于 Rust 开发。类似 soft-roce，让普通以太网卡支持 RDMA/RoCE 协议。

## 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户态 (User Space)                        │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              Rust ibverbs 库 (erdmaverbs)                   │  │
│  │  - Verbs API 实现                                           │  │
│  │  - CQ/WQ/SQ 管理                                            │  │
│  │  - MR/PD 内存注册                                            │  │
│  │  - io_uring 网络 I/O 层                                      │  │
│  └─────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        物理网卡 (Physical NIC)                    │
│  - 普通以太网卡 (Ethernet NIC)                                   │
│  - 通过 io_uring 发送 RoCEv2 报文                                │
└─────────────────────────────────────────────────────────────────┘
```

## 技术选型

| 方案 | 选择 | 理由 |
|------|------|------|
| io_uring 用户态 | ✅ | 开发效率高、调试容易、生态成熟 |
| Rust 内核模块 | ⏸️ 待研究 | 研究价值高，但开发周期长 |

---

# 方案一：io_uring 用户态实现 (进行中)

## 架构设计

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户态 (User Space)                        │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                    应用层 (Application)                      │  │
│  │   rdma-core 工具 / 自研应用 / TensorFlow / Spark 等         │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                    dlopen / 链接 liberdmaverbs                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │              erdmaverbs (Rust 实现的 Verbs API)              │  │
│  │                                                               │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │  │
│  │  │ Device/PD    │  │ MR (内存注册)│  │ CQ (完成队列)       │  │  │
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘  │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │  │
│  │  │ QP (队列对)  │  │ Work Request │  │ Completion          │  │  │
│  │  │  - SQ 发送   │  │  - Send/Recv │  │  - WC 轮询          │  │  │
│  │  │  - RQ 接收   │  │  - RDMA操作  │  │  - 事件通知         │  │  │
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                    共享内存 + eventfd                            │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │              io_uring I/O 层                                 │  │
│  │                                                               │  │
│  │  ┌─────────────────────┐  ┌─────────────────────────────┐  │  │
│  │  │   Submit Queue (SQ)  │  │      Completion Queue (CQ)    │  │  │
│  │  │   - sendmsg SQE      │  │      - recvmsg CQE           │  │  │
│  │  │   - recvmsg SQE      │  │      - send completions      │  │  │
│  │  └─────────────────────┘  └─────────────────────────────┘  │  │
│  │                                                               │  │
│  │  ┌─────────────────────┐  ┌─────────────────────────────┐  │  │
│  │  │  Registered Buffers  │  │    Fixed Files (sock_fds)    │  │  │
│  │  │  (零拷贝发送/接收)    │  │    (避免每次系统调用)        │  │  │
│  │  └─────────────────────┘  └─────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                    Socket 层                                │  │
│  │          UDP Socket (Port 4791) + IP 协议栈                 │  │
│  └─────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        以太网 (Ethernet)                          │
│              普通以太网卡发送 RoCEv2 报文 (UDP Port 4791)          │
└─────────────────────────────────────────────────────────────────┘
```

## io_uring 关键特性

### 提交队列 (Submit Queue)

```rust
// 使用 IORING_SETUP_SQPOLL 让内核轮询，减少用户态开销
let ring = IoUring::new(32, 
    IoUringBuilder::new()
        .setup_sqpoll(Duration::from_millis(100))  // 内核 100ms 轮询
        .setup_iopoll()  // 轮询 I/O 完成
)?;

// 批量提交 SQE，减少系统调用开销
let mut sq = ring.submission();
let mut sqes = Vec::with_capacity(16);
for _ in 0..16 {
    sqes.push(sq.next());
}
// 一次性提交
sq.submit()?;
```

### 完成队列 (Completion Queue)

```rust
let mut cq = ring.completion();

// 批量轮询 CQE
let mut completions = Vec::with_capacity(32);
for _ in 0..32 {
    if let Some(cqe) = cq.next() {
        completions.push(cqe);
    } else {
        break;
    }
}
```

### Registered Buffers (零拷贝)

```rust
// 注册发送/接收缓冲区，避免内核拷贝
let send_buf: &[u8] = &[0u8; 65536];
let recv_buf: &mut [u8] = &mut [0u8; 65536];

// 注册到 io_uring
let reg = ring.registration();
let buf_idx = reg.register_buffers(vec![send_buf])?;
let recv_idx = reg.register_buffers(vec![recv_buf])?;

// 使用固定缓冲区发送
let sqe = sq.next();
sqe.prep_send_zc(buf_idx, socket_fd, 0, 0);
sqe.set_flags(SQE::FIXED_FILE.bits());
```

### Fixed Files

```rust
// 注册 socket fd，避免每次操作都传递 fd
let socket_fd = libc::socket(libc::AF_INET, libc::SOCK_DGRAM, 0);
let fixed_fd = reg.register_files(&[socket_fd])?;

let sqe = sq.next();
sqe.prep_recv(fixed_fd[0], recv_idx, 0, 0);
```

## 实现步骤

### 阶段 0：环境搭建 (第 1 周)

**目标**：能在终端打印 "Hello io_uring"

主要工作：
- 安装 Rust toolchain (1.70+)
- 配置开发环境 (Ubuntu 22.04+ / Kernel 5.10+)
- 安装 liburing 开发库
- 创建 Cargo 项目
- 编译运行 hello world 示例

里程碑：运行 `./target/debug/erdma "Hello io_uring"`

参考代码：
```rust
use io_uring::IoUring;

fn main() -> std::io::Result<()> {
    let ring = IoUring::new(32, &Default::default())?;
    println!("io_uring initialized with {} entries", ring.entries());
    Ok(())
}
```

### 阶段 1：基础框架 (第 2 周)

**目标**：项目结构搭建完成，核心模块定义好

主要工作：
```
erdmaverbs/
├── Cargo.toml
├── src/
│   ├── lib.rs          # 库入口，模块组织
│   ├── device.rs       # Device/Context 管理
│   ├── pd.rs           # Protection Domain
│   ├── mr.rs           # Memory Region
│   ├── cq.rs           # Completion Queue
│   ├── qp.rs           # Queue Pair
│   ├── wr.rs           # Work Request
│   ├── wc.rs           # Work Completion
│   ├── roce.rs         # RoCEv2 协议栈
│   │   ├── bth.rs      # BTH 头部
│   │   ├── aeth.rs     # AETH 头部
│   │   ├── deth.rs     # DETH 头部
│   │   └── checksum.rs # 校验和计算
│   ├── io_uring.rs     # io_uring 封装
│   └── util.rs         # 工具函数
```

里程碑：`cargo build` 通过

### 阶段 2：RoCEv2 协议栈 (第 3-5 周)

**目标**：能发送标准 RoCEv2 报文，Wireshark 可验证

#### 2.1 IB 传输层头部 (第 3 周)

**BTH (Base Transport Header)**
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   opcode      |1|1|0|0|  ver  |         p_key                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         reserved          |          dest_qp                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|1|                    psn                                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

```rust
#[repr(C, packed)]
struct BthHeader {
    opcode: u8,              // Bits 0-7: opcode
    solicited_event: u8,     // Bit 0: solicited event, Bit 1: mig_req, Bits 2-5: pad_count, Bits 6-7: transport_version
    p_key: u16,              // Partition Key
    reserved: u8,            // Reserved
    dest_qp: u32,            // Destination QP (bits 0-23)
    ack_req: u8,            // Bit 0: ACK request, Bits 1-7: PSN high bits
    psn: u32,               // Packet Sequence Number (lower 24 bits)
}

impl BthHeader {
    const SIZE: usize = 12;
    
    fn new(opcode: u8, dest_qp: u32, psn: u32, ack_req: bool) -> Self {
        Self {
            opcode,
            solicited_event: 0,
            p_key: 0xffff,  // Default p_key
            reserved: 0,
            dest_qp,
            ack_req: if ack_req { 0x80 } else { 0 },
            psn: psn & 0xffffff,
        }
    }
    
    fn to_le_bytes(&self) -> [u8; 12] {
        let mut bytes = [0u8; 12];
        bytes[0] = self.opcode;
        bytes[1] = self.solicited_event;
        bytes[2..4].copy_from_slice(&self.p_key.to_le_bytes());
        bytes[4] = self.reserved;
        bytes[5..8].copy_from_slice(&(self.dest_qp as u32 & 0xffffff).to_le_bytes());
        bytes[8] = self.ack_req;
        bytes[9..12].copy_from_slice(&(self.psn as u32 & 0xffffff).to_le_bytes());
        bytes
    }
}
```

**AETH (Acknowledgment Extended Transport Header)**
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     syndrome   |                    msn                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

```rust
#[repr(C, packed)]
struct AethHeader {
    syndrome: u8,    // 0 = ACK, 1 = NACK, 3 = RNR NACK
    msn: u32,        // Message Sequence Number
}

impl AethHeader {
    const SIZE: usize = 5;
    
    fn ack(msn: u32) -> Self {
        Self { syndrome: 0, msn }
    }
    
    fn nack(msn: u32) -> Self {
        Self { syndrome: 1, msn }
    }
}
```

**DETH (Destination Extended Transport Header)**
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         q_key                  |          src_qp              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

```rust
#[repr(C, packed)]
struct DethHeader {
    q_key: u32,
    src_qp: u32,
}

impl DethHeader {
    const SIZE: usize = 8;
    
    fn new(q_key: u32, src_qp: u32) -> Self {
        Self { q_key, src_qp }
    }
}
```

里程碑：构造包含 BTH+AETH+DETH 的报文，字节序正确

#### 2.2 UDP/IP 封装 (第 4 周)

```rust
#[repr(C, packed)]
struct UdpHeader {
    src_port: u16,
    dst_port: u16,
    length: u16,
    checksum: u16,
}

#[repr(C, packed)]
struct IpHeader {
    version_ihl: u8,     // Version (4 bits) + IHL (4 bits)
    dscp_ecn: u8,
    total_length: u16,
    identification: u16,
    flags_fragment: u16,
    ttl: u8,
    protocol: u8,        // UDP = 17
    header_checksum: u16,
    src_addr: u32,
    dst_addr: u32,
}

impl IpHeader {
    const SIZE: usize = 20;
    
    fn new(src_addr: u32, dst_addr: u32, payload_len: u16) -> Self {
        Self {
            version_ihl: 0x45,  // IPv4, IHL = 5
            dscp_ecn: 0,
            total_length: (Self::SIZE + payload_len) as u16,
            identification: 0,
            flags_fragment: 0,
            ttl: 64,
            protocol: 17,  // UDP
            header_checksum: 0,  // Will compute
            src_addr,
            dst_addr,
        }
    }
}
```

RoCEv2 UDP 端口：4791

里程碑：能发送 UDP 包，Wireshark 抓到 UDP 4791 端口

#### 2.3 以太网帧封装 (第 5 周)

```rust
#[repr(C, packed)]
struct EthernetHeader {
    dst_mac: [u8; 6],
    src_mac: [u8; 6],
    ether_type: u16,  // 0x8915 for RoCE
}

impl EthernetHeader {
    const SIZE: usize = 14;
    const ROCE_ETHERTYPE: u16 = 0x8915;
    
    fn new(dst_mac: &[u8; 6], src_mac: &[u8; 6]) -> Self {
        Self {
            dst_mac: *dst_mac,
            src_mac: *src_mac,
            ether_type: Self::ROCE_ETHERTYPE.to_be(),
        }
    }
}
```

里程碑：Wireshark 抓到完整的 RoCEv2 报文 (Ethernet + IP + UDP + BTH)

### 阶段 3：内存注册 (MR) 与 Protection Domain (PD) (第 6-7 周)

**目标**：能够注册内存区域，返回正确的 lkey/rkey

```rust
pub struct MemoryRegion {
    addr: usize,           // 虚拟地址
    length: usize,        // 长度
    lkey: u32,            // 本地访问密钥
    rkey: u32,            // 远程访问密钥
    access: AccessFlags,  // 访问权限
    pd: Arc<ProtectionDomain>,
}

impl MemoryRegion {
    pub fn new(
        pd: &Arc<ProtectionDomain>,
        addr: usize,
        length: usize,
        access: AccessFlags,
    ) -> Result<Arc<Self>, Error> {
        // 分配 lkey/rkey
        let lkey = pd.alloc_lkey(access)?;
        let rkey = lkey;  // 简单实现，lkey == rkey
        
        Ok(Arc::new(Self {
            addr,
            length,
            lkey,
            rkey,
            access,
            pd: Arc::clone(pd),
        }))
    }
    
    pub fn to_sge(&self, offset: usize, length: u32) -> Sge {
        Sge {
            addr: (self.addr + offset) as u64,
            length,
            lkey: self.lkey,
        }
    }
}

pub struct ProtectionDomain {
    ctx: Arc<Context>,
    pd_handle: u32,
    lkey_table: BTreeMap<u32, Arc<MemoryRegion>>,
    rkey_table: BTreeMap<u32, Arc<MemoryRegion>>,
    next_lkey: AtomicU32,
}

impl ProtectionDomain {
    pub fn alloc_lkey(&self, access: AccessFlags) -> Result<u32, Error> {
        let lkey = self.next_lkey.fetch_add(4, Ordering::SeqCst);
        Ok(lkey)
    }
}
```

里程碑：`ibv_reg_mr` 返回正确的 lkey，WQE 中的 SGE lkey 正确

### 阶段 4：QP (Queue Pair) 状态机 (第 8-10 周)

**目标**：QP 能在 RESET → INIT → RTR → RTS 状态间转换

```rust
pub enum QpState {
    Reset,
    Init,
    Rtr,   // Ready to Receive
    Rts,   // Ready to Send,
    Error,
}

pub struct QueuePair {
    pd: Arc<ProtectionDomain>,
    qp_num: u32,
    state: AtomicU8,
    sq: Vec<WorkRequest>,    // Send Queue
    rq: Vec<WorkRequest>,    // Receive Queue
    next_psn: AtomicU32,
    remote_qp_num: AtomicU32,
    remote_addr: AtomicU64,
    remote_rkey: AtomicU32,
}

impl QueuePair {
    pub fn create(
        pd: &Arc<ProtectionDomain>,
        max_sge: usize,
        max_wr: usize,
    ) -> Result<Arc<Self>, Error> {
        let qp_num = pd.ctx.alloc_qp_num()?;
        
        Ok(Arc::new(Self {
            pd: Arc::clone(pd),
            qp_num,
            state: AtomicU8::new(QpState::Reset as u8),
            sq: Vec::with_capacity(max_wr),
            rq: Vec::with_capacity(max_wr),
            next_psn: AtomicU32::new(0),
            remote_qp_num: AtomicU32::new(0),
            remote_addr: AtomicU64::new(0),
            remote_rkey: AtomicU32::new(0),
        }))
    }
    
    pub fn modify_qp(
        &self,
        attr: &QpAttr,
        attr_mask: QpAttrMask,
    ) -> Result<(), Error> {
        match (self.state.load(Ordering::SeqCst), attr.qp_state) {
            (QpState::Reset as u8, QpState::Init) => {
                self.state.store(QpState::Init as u8, Ordering::SeqCst);
                Ok(())
            },
            (QpState::Init as u8, QpState::Rtr) => {
                self.state.store(QpState::Rtr as u8, Ordering::SeqCst);
                Ok(())
            },
            (QpState::Rtr as u8, QpState::Rts) => {
                self.state.store(QpState::Rts as u8, Ordering::SeqCst);
                Ok(())
            },
            _ => Err(Error::InvalidQpStateTransition),
        }
    }
}
```

里程碑：`ibv_create_qp` → `ibv_modify_qp` 成功，状态转换正确

### 阶段 5：完成队列 (CQ) (第 11 周)

**目标**：能够 poll 到正确的 WC

```rust
pub enum WcStatus {
    Success,
    LocQpOperationErr,
    LocProtErr,
    WrFlushed,
    // ... 更多状态码
}

pub struct Wc {
    wr_id: u64,
    status: WcStatus,
    opcode: WcOpcode,
    vendor_err: u32,
    qp_num: u32,
    // ... 更多字段
}

pub struct CompletionQueue {
    cqe_size: usize,
    cqes: Vec<Wc>,
    producer_idx: AtomicUsize,
    consumer_idx: AtomicUsize,
}

impl CompletionQueue {
    pub fn poll(&self, count: usize) -> Result<Vec<Wc>, Error> {
        let mut wc_list = Vec::with_capacity(count);
        let consumer = self.consumer_idx.load(Ordering::SeqCst);
        
        for _ in 0..count {
            if consumer >= self.producer_idx.load(Ordering::SeqCst) {
                break;
            }
            // 读取 CQE
            let cqe = &self.cqes[consumer % self.cqe_size];
            if cqe.wr_id == 0 {
                break;  // 空 CQE
            }
            wc_list.push(*cqe);
            self.consumer_idx.store(consumer + 1, Ordering::SeqCst);
        }
        
        Ok(wc_list)
    }
    
    pub fn push(&self, wc: Wc) {
        let producer = self.producer_idx.fetch_add(1, Ordering::SeqCst);
        self.cqes[producer % self.cqe_size] = wc;
    }
}
```

里程碑：`ibv_poll_cq` 能返回正确的 WC

### 阶段 6：Work Request (第 12 周)

**目标**：能够正确构造和解析 WQE

```rust
pub enum WrOpcode {
    Send,
    SendWithImm,
    Write,
    WriteWithImm,
    Read,
    AtomicCmpAndSwap,
    FetchAndAdd,
    // ... 更多 opcode
}

pub struct Sge {
    pub addr: u64,    // 虚拟地址
    pub length: u32,   // 长度
    pub lkey: u32,     // 本地访问密钥
}

pub struct WorkRequest {
    pub wr_id: u64,
    pub opcode: WrOpcode,
    pub sges: Vec<Sge>,
    pub imm_data: Option<u32>,  // Immediate Data
    pub remote_addr: u64,        // RDMA Read/Write 目标地址
    pub remote_rkey: u32,        // RDMA 访问密钥
    pub send_flags: SendFlags,
}

pub struct RecvWr {
    pub wr_id: u64,
    pub sges: Vec<Sge>,
}
```

里程碑：`ibv_post_send` / `ibv_post_recv` 成功，WQE 格式正确

### 阶段 7：集成 io_uring (第 13-14 周)

**目标**：完整的发送/接收流程

```rust
pub struct IoUringContext {
    ring: IoUring<Arc<()>>,  // Arc 确保跨线程存活
    socket_fd: RawFd,
    registered_buffers: Vec<Vec<u8>>,
    registered_files: Vec<RawFd>,
    send_queue: Vec<WorkRequest>,
    recv_queue: Vec<WorkRequest>,
}

impl IoUringContext {
    pub fn new(iface: &str) -> Result<Self, Error> {
        // 创建 UDP socket
        let socket_fd = unsafe {
            libc::socket(libc::AF_INET, libc::SOCK_DGRAM, libc::IPPROTO_UDP)
        }?;
        
        // 绑定到 iface
        // ...
        
        let ring = IoUring::new(1024, &IoUringBuilder::new().setup_sqpoll(100))?;
        let reg = ring.registration();
        
        Ok(Self {
            ring,
            socket_fd,
            registered_buffers: Vec::new(),
            registered_files: vec![socket_fd],
            send_queue: Vec::new(),
            recv_queue: Vec::new(),
        })
    }
    
    pub fn send(&self, data: &[u8], dest: &SocketAddr) -> Result<usize, Error> {
        let sq = self.ring.submission();
        let mut sqes = sq.next();
        
        // 构造 RoCEv2 报文
        let packet = self.build_roce_packet(data, dest)?;
        
        // 准备 sendmsg SQE
        let msghdr = self.build_msghdr(&packet, dest);
        sqes.prep_sendmsg(self.socket_fd, &msghdr, 0);
        
        sq.submit()?;
        
        // 等待完成
        let cq = self.ring.completion();
        for _ in 0..1000 {  // 最多等待 1s
            if let Some(cqe) = cq.next() {
                if cqe.result() >= 0 {
                    return Ok(cqe.result() as usize);
                }
                return Err(Error::IoError(cqe.result()));
            }
            std::thread::sleep(std::time::Duration::from_millis(1));
        }
        
        Err(Error::Timeout)
    }
    
    pub fn recv(&self, buf: &mut [u8]) -> Result<usize, Error> {
        let sq = self.ring.submission();
        let mut sqes = sq.next();
        
        let mut src_addr: libc::sockaddr = unsafe { std::mem::zeroed() };
        let mut addr_len: libc::socklen_t = std::mem::size_of::<libc::sockaddr>() as libc::socklen_t;
        
        sqes.prep_recv(self.socket_fd, buf.as_mut_ptr() as *mut _, buf.len(), 0);
        
        sq.submit()?;
        
        // 等待完成
        let cq = self.ring.completion();
        if let Some(cqe) = cq.next() {
            if cqe.result() >= 0 {
                return Ok(cqe.result() as usize);
            }
        }
        
        Err(Error::IoError(-1))
    }
}
```

里程碑：两端通过 io_uring 成功收发 RoCEv2 报文

### 阶段 8：ibverbs API 完整实现 (第 15-16 周)

**目标**：完整的 C ABI 兼容 API

```rust
use std::os::raw::{c_int, c_uint, c_void, c_ulong};
use std::ptr::{null, null_mut};

// C ABI 兼容类型
pub type ibv_device = c_void;
pub type ibv_context = c_void;
pub type ibv_pd = c_void;
pub type ibv_mr = c_void;
pub type ibv_cq = c_void;
pub type ibv_qp = c_void;
pub type ibv_wr = c_void;
pub type ibv_wc = c_void;

// Flags
pub const IBV_ACCESS_LOCAL_WRITE: c_uint = 0x0001;
pub const IBV_ACCESS_REMOTE_WRITE: c_uint = 0x0002;
pub const IBV_ACCESS_REMOTE_READ: c_uint = 0x0004;
pub const IBV_ACCESS_REMOTE_ATOMIC: c_uint = 0x0008;
pub const IBV_ACCESS_MW_BIND: c_uint = 0x0010;
pub const IBV_ZERO_BASED: c_uint = 0x0020;

// QP State
pub const IBV_QPS_RESET: c_int = 0;
pub const IBV_QPS_INIT: c_int = 1;
pub const IBV_QPS_RTR: c_int = 2;
pub const IBV_QPS_RTS: c_int = 3;

// API 函数声明
#[link(name = "erdmaverbs")]
extern "C" {
    pub fn ibv_open_device(device: *mut ibv_device) -> *mut ibv_context;
    pub fn ibv_close_device(context: *mut ibv_context) -> c_int;
    pub fn ibv_alloc_pd(context: *mut ibv_context) -> *mut ibv_pd;
    pub fn ibv_dealloc_pd(pd: *mut ibv_pd) -> c_int;
    pub fn ibv_reg_mr(
        pd: *mut ibv_pd,
        addr: *mut c_void,
        length: usize,
        access: c_int,
    ) -> *mut ibv_mr;
    pub fn ibv_dereg_mr(mr: *mut ibv_mr) -> c_int;
    pub fn ibv_create_cq(
        context: *mut ibv_context,
        cqe: c_int,
        cq_context: *mut c_void,
        channel: *mut c_void,
        comp_vector: c_int,
    ) -> *mut ibv_cq;
    pub fn ibv_destroy_cq(cq: *mut ibv_cq) -> c_int;
    pub fn ibv_create_qp(
        pd: *mut ibv_pd,
        attr: *mut QpInitAttr,
    ) -> *mut ibv_qp;
    pub fn ibv_destroy_qp(qp: *mut ibv_qp) -> c_int;
    pub fn ibv_modify_qp(
        qp: *mut ibv_qp,
        attr: *mut QpAttr,
        attr_mask: c_int,
    ) -> c_int;
    pub fn ibv_post_send(
        qp: *mut ibv_qp,
        wr: *mut ibv_wr,
        bad_wr: *mut *mut ibv_wr,
    ) -> c_int;
    pub fn ibv_post_recv(
        qp: *mut ibv_qp,
        wr: *mut ibv_wr,
        bad_wr: *mut *mut ibv_wr,
    ) -> c_int;
    pub fn ibv_poll_cq(
        cq: *mut ibv_cq,
        num: c_int,
        wc: *mut ibv_wc,
    ) -> c_int;
    pub fn ibv_req_notify_cq(
        cq: *mut ibv_cq,
        solicited_only: c_int,
    ) -> c_int;
}
```

里程碑：`ibv_devinfo` 能识别 erdma 设备

### 阶段 9：测试与调优 (第 17-20 周)

**目标**：完整的 ping-pong 测试

```
┌─────────────────────────────────────────────────────────────────┐
│                      ping-pong 测试拓扑                           │
│                                                                      │
│    Server                                         Client           │
│   ┌──────────┐                                   ┌──────────┐     │
│   │  QP 0    │ ════════════════════════════════ │  QP 0    │     │
│   │  (RTS)   │     RDMA Send/Recv              │  (RTR)   │     │
│   └──────────┘                                   └──────────┘     │
│       │                                               │              │
│       │       RoCEv2 over UDP (Port 4791)            │              │
│       └────────────────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────────────┘
```

```rust
// ping-pong server
fn run_server(ctx: &Arc<Context>, port: u16) -> Result<(), Error> {
    let pd = ibv_alloc_pd(ctx)?;
    let mr = ibv_reg_mr(pd, BUF, BUF.len(), IBV_ACCESS_LOCAL_WRITE)?;
    
    let cq = ibv_create_cq(ctx, 128, null(), null(), 0)?;
    let qp = ibv_create_qp(pd, &QpInitAttr::new(cq, cq))?;
    
    // Configure QP to RTR then RTS
    let mut qp_attr = QpAttr::new();
    qp_attr.qp_state = IBV_QPS_INIT;
    qp_attr.pkey_index = 0;
    qp_attr.port_num = 1;
    ibv_modify_qp(qp, &qp_attr, IBV_QP_STATE)?;
    
    qp_attr.qp_state = IBV_QPS_RTR;
    qp_attr.path_mtu = 1024;
    qp_attr.dest_qp_num = client_qp_num;
    qp_attr.rq_psn = 0;
    ibv_modify_qp(qp, &qp_attr, IBV_QP_STATE)?;
    
    qp_attr.qp_state = IBV_QPS_RTS;
    qp_attr.sq_psn = 0;
    ibv_modify_qp(qp, &qp_attr, IBV_QP_STATE)?;
    
    // Post receive
    let recv_wr = build_recv_wr(mr.sge());
    ibv_post_recv(qp, &recv_wr, std::ptr::null_mut())?;
    
    // Poll for completion
    let mut wc: ibv_wc = unsafe { std::mem::zeroed() };
    while ibv_poll_cq(cq, 1, &mut wc) == 0 {
        std::thread::sleep(Duration::from_micros(100));
    }
    
    println!("Server received: {} bytes", wc.byte_len);
    
    Ok(())
}
```

里程碑：
- [ ] ping-pong 测试成功 (Send/Recv)
- [ ] RDMA Write/Read 测试成功
- [ ] Wireshark 报文分析正确
- [ ] 延迟 < 100μs (baseline: soft-roce)

---

# 方案二：Rust 内核模块实现 (待研究)

## 架构设计

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户态 (User Space)                        │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              Rust ibverbs 库                                │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                    字符设备 (/dev/erdma)                     │  │
│  │                    ioctl 控制接口                            │  │
│  └─────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        内核态 (Kernel Space)                      │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │              Rust 内核模块 (erdma.ko)                         │  │
│  │  - RoCEv2 协议栈                                              │  │
│  │  - QP 状态机                                                  │  │
│  │  - 内存注册与 DMA 映射                                        │  │
│  │  - raw socket 发送/接收                                       │  │
│  └─────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## 待研究内容

### 环境搭建
- rust-for-linux 项目状态
- 内核版本兼容性
- 编译工具链配置

### 核心技术挑战
- Rust 内核编程约束
- 内存安全与内核兼容性
- 调试与错误排查
- 性能优化空间

### 研究方向
- Rust ownership 在内核场景的应用
- 内存安全 vs 性能权衡
- 与现有 C 代码的互操作

## 参考项目

- [rust-for-linux](https://github.com/Rust-for-Linux/linux)
- [soft-roce](https://github.com/SoftRoCE/rxe-rdma-kernel)
- [Linux RDMA 子系统文档](https://www.kernel.org/doc/Documentation/rdma/)

## 启动条件

- io_uring 方案完成基础功能后
- 或有充足时间 (6+ 个月)
- 或有明确的学术研究目标

---

# 附录

## 技术栈总结

| 层级 | 技术 | 用途 |
|------|------|------|
| 运行时 | Rust stable + tokio | 异步编程 |
| I/O | io_uring (liburing) | 高性能网络 I/O |
| 系统调用 | libc / nix | POSIX 接口 |
| 协议栈 | 手写 RoCEv2 | IB over UDP |
| 测试 | Wireshark | 报文分析 |
| 性能 | perf / eBPF | 性能分析 |

## 参考资料

### io_uring
- [io_uring 官方文档](https://unixism.net/2020/04/io-uring-by-example-1-introduction-to-io_uring/)
- [liburing 源码](https://github.com/axboe/liburing)
- [io_uring-tokio 集成](https://github.com/tokio-rs/io-uring)

### RoCEv2 / IB
- [IB Spec 1.5](https://www.infiniband.org/specs/)
- [RoCEv2 协议规范](https://www.infiniband.org/specs/roce/)
- [RDMAmojo](https://www.rdmamojo.com/)

### RDMA 参考实现
- [soft-roce (rxe)](https://github.com/SoftRoCE/rxe-rdma-kernel)
- [DPDK rte_ethdev](https://doc.dpdk.org/guides/prog_guide/eth_app.html)
- [rdma-core](https://github.com/linux-rdma/rdma-core)

### Rust
- [Rust 异步编程](https://rust-lang.github.io/async-book/)
- [Tokio 教程](https://tokio.rs/tokio/tutorial)
