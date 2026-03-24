# cilium

## 编译部署

基于ubuntu24.04 + docker(no-privileged) + kind
1. `make kind` or others.具体参见`Makefile.kind`文件。`make kind`启动一个最基本的环境（一主一从）
   1. `make kind-down`, 停止kind集群。
   2. 创建完之后进入node：`docker exec -it kind-control-plane -- /bin/bash`
      1. `echo "nameserver 223.5.5.5" > /etc/resolv.conf`，方便拉取镜像。
2. 构建镜像
   1. `make kind-image`，基于`docker build`，需要拉取base镜像，，
   2. Linux环境下，本地编译，并进行文件挂载替换：`make kind-image-fast`。基于kind的一些特性（kind在宿主机使用docker模拟node，node内部有containerd，可以用`crictl ps`查看node内部的容器状态），实际上是copy文件到node中。
   3. 构建完镜像同时会在本地生成编译产物：
      1. 修改`plugins/cilium-cni/cmd/cmd.go`中的`version: "Cilium CNI plugin " + version.Version,`
      2. 编译完之后在本地执行:
         ```shell
         ➜  cilium git:(main) ✗ ./plugins/cilium-cni/cilium-cni
         tzx's Cilium CNI plugin 1.20.0-dev 5f29ff2a96 2026-03-11T10:01:52+00:00 go version go1.26.1 linux/amd64
         CNI protocol versions supported: 0.1.0, 0.2.0, 0.3.0, 0.3.1, 0.4.0, 1.0.0, 1.1.0
         ```
3. 安装
   1. `make kind-install-cilium` or `make kind-install-cilium-fast`
   2. `cilium status`查看状态，正常情况下应该全部OK

使用cilium-dbg分析：`kubectl -n kube-system exec ds/cilium -- cilium-dbg status --verbose`

## trace id

> trace_id_from_ip4()

- 仅限于ipv4，在ip头部的`options`字段中添加traceID。
- 在ebpf内核态程序的入口函数中调用`check_and_store_ip_trace_id()`，将traceID存储到`cilium_percpu_trace_id`的percpu_map中。
- 用户态程序`cilium_percpu_trace_id`使用cell进行依赖注入管理。pkg/maps/iptrace/iptrace_map.go
- 主动notify的机制，ebpf内核态代码中send_trace_notify，上报关键事件，有些事件会携带traceID。`trace_reason`

## datapath

数据包路由路径，抽象概念。

主体是eBPF内核态程序+cilium-agent操作的map。

## ebpf program type & hooks

