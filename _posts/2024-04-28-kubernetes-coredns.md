---
title: "[Kubernetes Deep Dive] Core DNS"
author: cawcaw253
date: 2023-04-28 17:32:00 +0900
categories:
  - Kubernetes
tags:
  - kubernetes
  - coredns
published: false
---

---
# 개요

Kubernetes에서는 클러스터 내부의 Pod에서 도메인을 찾고자 할 때 외부 DNS 서버가 아닌 내부 DNS인 CoreDNS에 질의를 하게됩니다.

```
root@nginx:/# cat /etc/resolv.conf 

search default.svc.cluster.local svc.cluster.local cluster.local kornet
nameserver 10.96.0.10
options ndots:5
```



트러블슈팅 하면서 Node Health Check에서 Lease가 있다는 것은 알았지만 자세히는 몰랐는데, 이번 기회에 공부하게 되었습니다. 

이번 글에서는 Lease가 어떤 기능을 하는지 그리고 어떻게 삭제, 생성되는지에 대해서 정리해 보도록 하겠습니다.

# Lease의 기능


```bash
$ kubectl describe cm -n kube-system coredns

Name:         coredns
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Data
====
Corefile:
----
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf {
       max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}
```

## Node Heartbeat

쿠버네티스는 Lease API를 사용하여 kubelet 노드의 하트비트를 쿠버네티스 API Server에 전달합니다.
모든 노드에는 아래와 같이 같은 이름을 가진 Lease 오브젝트가 kube-node-lease namespace에 존재합니다.