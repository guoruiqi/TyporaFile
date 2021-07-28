### 1.1 

#### 1.参数验证

敲下 `kubectl` 命令后，它首先会做一些客户端侧的验证。如果命令行参数有问题，例如，**镜像名为空或格式不对**[7]， 这里会直接报错，从而避免了将明显错误的请求发给 kube-apiserver，减轻了后者的压力。

#### 2.创建 HTTP 请求

创建 HTTP 请求会封装了资源的序列化（serialization）操作，生成一个该资源的运行时对象（runtime object）

### 1.2  API group 和版本协商

有了 runtime object 之后，kubectl 需要用合适的 API 将请求发送给 kube-apiserver。**搜索合适的 API group 和版本**，**创建一个正确版本的客户端**。

发送HTTP 请求

### 1.3 客户端认证（client auth）

**用户凭证（credentials）一般都放在 kubeconfig 文件中，但这个文件可以位于多个位置**，

客户端就可以组装 HTTP 请求的认证头了。



## **2 kube-apiserver**

### 2.1 认证（Authentication）

kube-apiserver 首先会对请求进行认证（authentication），以确保用户身份是合法的

**如果认证成功，就会将 `Authorization` 头从请求中删除**，然后在上下文中**加上用户信息**

### 2.2 鉴权（Authorization）

**发送者身份（认证）是一个问题，但他是否有权限执行这个操作（鉴权），是另一个问题**。

## **3 写入 etcd**

将资源最终**写到 etcd**

最后，storage provider 执行一次 `get` 操作，确保对象真的创建成功了

## **5 Control loops（控制循环）**

K8s 中大量使用 "controllers"，

- 一个 controller 就是一个异步脚本（an asynchronous script），
- 不断检查资源的**当前状态**（current state）和**期望状态**（desired state）是否一致，
- 如果不一致就尝试将其变成期望状态，这个过程称为 **reconcile**。

每个 controller 负责的东西都比较少，**所有 controller 并行运行， 由 kube-controller-manager 统一管理**。

### 5.1 Deployments controller

controller 会通过 label selector 从 kube-apiserver 查询 与这个 deployment 关联的 ReplicaSet 或 Pod records（然后发现没有）。

如果发现当前状态与预期状态不一致，就会触发同步过程（（synchronization process））。

#### 执行扩容（scale up）

如上，发现 pod 不存在之后，它会开始扩容过程（scaling process）：

**Deployment controller 只负责 ReplicaSet 的创建**，因此下一步 （ReplicaSet -> Pod）要由 reconciliation 过程中的另一个 controller —— ReplicaSet controller 来完成。

### 5.2 ReplicaSets controller

上一步周，Deployments controller 已经创建了 Deployment 的第一个 ReplicaSet，但此时还没有任何 Pod。下面就轮到 ReplicaSet controller 出场了。它的任务是监控 ReplicaSet 及其依赖资源（pods）的生命周期，实现方式也是注册事件回调函数。

### 5.3 Informers

很多 controller（例如 RBAC authorizer 或 Deployment controller）需要将集群信息拉到本地。

例如 RBAC authorizer 中，authenticator 会将用户信息保存到请求上下文中。随后， RBAC authorizer 会用这个信息获取 etcd 中所有与这个用户相关的 role 和 role bindings。

那么，controller 是如何访问和修改这些资源的？在 K8s 中，这是通过 informer 机制实现的。

### 5.4 Scheduler（调度器）

以上 controllers 执行完各自的处理之后，etcd 中已经有了一个 Deployment、一个 ReplicaSet 和三个 Pods，可以通过 kube-apiserver 查询到。但此时，**这三个 pod 还卡在 Pending 状态，因为它们还没有被调度到任何节点**。**另外一个 controller —— 调度器** —— 负责做这件事情。



#### 调度算法

预选：

**如果 PodSpec 里面显式要求了 CPU 或 RAM 资源，而一个 node 无法满足这些条件**， 那就会将这个 node 从备选列表中删除。

优选：

**如果 PodSpec 里面显式要求了 CPU 或 RAM 资源，而一个 node 无法满足这些条件**， 那就会将这个 node 从备选列表中删除。





## 6 kubelet

每个 K8s node 上都会运行一个名为 kubelet 的 agent，它负责

- pod 生命周期管理。

  这意味着，它负责将 “Pod” 的逻辑抽象（etcd 中的元数据）转换成具体的容器（container）。

- 挂载目录

- 创建容器日志

- 垃圾回收等等

### 6.1 Pod sync（状态同步）

**kubelet 也可以认为是一个 controller**，它

1. 通过 ListWatch 接口，从 kube-apiserver **获取属于本节点的 Pod 列表**（根据`spec.nodeName` **过滤**[46]），
2. 然后与自己缓存的 pod 列表对比，如果有 pod 创建、删除、更新等操作，就开始同步状态。

### 6.2 CRI 及创建 pause 容器

至此，大部分准备工作都已完成，接下来即将创建容器了。**创建容器是通过 Container Runtime （例如 `docker` 或 `rkt`）完成的**。

为实现可扩展，kubelet 从 v1.5.0 开始，**使用 CRI（Container Runtime Interface）与具体的容器运行时交互**。简单来说，CRI 提供了 kubelet 和具体 runtime implementation 之间的抽象接口， 用 **protocol buffers**[54] 和 gRPC 通信。



#### CRI create sandbox

kubelet **发起 RunPodSandbox**[55] RPC 调用。

**“sandbox” 是一个 CRI 术语，它表示一组容器，在 K8s 里就是一个 Pod**。

在这种 runtime 中，**创建一个 sandbox 会转换成创建一个 “pause” 容器的操作**。



### 6.3 CNI 前半部分：CNI plugin manager 处理

现在我们的 pod 已经有了一个占坑用的 pause 容器，它占住了 pod 需要用到的所有 namespace。接下来需要做的就是：**调用底层的具体网络方案**（bridge/flannel/calico/cilium 等等） 提供的 CNI 插件，**创建并打通容器的网络**。

### 6.4 CNI 后半部分：CNI plugin 实现

下面看 CNI 处理的后半部分：CNI 插件为容器创建网络，也就是可执行文件 `/opt/cni/bin/xxx` 的实现。

CNI 相关的代码维护在一个单独的项目 **github.com/containernetworking/cni**[58]。每个 CNI 插件只需要实现其中的几个方法，然后**编译成独立的可执行文件**，放在 `/etc/cni/bin` 下面即可。下面是一些具体的插件，

### 6.6 创建 `init` 容器及业务容器

至此，网络部分都配置好了。接下来就开始启动真正的业务容器。