> [hooks](https://docs.cilium.io/en/stable/internals/hooks/)

Cilium uses the following eBPF program types to attach programs to the kernel:

- ``BPF_PROG_TYPE_XDP``
- ``BPF_PROG_TYPE_SCHED_ACT``
- ``BPF_PROG_TYPE_CGROUP_SOCK_ADDR``

- XDP：
   - 挂载在宿主机物理网卡中，提前过滤流量，提前处理LB策略。
   - 如果流量需要进入上层网络栈，会调用`bpf_xdp_exit`，将流量传递给上层网络栈。TC？
- TC：
   - 在 Linux tc 语义里：
      - ingress：表示“进入该接口（从线/对端进入接口）的包”
      - egress：表示“从该接口发出去的包”
   - ```shell
     root@node1:~# bpftool net list -jp
      [{
            "xdp": [],
            "tc": [{
                     "devname": "ens18",
                     "ifindex": 2,
                     "kind": "tcx/ingress",
                     "name": "cil_from_netdev",
                     "prog_id": 1482,
                     "prog_flags": [],
                     "link_id": 7,
                     "link_flags": []
                  },{
                     "devname": "cilium_net",
                     "ifindex": 3,
                     "kind": "tcx/ingress",
                     "name": "cil_to_host",
                     "prog_id": 1475,
                     "prog_flags": [],
                     "link_id": 6,
                     "link_flags": []
                  },{
                     "devname": "cilium_host",
                     "ifindex": 4,
                     "kind": "tcx/ingress",
                     "name": "cil_to_host",
                     "prog_id": 1468,
                     "prog_flags": [],
                     "link_id": 4,
                     "link_flags": []
                  },{
                     "devname": "cilium_host",
                     "ifindex": 4,
                     "kind": "tcx/egress",
                     "name": "cil_from_host",
                     "prog_id": 1464,
                     "prog_flags": [],
                     "link_id": 5,
                     "link_flags": []
                  },{
                     "devname": "cilium_vxlan",
                     "ifindex": 5,
                     "kind": "tcx/ingress",
                     "name": "cil_from_overlay",
                     "prog_id": 1459,
                     "prog_flags": [],
                     "link_id": 2,
                     "link_flags": []
                  },{
                     "devname": "cilium_vxlan",
                     "ifindex": 5,
                     "kind": "tcx/egress",
                     "name": "cil_to_overlay",
                     "prog_id": 1460,
                     "prog_flags": [],
                     "link_id": 3,
                     "link_flags": []
                  },{
                     "devname": "lxc_health",
                     "ifindex": 167,
                     "kind": "tcx/ingress",
                     "name": "cil_from_container",
                     "prog_id": 1493,
                     "prog_flags": [],
                     "link_id": 90,
                     "link_flags": []
                  },{
                     "devname": "lxc1ae7fadfc998",
                     "ifindex": 169,
                     "kind": "tcx/ingress",
                     "name": "cil_from_container",
                     "prog_id": 1521,
                     "prog_flags": [],
                     "link_id": 91,
                     "link_flags": []
                  },{
                     "devname": "lxcbceb174c3a7a",
                     "ifindex": 221,
                     "kind": "tcx/ingress",
                     "name": "cil_from_container",
                     "prog_id": 4879,
                     "prog_flags": [],
                     "link_id": 117,
                     "link_flags": []
                  }
            ],
            "flow_dissector": [],
            "netfilter": []
         }
      ]
     ```

### BPF_PROG_TYPE_XDP

> bpf_xdp.c

大致逻辑：check_and_store_ip_trace_id（合法性检测+设置traceID） -> check_filters（直接处理LB） -> `return bpf_xdp_exit(ctx, ret);`（如果需要进入上层网络栈）

仅分析ipv4

1. 设置traceID
2. 查cilium_cidr_v4_fix 和 cilium_cidr_v4_dyn这两个map，判断是否需要drop。cilium_cidr_v4_fix为精确匹配ip地址， cilium_cidr_v4_dyn为lpm匹配ip地址前缀。这两个map的value没有意义。
3. prefilter_v4完成之后，check_v4_lb。通过tail_call机制调用`tail_lb_ipv4`。进行负载均衡（转发）策略。
4. 结合k8s的网络资源抽象，进行流量转发。例如NodePort、LoadBalancer等。NodePort需要将流量转发到后端的Pod上，有可能这个Pod不在当前node上。

LB的核心：XDP层直接处理NodePort和ClusterIP服务的负载均衡，避免流量进入上层网络栈，显著提升服务发现和转发的性能。

### BPF_PROG_TYPE_SCHED_ACT



### NodePort 

> nodeport.h

- ingress流量，`nodeport_lb4()`：
   - 流量从node的端口进入，然后被转发到后端的Pod上。
   - 三种情况：
      - 找到匹配的service，`nodeport_svc_lb4`：
         - 后端Pod在当前node上：直接传递到后端Pod。
         - 后端Pod不在当前node上：通过DSR机制，将流量直接转发到后端Pod；如果没有配置DSR，默认会进行SNAT转发。**直接egress**
      - 目的地不是svc ip：来自外部后端pod的回复流量。

- DSR机制：
   - Direct Server Return，直接返回后端Pod的回复流量，而不是通过node的端口返回。
   - 正常情况：当后端Pod不在当前node上时，当前node需要作为中间节点，将请求SNAT转发到后端pod对应的node上，再将回复流量返回给客户端。
   - DSR通过允许后端服务器直接响应客户端，后端能用 Service IP 直接回包，不再通过入口node做反向SNAT。回程少一跳。
   - DSR机制在入口node上不进行SNAT，后端Pod看到的源ip仍然是外部客户端的ip。
   - 难点：请求包目的 IP 必须是 Pod IP（否则路由不通/无法到达后端），而返回流量的源 IP 必须是 Service IP（否则客户端无法识别回复流量）（对外表现一致），并且直接回客户端。
   - cilium需要把service IP信息编码到请求包中传递给后端Pod。
      - opt机制：在ip头部的`options`字段中添加service IP信息。
      - geneve：入口节点将发往后端的报文再做一层 Geneve encapsulation
   - 字节跳动应用场景：返回数据包绕过LB，直接返回给客户端（通过公网）。降跳数 & LB负载
      - 请求：client -{client_ip;client_port->vip,vport}-> lb(网关) -{client_ip;client_port->pip,pport + ip option:vip,vport}-> node -{经过宿主机网络协议栈路由到pod}-> pod
      - **回复**：pod --> node(在主机侧SNAT) -{vip,vport->client_ip;client_port}-> client


ctx_{load,store}_meta():
- 宏的位置：`common.h::CB_SRC_LABEL`，enum，只有5个位置。
- 对于tc，使用了__sk_buff中的`cb`字段。[link](https://docs.ebpf.io/linux/program-context/__sk_buff/#cb)
  - tc程序中对__sk_buff结构体的修改会被直接映射到真正的sk_buff结构体中。__sk_buff仅仅是为了验证器检查程序是否安全，例如将一些字段设置成只读，防止ebpf程序修改这些字段。真正执行的时候操作的是sk_buff结构体。
- 对于xdp，使用了`cilium_xdp_scratch`这个map。

# 网卡

```shell
# 一主三从，node1中的网卡配置
root@node1:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: ens18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether bc:24:11:8f:a6:cb brd ff:ff:ff:ff:ff:ff
    altname enp0s18
    inet 192.168.100.61/24 brd 192.168.100.255 scope global noprefixroute ens18
       valid_lft forever preferred_lft forever
3: cilium_net@cilium_host: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 32:59:27:2f:a1:84 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::3059:27ff:fe2f:a184/64 scope link 
       valid_lft forever preferred_lft forever
4: cilium_host@cilium_net: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 56:84:40:76:36:ee brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.238/32 scope global cilium_host
       valid_lft forever preferred_lft forever
    inet6 fe80::5484:40ff:fe76:36ee/64 scope link 
       valid_lft forever preferred_lft forever
5: cilium_vxlan: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default 
    link/ether ba:96:d9:dc:11:91 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::b896:d9ff:fedc:1191/64 scope link 
       valid_lft forever preferred_lft forever
7: lxc35ed58852581@if6: <BROADCAST,MULTICAST> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether f2:60:d7:26:1a:a2 brd ff:ff:ff:ff:ff:ff link-netns cni-7cf0fb52-6b1e-f3b0-8916-b51a520d4c3b
167: lxc_health@if166: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether aa:1d:3b:e2:6e:6a brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::a81d:3bff:fee2:6e6a/64 scope link 
       valid_lft forever preferred_lft forever
169: lxc1ae7fadfc998@if168: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether ca:b8:d4:bf:d6:dc brd ff:ff:ff:ff:ff:ff link-netns cni-d399aa57-c436-2bc8-bffa-4098f89fd03c
    inet6 fe80::c8b8:d4ff:febf:d6dc/64 scope link 
       valid_lft forever preferred_lft forever
221: lxcbceb174c3a7a@if220: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether c2:9b:51:b9:e8:ce brd ff:ff:ff:ff:ff:ff link-netns cni-308d7ebf-6122-7488-2772-6dc47373c463
    inet6 fe80::c09b:51ff:feb9:e8ce/64 scope link 
       valid_lft forever preferred_lft forever
root@node1:~# ip r
default via 192.168.100.1 dev ens18 proto static metric 100 
10.0.0.0/24 via 10.0.1.238 dev cilium_host proto kernel src 10.0.1.238 mtu 1450 
10.0.1.0/24 via 10.0.1.238 dev cilium_host proto kernel src 10.0.1.238 
10.0.1.238 dev cilium_host proto kernel scope link 
10.0.2.0/24 via 10.0.1.238 dev cilium_host proto kernel src 10.0.1.238 mtu 1450 
10.0.3.0/24 via 10.0.1.238 dev cilium_host proto kernel src 10.0.1.238 mtu 1450 
192.168.100.0/24 dev ens18 proto kernel scope link src 192.168.100.61 metric 100
```

## cilium_net@cilium_host(veth pair)

```shell
3: cilium_net@cilium_host: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 32:59:27:2f:a1:84 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::3059:27ff:fe2f:a184/64 scope link 
       valid_lft forever preferred_lft forever
4: cilium_host@cilium_net: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 56:84:40:76:36:ee brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.238/32 scope global cilium_host
       valid_lft forever preferred_lft forever
    inet6 fe80::5484:40ff:fe76:36ee/64 scope link 
       valid_lft forever preferred_lft forever

root@node1:~# bpftool net show dev cilium_net
xdp:

tc:
cilium_net(3) tcx/ingress cil_to_host prog_id 1475 link_id 6 

flow_dissector:

netfilter:

root@node1:~# bpftool net show dev cilium_host
xdp:

tc:
cilium_host(4) tcx/ingress cil_to_host prog_id 1468 link_id 4 
cilium_host(4) tcx/egress cil_from_host prog_id 1464 link_id 5 

flow_dissector:

netfilter:
```

- 一对veth pair，都在宿主机的root network namespace中，
- Cilium 用这对 veth 把“Linux 主机协议栈（host stack）”与“Cilium 的主机数据面（host datapath）”逻辑隔离开，形成一个明确的“入口/出口”边界

- cilium_host：Cilium 的 “host datapath 锚点”（host IP /32 挂在这里）
   - 从 Linux 路由表视角：主机协议栈把去 Pod 的包都交给 cilium_host。让 host 上任何进程（kubelet、node-local 服务、hostNetwork Pod、调试 shell）要访问任意 Pod IP 时，都必然走入 Cilium 管控的数据面，而不是走 Linux 默认路由/iptables 的路径。
   - 看起来像“虚拟网关”，但底层的路由逻辑由bpf实现。
- cilium_net：连接主机协议栈的一侧（通常不配 IPv4，仅 link-local IPv6）
   - 作为与 Linux networking/路由更贴近的一侧，承接主机协议栈的流量，便于把某些路径（例如到 host stack、到路由）用 veth 明确隔开，减少对物理网卡/主路由的侵入。

## cilium_vxlan虚拟网卡

> [docs](https://docs.cilium.io/en/stable/network/ebpf/lifeofapacket/)

### 功能

- 在启用 overlay（VXLAN）时，流量会经过 overlay 设备（默认名 cilium_vxlan）”的那条路径：出方向经过 overlay egress，入方向经过 overlay ingress
- 承载跨节点转发

### cil_from_overlay

挂载在cilium_vxlan虚拟网卡的ingress接收路径，处理“从隧道进来的包”。

cilium_vxlan 是 Linux 内核的 VXLAN 设备。远端节点发来的 VXLAN UDP 报文到达本机后，内核会在 VXLAN 设备上解封装（decap）出内层二层帧（inner Ethernet frame），随后这些“内层包”会进入 cilium_vxlan 设备的接收路径，从而触发其 tc ingress hook。

Cilium 在这里挂 cil_from_overlay，就是为了在“解封装之后、送往本机 Pod/host 之前”接管数据面处理：识别这是 overlay 流量、恢复/计算 identity、做策略/CT/LB 相关的后续链路跳转（大量逻辑通过 tail call 继续）。这与社区对 from-overlay 的典型描述一致：vxlan decap 后进入 tc ingress（from-overlay）。

逻辑：当数据包从隧道设备进入时，cil_from_overlay()函数首先验证以太网类型，然后调用get_tunnel_key()从隧道头部提取密钥信息。这个密钥包含了发送端的身份标识，通过get_id_from_tunnel_id()解析后，可以识别数据包的来源节点。

### cil_to_overlay

挂载在cilium_vxlan虚拟网卡的egress发送路径，处理“要从隧道发出去的包”。

当本机某个 Pod 的流量需要跨节点到远端 Pod（你是 routing-mode=tunnel），Cilium 会在宿主机 datapath 中决定“目的 endpoint 在远端”，于是把包引导到 overlay 设备 cilium_vxlan 发出。

到了 cilium_vxlan 的发送路径，会触发其 tc egress hook，因此 Cilium 在这里挂 cil_to_overlay，就是为了在“交给内核 VXLAN 封装并发到物理网卡之前”做最后的 overlay 出方向处理。

逻辑：当数据包需要发往其他节点时，cil_to_overlay()函数负责封装处理。它会检查SNAT状态，设置隧道密钥，将数据包封装到隧道协议中，然后发送到目标节点。

### 例子

NodeA 上的 PodA（10.0.1.10）
NodeB 上的 PodB（10.0.2.20）
你使用 Cilium VXLAN tunnel

3.1 PodA -> PodB（出方向：会经过 cil_to_overlay）
PodA 发包，先在 NodeA 的 lxcXXXX（host 侧 veth）进入 Cilium（例如 cil_from_container）。
Cilium 查 ipcache / endpoint 信息发现：10.0.2.20 在 NodeB，走 tunnel。
Cilium 将包 redirect 到 cilium_vxlan。
包进入 cilium_vxlan tcx/egress，触发 cil_to_overlay（你看到的 prog_id=1460）。
cil_to_overlay 确认 tunnel 元信息后，放行给内核：内核对该包做 VXLAN 封装（外层 IP=NodeA 节点 IP，UDP dst=4789 等），随后在物理网卡 egress 发出（物理网卡上还会经过 cil_to_netdev 之类的 hook，这是 host datapath 部分）。

3.2 NodeB 收到并解封装（入方向：会经过 cil_from_overlay）
NodeB 物理网卡收到外层 VXLAN UDP 包，内核识别后交给 cilium_vxlan 设备解封装。
解封装得到内层包（dst=10.0.2.20），进入 cilium_vxlan tcx/ingress，触发 cil_from_overlay（你看到的 prog_id=1459）。
cil_from_overlay 恢复源 identity / trace 信息，并进入“ingress to endpoint”的 local delivery 逻辑。
最终包被送到 PodB 对应的 lxc veth，再进入 PodB netns 的 eth0。

## lxcXXXX@ifYY

```shell
169: lxc1ae7fadfc998@if168: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether ca:b8:d4:bf:d6:dc brd ff:ff:ff:ff:ff:ff link-netns cni-d399aa57-c436-2bc8-bffa-4098f89fd03c
    inet6 fe80::c8b8:d4ff:febf:d6dc/64 scope link 
       valid_lft forever preferred_lft forever
221: lxcbceb174c3a7a@if220: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether c2:9b:51:b9:e8:ce brd ff:ff:ff:ff:ff:ff link-netns cni-308d7ebf-6122-7488-2772-6dc47373c463
    inet6 fe80::c09b:51ff:feb9:e8ce/64 scope link 
       valid_lft forever preferred_lft forever
root@node1:~# bpftool net show dev lxc1ae7fadfc998
xdp:

tc:
lxc1ae7fadfc998(169) tcx/ingress cil_from_container prog_id 1521 link_id 91 

flow_dissector:

netfilter:
### pod内
root@node1:~# ip netns exec cni-d399aa57-c436-2bc8-bffa-4098f89fd03c ip -details link show dev eth0
168: eth0@if169: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether 4e:f1:a9:cd:9c:32 brd ff:ff:ff:ff:ff:ff link-netnsid 0 promiscuity 0  allmulti 0 minmtu 68 maxmtu 65535 
    veth addrgenmode eui64 numtxqueues 8 numrxqueues 8 gso_max_size 65536 gso_max_segs 65535 tso_max_size 524280 tso_max_segs 65535 gro_max_size 65536 
```

在 Linux tc 语义里：
- ingress：表示“进入该接口（从线/对端进入接口）的包”
- egress：表示“从该接口发出去的包”

对 lxc*（host 侧 veth）来说，它的“对端”就是容器：
- 当 容器发包：包从容器侧 eth0 进入 veth pair，来到宿主机侧 lxc* —— 这对 lxc* 来说就是 ingress。
- 所以 Cilium 把程序挂在 lxc* 的 ingress，并把这个入口命名为 cil_from_container，意思就是“从容器出来”的入口。

因此这句话可以精确翻译为：“所有从该 Pod 发出的流量，在到达宿主机 lxc1ae7... 的 ingress 时，会先进入 Cilium 的 cil_from_container 数据面入口。”

#### 例子 A：Pod 访问同节点的另一个 Pod（PodA -> PodB，同一台 node）
1) PodA 在容器内发包（dst=PodB IP）
2) 包到达宿主机侧 `lxc1ae7...` ingress
3) 触发 `cil_from_container`
4) Cilium 查 endpoint map / ipcache 发现 PodB 在本机
5) Cilium 将包转发到 PodB 对应的另一个 `lxcXXXX`，最终进入 PodB

