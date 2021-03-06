---
assignees:
- bprashanth
- davidopp
- lavalamp
- liggitt
title: Managing Service Accounts
---

*本文是 service accounts 管理员手册。需要[Service Accounts 用户手册](/docs/user-guide/service-accounts)的知识背景。*

*对于 authorization 和 user accounts 支持已经在计划中，但是还未完成。为了更好地解释 service
accounts，有时会提到一些未完成的功能特性。*


## 用户账号 vs 服务账号


Kubernetes 将 user account 和 service account 的概念区分开，主要基于以下几个原因：

  - user account 是给人使用的。servcice account是给 pod 中运行的进程使用的。
  - user account 为全局设计。命名必须在一个集群的所有命名空间中唯一，未来的用户资源不会被设计到命名空间中。
    service account 是按命名空间的。
  - 典型场景中，一个集群的 user account 是来自于企业数据库的，在数据库中新建用户一般需要特殊权限，并且账号是与复杂都业务流程关联的。
    service account的创建往往更轻量，允许集群用户为特定的任务创建service account（即，权限最小化原则）。
  - 对于人和服务的账号，审计要求会是不同的。
  - 对于复杂系统而言，配置包可以包含该系统各类组件的 service account 定义，
    因为 service account 可以有临时的创建需求和自己的命名空间，这类配置是便携式的。


## 服务账号自动化


service accounts 的自动化由三个独立的组建共同配合实现：

  - A Service account admission controller (service account 准入控制器)
  - A Token controller (令牌控制器)
  - A Service account controller (service account 控制器)


### 服务账号准入控制器


对于 pods 的操作是通过一个叫做 [准入控制器](/docs/admin/admission-controllers) 的插件实现的。它是 apiserver 的一部分。
当准入控制器被创建或更新时，他会对 pod 同步进行操作。当这个插件时活动状态时（大部分版本默认是活动状态），并在 pod 被创建或者更改时，
它会做如下操作：

 1. 如果 pod 没有配置 `ServiceAccount`，它会将 `ServiceAccount` 设置为  `default`。
 2. 它会确保被 pod 关联的 `ServiceAccount` 是存在的，否则就拒绝请求。
 
 4. 如果 pod 没有包含任何的 `ImagePullSecrets`，那么 `ServiceAccount` 的 `ImagePullSecrets` 就会被添加到 pod。
 5. 它会把一个 `volume` 添加给 pod， 该 pod 包含有一个用于 API 访问的 token。
 6. 它会把一个 `volumeSource` 添加到 pod 的每个 container，并挂载到 `/var/run/secrets/kubernetes.io/serviceaccount`。


### 令牌控制器


令牌控制器（TokenController）作为 controller-manager 的一部分运行。它异步运行。它会：

- 监听对于 serviceAccount 的创建动作，并创建对应的 Secret 以允许 API 访问。
- 监听对于 serviceAccount 的删除动作，并删除所有对应的 ServiceAccountToken Secrets。
- 监听对于 secret 的添加动作，确保相关联的 ServiceAccount 是存在的，并在根据需要为 secret 添加一个 token。
- 监听对于 secret 的删除动作，并根据需要删除对应 ServiceAccount 的关联。

你必须给令牌控制器（token controller）传递一个 service account 的私钥（private key），可以通过 `--service-account-private-key-file` 参数完成。
传递的私钥将被用来对 service account tokens 进行签名。类似的，你必须给 kube-apiserver 传递一个公钥（public key），通过 `--service-account-key-file`
参数完成。传递的公钥会被用来验证认证过程中的令牌（token）。


#### 创建额外的 API 令牌


控制器的循环运行会确保对于每个 service account 都存在一个带有 API token（API 令牌）的secret。
如需要为一个 service account 创建一个额外的 API 令牌（API token），可以创建一个 `ServiceAccountToken`
类型的 secret，并添加与 service account 对应的 annotation 属性，控制器会为它更新 token：

secret.json:

```json
{
    "kind": "Secret",
    "apiVersion": "v1",
    "metadata": {
        "name": "mysecretname",
        "annotations": {
            "kubernetes.io/service-account.name": "myserviceaccount"
        }
    },
    "type": "kubernetes.io/service-account-token"
}
```

```shell
kubectl create -f ./secret.json
kubectl describe secret mysecretname
```


#### 删除／作废服务账号令牌

```shell
kubectl delete secret mysecretname
```


### 服务账号控制器


Service Account Controller 在 namespaces 内管理 ServiceAccount，需要保证名为 "default" 的
ServiceAccount在每个命名空间中存在。