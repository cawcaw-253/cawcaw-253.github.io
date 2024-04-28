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

Corefile은 DNS 서버가 작동하고 들어오는 요청에 응답하는 방법을 지정하는 텍스트 파일입니다.

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

Corefile은 ConfigMap 오브젝트를 통해서 확인할 수 있으며, 여러 플러그인의 조합으로 구성되어 있습니다.

위의 내용을 보면 클러스터의 최상위 도메인 이름은 쿠버네티스 플러그인 구성에서 "cluster.local"로 지정되는 것을 확인 할 수 있습니다. 플러그인은 in-addr.arpa 및 ip6.arpa 도메인을 사용하여 IPv4 및 IPv6 주소에 대한 역방향 DNS 조회를 처리하도록 구성됩니다.

> 여기서 역방향 DNS 조회란 일반적으로 도메인 이름을 쿼리하여 IP 주소를 얻어내는 것이 아닌, 해당 IP 주소에 연결된 도메인 이름이 결과로 반환되는 DNS 쿼리입니다.
> 더 자세한 내용은 Reference의 `역방향 DNS`를 참고해주세요.

여기서 기본적으로 설정되어있는 플러그인에 대해서 설명하자면 다음과 같습니다.

- [errors](https://coredns.io/plugins/errors/) : 오류를 stdout에 기록합니다.
- [health](https://coredns.io/plugins/health/) : CoreDNS의 상태를 http://localhost:8080/health 를 통해 보고합니다. 이 확장 구문에서 lameduck은 프로세스가 종료되기 전에 unhealty 상태로 만들어 5초 동안 기다리게 해줍니다.
- [ready](https://coredns.io/plugins/ready/) : 준비 상태를 알릴 수 있는 모든 플러그인이 준비 상태를 알린 경우 포트 8181의 HTTP 엔드포인트를 통해 200 OK를 반환합니다.
- [kubernetes](https://coredns.io/plugins/kubernetes/) : kubernetes의 서비스 및 파드의 IP를 기반으로 DNS 쿼리에 응답하게 합니다. ttl등의 설정을 통해 자세한 설정을 할 수 있습니다.
  이 플러그인에 대한 자세한 내용은 [CoreDNS 웹사이트](https://coredns.io/plugins/kubernetes/)에서 확인할 수 있습니다.
- [cache](https://coredns.io/plugins/cache/) : 프론트엔드 캐시를 활성화합니다.  
- [loop](https://coredns.io/plugins/loop/) : 단순 포워딩 루프를 감지하고 루프가 발견되면 CoreDNS 프로세스를 중지합니다.  

Corefile과 구문에 대해 더 자세히 알아보려면 [CoreDNS 매뉴얼](https://coredns.io/manual/toc/) 또는 [CoreDNS ConfigMap 옵션](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/#coredns-configmap-options)에서 제공되는 공식 문서를 참조해 주세요.


# FDQN

쿠버네티스는 Lease API를 사용하여 kubelet 노드의 하트비트를 쿠버네티스 API Server에 전달합니다.
모든 노드에는 아래와 같이 같은 이름을 가진 Lease 오브젝트가 kube-node-lease namespace에 존재합니다.


# Reference
- [coresdns is still labeled as kube-dns](https://github.com/coredns/deployment/issues/116)
- https://jonnung.dev/kubernetes/2020/05/11/kubernetes-dns-about-coredns/
- https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- https://cprayer.github.io/posts/k8s-and-etc-resolv-conf/
- [역방향 DNS](https://powerdmarc.com/ko/what-are-reverse-dns-records/)


이미지 출처
https://h-susu.tistory.com/13