**你能获得什么能力**：
- 对这个连接执行 egress policy / ingress policy（PodB 的 ingress policy 也会在后续链路检查）
- 做 CT、记录可观测性事件（drop/trace/flow log）

#### 例子 B：Pod 跨节点访问远端 Pod（你是 vxlan tunnel：PodA -> PodC（远端 node））
1) PodA 发包（dst=远端 PodC IP）
2) 到 `lxc1ae7...` ingress → `cil_from_container`
3) Cilium 查 ipcache 发现目的在远端 node
4) Cilium 把包 redirect 到 `cilium_vxlan`（然后你会在 `cilium_vxlan tcx/egress cil_to_overlay` 看到出方向入口参与）
5) 远端 node 解封装后在 `cilium_vxlan tcx/ingress cil_from_overlay` 进入远端 datapath，再送到 PodC

**为什么入口必须在 lxc ingress**：
- 只有这里最早知道“这是哪个 Pod 发出的”（能绑定到 endpoint identity），从而做 policy 与 observability
- 只有这里最早能决定“本机转发还是走 overlay”

#### 例子 C：Pod 访问 ClusterIP（在你集群里可能仍由 kube-proxy/iptables 实现，但 Pod 流量仍先进入 Cilium datapath）
1) PodA 发包到某个 Service ClusterIP
2) 先到 `lxc1ae7...` ingress → `cil_from_container`
3) 后续走向取决于你是否 kube-proxy replacement：
   - 你当前 `kube-proxy-replacement=false`，所以 service 转发很可能仍由 kube-proxy/iptables/ipvs 决定，但 Cilium 仍可以在 datapath 做 policy/CT/trace 等（具体取决于你启用的功能组合）
   - 如果未来你改为 kube-proxy replacement，那么 service 负载均衡也会在 Cilium BPF 链中完成
