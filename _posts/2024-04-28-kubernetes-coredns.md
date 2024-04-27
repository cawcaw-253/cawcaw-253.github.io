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

> 기존에는 Kube-DNS가 이 역할을 했었습니다. 그러나 `1.12`버전부터 CoreDNS가 표준으로 채택되었고, 그 이후부터는 `kubeadm`으로 설치하는 경우 CoreDNS가 설치되고 있습니다.

이번에는 이 CoreDNS에 대해서 공부한 내용을 정리해 보도록 하겠습니다.

# CoreDNS 기본 구성

우선 CoreDNS는 Pod로 실행되기에 Service를 통해 요청을 받게됩니다. 그래서 `kubeadm`으로  클러스터를 생성한 뒤 오브젝트를 확인해보면 다음과 같이 Pod와 Service가 있는 것을 확인할 수 있습니다.

```
$ kubectl get po -n kube-system -l k8s-app=kube-dns

NAME                       READY   STATUS    RESTARTS   AGE
coredns-76f75df574-c6dwp   1/1     Running   0          58m
coredns-76f75df574-s8nfx   1/1     Running   0          58m

$ kubectl get svc -n kube-system

NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   60m
```

이러한 CoreDNS의 Service 주소는 Pod 생성시에 자동으로 `/etc/resolv.conf`에 설정되게 됩니다.

```
root@nginx:/# cat /etc/resolv.conf 

search default.svc.cluster.local svc.cluster.local cluster.local kornet
nameserver 10.96.0.10
options ndots:5
```

따라서 Pod는 내부 외부 도메인이냐에 따라서 아래와 같이 CoreDNS에 쿼리를 보내고 동작을 하게됩니다.

![internal-coredns](posts/20240428/coredns.png)

![external-coredns](posts/20240428/coredns-external.png)


# Corefile을 이용한 설정


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


# FDQN

쿠버네티스는 Lease API를 사용하여 kubelet 노드의 하트비트를 쿠버네티스 API Server에 전달합니다.
모든 노드에는 아래와 같이 같은 이름을 가진 Lease 오브젝트가 kube-node-lease namespace에 존재합니다.


# Reference
- [coresdns is still labeled as kube-dns](https://github.com/coredns/deployment/issues/116)
- https://jonnung.dev/kubernetes/2020/05/11/kubernetes-dns-about-coredns/
- https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- https://cprayer.github.io/posts/k8s-and-etc-resolv-conf/