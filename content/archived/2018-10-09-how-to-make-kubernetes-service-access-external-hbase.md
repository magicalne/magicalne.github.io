---
title: 如何让kubernetes中的服务访问外部hbase
layout: post
date: 2018-10-09
tag:
- kubernetes
- istio
- hbase
---

Istio通过修改iptables，以此来管理网络访问可行性。也就是说，你在部署你的服务之前，必须先定义好你需要访问的一切。是一切哦。

# Service Entry
这就是通过定义service entry来实现的。如果是访问内部服务，貌似只需要指定要访问的service name作为host。访问外部服务就有点麻烦了。

## HTTPS
我的服务中需要访问aws codecommit，这就有点烦了，不过官方文档还是能找到例子的。对于HTTPS的external service，不仅仅需要定义service entry，还要定义vritual service。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc-codecommit
spec:
  hosts:
  - git-codecommit.ap-southeast-1.amazonaws.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: external-svc-codecommit
spec:
  hosts:
  - git-codecommit.ap-southeast-1.amazonaws.com
  tls:
  - match:
    - port: 443
      sni_hosts:
      - git-codecommit.ap-southeast-1.amazonaws.com
    route:
    - destination:
        host: git-codecommit.ap-southeast-1.amazonaws.com
        port:
          number: 443
      weight: 100
```
从上面可以看出，对于https的请求，需要先访问到virtual service中，然后再转到外部的https请求。这个实现也是挺曲线救国的。

## 访问HBase集群
访问HBase有一点tricky，因为HBase是富客户端模式，client需要访问master，zookeeper和region servers。所以对应的，要找到它们在集群的端口号。

我之前遇到的问题实在蠢的不行，我自己的服务是用gRPC做网关，就像当然的以为HBase的rpc也是用的GRPC。我之前就知道HBase序列化用的protobuf，而且我之前确实看到代码里有grpc的字样。但其实HBase的rpc是自己实现的，只不过用protobuf填充header和payload。所以我在一开始的时候，设置region server的service entry的类型是grpc，而不是tcp，所以会一直报错。

正确的配置如下所示：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc-hbase
spec:
  hosts:
  - "*.ap-southeast-1.compute.internal"
  location: MESH_EXTERNAL
  ports:
  - number: 16020
    name: tcp-regionserver
    protocol: TCP
  - number: 16000
    name: HMaster
    protocol: TCP
  - number: 2181
    name: zookeeper
    protocol: TCP
  resolution: NONE
```

# 总结
现在回头看还是很简单的，但当时确实搞了好久，对istio的理解又深刻了一丢丢。有一点尤为重要，服务所需要访问的一切资源都需要提前定义。先定义好service entry，再部署服务。因为istio的sidecar proxy需要在部署中初始化。
