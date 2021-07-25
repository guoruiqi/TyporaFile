**1、说说pod的创建过程**



**2、假如有一台机器k8s-node-1要升级内存，简要写出方案**

```shell
#设置k8s-node-1不可调度
kubectl cordon k8s-node-1
#在业务低峰期驱逐已经执行的业务容器
kubectl drain --ignore-daemonsets --delete-local-data k8s-node-1
#停docker、kubelet服务
systemctl stop kubelet
systemctl stop docker
```

**3、某个pod启动的时候需要用到pod的名称，请问怎么获取pod的名称，简要写出对应的yaml配置**

```yaml
env:
- name: test  
  valueFrom:    
    fieldRef:      
      fieldPath: metadata.name
```

**4、假如某个pod有多个副本，如何让两个pod分布在不同的node节点上，请简要写出对应的yaml文件**

```yaml
affinity:  
  podAntiAffinity:    
    requiredDuringSchedulingIgnoredDuringExecution:    
    - topologyKey: "kubernetes.io/hostname"      
      labelSelector:        
        matchLabels:          
          app: test
```

**5、请用系统镜像为centos:latest制作一个jdk版本为1.8.142的基础镜像，请写出dockerfile**

```dockerfile
FROM centos:latest
ENV JDK_HOME=/root/jdk1.8.0_142
WORKDIR /root
ADD jdk-8u142-linux-x64.tar.gz /root
ENV JAVA_HOME=/root/jdk1.8.0_142
ENV PATH=$PATH:$JAVA_HOME/bin
CMD ["bash"]
```

**6、某个pod需要配置某个内网服务的hosts，比如数据库的host，形如：192.168.4.124 db.test.com，请问有什么方法可以解决，简要写出对应的yaml配置或其他解决办法**

```yaml
解决办法：搭建内部的dns，在coredns配置中配置内网dns的IP要是内部没有dns的话，在yaml文件中配置hostAliases，形如：
hostAliases:
- ip: "192.168.4.124"  
  hostnames:  
  - "db.test.com"
```

**7、harbor镜像仓库的过期镜像如何清理**

**8、集群怎么升级，证书过期怎么解决，简要说出做法**

**9、prometheus-operator如何自定义监控，比如监控etcd集群**

**10、k8s日志如何收集，简要说出做法**

