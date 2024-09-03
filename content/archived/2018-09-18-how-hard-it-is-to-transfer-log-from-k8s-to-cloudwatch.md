---
title: 把kubernetes的log转发到cloudwatch有多难？
layout: post
date: 2018-09-19
tag:
- cloudwatch
- kubernetes
---

# kubernetes日志收集

## docer level

最近尝试把kubernetes的log转发到aws的cloudwatch中。原因当然是因为方便查log咯。之前试过用docker的方式导出，还是比较方便的。原理就是在使用kops创建集群的时候，添加两个配置项：
1. aws cloudwatch权限
2. docker配置里，设置使用awslog作为log收集器

这样就可以简单的将kubernetes中所有container的log收集到cloudwatch下的group名字为：kubernetes的下面。唯一的问题是，log name使用的是container id，不够对人类友好。当时的想法是对kubernetes的dashboard进行二次开发。因为使用了docker方式的日志收集之后，dashboard下的log功能已经废弃了，所以我想在dashboard中点击log的时候，通过获取container id的方式，直接拿到cloudwatch下的log。听上去还是很美好的，也是可行的，但是并没有动手做。

## fluentd

后来折腾grpc on istio的时候，因为看log实在是太麻烦了，本来问题就很tricky，看log又麻烦。索性就用fluentd重新搞了一遍log收集。

之前用fluentd cloudwatch搞过一次，因为失败了就没再搞。这次算是抱着必死的决心，一定要搞定！

### Helm

这里使用[helm](https://docs.helm.sh/using_helm/#installing-helm)安装。首先吐槽helm，helm的文档基本是摆设，实际功能全靠自己摸索。虽然说kubernetes很复杂，但是人家谷歌文档确实写得好啊。helm的文档从排版到内容，啧啧...

我这里使用kops安装kubernetes的，是一台内存1g的linux，使用snap安装helm。helm在init的时候需要为tiller创建service account和role:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```

注意这里指定了**namespace: kube-system**。这是kubernetes所谓role based access controll(RBAC)的重要一步。少了这一步，后面会有各种各样的麻烦，helm文档居然没有强制要求，最佳实践也是一笔带过。

### kube2iam

接下来需要装[kube2iam](https://github.com/helm/charts/tree/master/stable/kube2iam)。这里就需要之前安装的tiller了：
```sh
helm install stable/kube2iam --namespace kube-system --name my-release   --set=extraArgs.base-role-arn=arn:aws:iam::xxxxxxxxxx:role/,extraArgs.default-role=kube2iam-default,host.iptables=true,host.interface=cbr0,rbac.create=true
```

关于role的解释在[这里](https://gist.github.com/snoby/77a49b6b79d0dd2ad9afbbf533588f54)。对应kops下创建的集群，kops会在aws下创建多个role，对应node的role的名字应该是nodes.xxx.xxx.local。这里的问题是，如果我现在需要对这个role新增policy，就需要每次修改node role。kube2iam就是来解决这个问题的。kube2iam使用了aws iam中assume的功能，这里的assume可以理解为**承担**。你需要创建一个新role，这里就是kube2iam-default，在这个role中修改trust relation：
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    },
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::xxxxx:role/nodes.xxxxx.k8s.local"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
修改kops的配置。输入命令：
```sh
kops edit cluster
```
然后在spec下新添加配置：
```yaml
  additionalPolicies:
    nodes: |
      [
        {
          "Effect": "Allow",
          "Action": [
            "sts:AssumeRole"
          ],
          "Resource": [
            "arn:aws:iam::xxxxxxxxx:role/kube2iam-default"
          ]
        }
      ]
```
也可以将**kube2iam-default**替换为**kube2iam-* **，意思就是：让所有以kube2iam开头的role承担对应功能，你可以为s3功能/资源添加新的kube2iam-s3，或是为了cloudwatch添加新的kube2iam-cloudwatch，等等。

### fluentd-cloudwatch

下面就该是安装[fluentd-cloudwatch](https://github.com/helm/charts/tree/master/incubator/fluentd-cloudwatch)。这里心疼自己一秒钟，摸摸头。首先还是RBAC的问题，这里用**rbac.create**是不行的。至今为止，我遇到的，helm上所有的默认配置都是不能直接工作的！

首先，fluentd-cloudwatch还是incubator项目，所以不能直接在helm上install，需要先add incubator。
然后，配置role，需要新创建role和service account。
```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: fluentd-service-account
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: fluentd-service-account
subjects:
- kind: ServiceAccount
  name: fluentd-service-account
  namespace: kube-system

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: fluentd-service-account
  namespace: kube-system
rules:
  - apiGroups: ["*"]
    resources:
      - pods
      - namespaces
    verbs:
      - get
      - watch
      - list

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd-service-account
  namespace: kube-system
```
如果这里没配置正确，或者没有让fluentd-cloudwatch拿到这个service account，就会报cannot access resources的异常。所以需要指定**rbac.serviceAccountName=fluentd-service-account**。
我之前遇到了一个诡异的问题，就是无法rbac.serviceAccountName赋值，在helm install之后，用这个命令查看：
```sh
kubectl --namespace=kube-system get pod fluentd-cloudwatch-xxxxx -o yaml | grep serviceAccount
```
永远都是default...囧
我当时一直用的**helm install -f fluentd-cloudwatch.yaml**的方式安装，但是后来不指定yaml，用命令行传参数后，就可以了-。- 好迷~

但是用命令行还是不行，为什么呢？文档说了：

> Starting with fluentd-kubernetes-daemonset v0.12.43-cloudwatch, the container runs as user fluentd. To be able to write pos files to the host system, you'll need to run fluentd as root. Add the following extraVars value to run as root.

>> "{ name: FLUENT_UID, value: '0' }"

我一开始是这样的：
```sh
helm install --name fluentd-cloudwatch --namespace kube-system --set awsRegion=ap-southeast-1,awsRole=kube2iam-default,rbac.serviceAccountName=fluentd-service-account,extraVars[0]="{ name: FLUENT_UID, value: '0' }" incubator/fluentd-cloudwatch
```
结果不行，报错，说什么yaml不能写json，看了看template的源码，确实写的有问题，难不成又要自己改？google了一下，发现很多人发现这个问题，helm里也有人提pull request了，可是都一个月了都没merge进去...然后我尝试用values.yaml传这个参数，结果成功了。
```sh
helm install incubator/fluentd-cloudwatch --name fluentd-cloudwatch --namespace kube-system --set awsRegion=ap-southeast-1,awsRole=kube2iam-default,rbac.serviceAccountName=fluentd-service-account -f values.yaml
```
这里的awsRole就是之前我在kube2iam创建的那个role。

# Long day...

一切就绪，可以在cloudwatch下的kubernetes group下看到流进来的log了，包含了pod、namespace和container id。够用了。
现在看看感觉也没啥难的，但是每一个坑都要亲脚踩！回头想想，主要是对kubernetes的RBAC的理解不够深刻。剩下的就是开源软件不够健壮的问题了，见怪不怪。
