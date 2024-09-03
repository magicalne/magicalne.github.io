---
title: 如何在kubernetes上构建grpc服务以及负载均衡
layout: post
date: 2018-10-09
tag:
- kubernetes
- istio
- grpc
---

最近一直在搞kubernetes上的服务化。首先切到kubernetes上的项目是一个没有状态的data worker。一切如丝般润滑，简单到爆。但是为了部署我的grpc service时，引入了istio，一切变得艰难了好多。

首先，如果不考虑istio，单看kubernetes的话，这里需要定义一个**Deployment**，方便进行rolling update。然后，grpc service是一个service，这里还需要定义一个**Service**。此时，kubernetes的工作就结束了。到这里为止，k8s cluster上有一个grpc service instance了。我们理应可以通过ip访问这个服务。但是如果我们希望有多个instance，如何做load balance？

这里就要引入istio了。istio管理了kubernetes上所有的网络通信。所有跟网络相关的配置，都需要走istio。

# 如何构建全局的load balancer
我希望可以通过一个域名就能访问到k8s集群上所有的服务。使用kops在aws上创建的集群已经有一个aws classic load balancer，这里就可以直接使用了。这个load balancer不做任何事，只是把请求转发给istio的ingress gateway。注意，这里的loadbalancer是用来访问需要暴露到outside的服务。

## Gateway & Virtual Service
Istio使用ingress gateway来管理所有流入的流量。我们需要创建不同功能的gateway用来从ingress gateway中分流到自己所需的流量。但是gateway不能单独使用，这里还要配合virtual service。virtual service & gateway共同定义了流量特征。

比如，对应http服务，我希望获取uri开头为/hello的请求，并将这些流量转给端口号为8080的hello-world service。这里就需要定义virtual service和gateway来做这件事。

在这里，我看文档的时候有点懵。因为我发现virtual service和gateway有一些相同的配置，我不明白这两个为什么一定要拆开定义，这里的设计没太看懂。

另外，istio是以sidecar的形式，在初始化服务的时候，将转发的配置信息注入进来的。所以，你会看到一个服务会有两个pod：服务本身和sidecar。

最后，kubernetes和istio对gRPC都十分友好。gRPC使用的是HTTP2，这里在定义virtual service的时候也可以通过uri prefix的方式分割不同的服务流量。

配置好的yaml看起来就像下面这样：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: main-server-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 8888
      name: grpc
      protocol: GRPC #or GRPC, which gives the same result
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: main-server
spec:
  hosts:
  - "*"
  gateways:
  - main-server-gateway
  http:
  - match:
    - uri:
        prefix: "/main.MainService"
    route:
    - destination:
        port:
          number: 8888
        host: main-server
---
apiVersion: v1
kind: Service
metadata:
  name: main-server
  labels:
    app: main-server
spec:
  ports:
  - name: grpc
    port: 8888
  selector:
    app: main-server
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: main-server
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: main-server
        version: v1
    spec:
      containers:
      - image: xxxxxxx.dkr.ecr.ap-southeast-1.amazonaws.com/grpc-test
        imagePullPolicy: Always
        name: main-server
        env:
        - name: EXTERNAL_SERVER_HOST
          value: "ip-xxxxxxxxxx.ap-southeast-1.compute.internal"
        - name: EXTERNAL_SERVER_PORT
          value: "8888"
        ports:
        - containerPort: 8888
```

只要几段配置就能做grpc服务的负载均衡，确实挺方便。科技是我幸福。
