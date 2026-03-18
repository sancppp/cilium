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

## trace id

> trace_id_from_ip4()

- 仅限于ipv4，在ip头部的`options`字段中添加traceID。
- 在ebpf内核态程序的入口函数中调用`check_and_store_ip_trace_id()`，将traceID存储到`cilium_percpu_trace_id`的percpu_map中。
- 用户态程序`cilium_percpu_trace_id`使用cell进行依赖注入管理。pkg/maps/iptrace/iptrace_map.go
- 主动notify的机制，ebpf内核态代码中send_trace_notify，上报关键事件，有些事件会携带traceID。`trace_reason`

## EBPF_PROG_TYPE_XDP

> bpf_xdp.c

仅分析ipv4

1. 设置traceID
2. 查cilium_cidr_v4_fix 和 cilium_cidr_v4_dyn这两个map，判断是否需要drop。cilium_cidr_v4_fix为精确匹配ip地址， cilium_cidr_v4_dyn为lpm匹配ip地址前缀。这两个map的value没有意义。
3. prefilter_v4完成之后，check_v4_lb。通过tail_call机制调用`tail_lb_ipv4`。进行负载均衡（转发）策略。
4. 结合k8s的网络资源抽象，进行流量转发。例如NodePort、LoadBalancer等。NodePort需要将流量转发到后端的Pod上，有可能这个Pod不在当前node上。
`
## NodePort 

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


ctx_{load,store}_meta():
- 宏的位置：`common.h::CB_SRC_LABEL`，enum，只有5个位置。
- 对于tc，使用了__sk_buff中的`cb`字段。[link](https://docs.ebpf.io/linux/program-context/__sk_buff/#cb)
  - tc程序中对__sk_buff结构体的修改会被直接映射到真正的sk_buff结构体中。__sk_buff仅仅是为了验证器检查程序是否安全，例如将一些字段设置成只读，防止ebpf程序修改这些字段。真正执行的时候操作的是sk_buff结构体。
- 对于xdp，使用了`cilium_xdp_scratch`这个map。