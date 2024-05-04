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

# CoreDNS 상세

## Corefile을 이용한 설정

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

## DNS Resolve

일반적으로 호스트에서 특정 도메인으로 요청을 보내면 DNS로 IP주소를 가져와서 해당 주소로 요청을 보내게 됩니다.

마찬가지로 Kubernetes의 Pod에서도 요청을 보내면 CoreDNS를 대상으로 Query를 보내고 주소를 가져와서 요청을 처리하게 됩니다.

따라서 이해한 바에 따르면 `amazon.com`에 요청을 보내면 바로 주소를 가져와서 요청을 처리해야할 것입니다. 그런데 Pod에서 `curl amazon.com`을 실행한 뒤 CoreDNS의 로그를 보면 다음과 같은 로그를 보실 수 있습니다.

```bash
[INFO] 10.29.101.97:49777 - 15626 "AAAA IN amazon.com.default.svc.cluster.local. udp 54 false 512" NXDOMAIN qr,aa,rd 147 0.000245472s
[INFO] 10.29.101.97:49777 - 43276 "A IN amazon.com.default.svc.cluster.local. udp 54 false 512" NXDOMAIN qr,aa,rd 147 0.000376412s
[INFO] 10.29.101.97:45709 - 13560 "AAAA IN amazon.com.svc.cluster.local. udp 46 false 512" NXDOMAIN qr,aa,rd 139 0.000214814s
[INFO] 10.29.101.97:45709 - 14586 "A IN amazon.com.svc.cluster.local. udp 46 false 512" NXDOMAIN qr,aa,rd 139 0.000286298s
[INFO] 10.29.101.97:59010 - 20726 "A IN amazon.com.cluster.local. udp 42 false 512" NXDOMAIN qr,aa,rd 135 0.000150982s
[INFO] 10.29.101.97:59010 - 7664 "AAAA IN amazon.com.cluster.local. udp 42 false 512" NXDOMAIN qr,aa,rd 135 0.00031106s
[INFO] 10.29.101.97:51305 - 54019 "AAAA IN amazon.com.ap-northeast-2.compute.internal. udp 60 false 512" NXDOMAIN qr,rd,ra 183 0.001475703s
[INFO] 10.29.101.97:51305 - 50693 "A IN amazon.com.ap-northeast-2.compute.internal. udp 60 false 512" NXDOMAIN qr,rd,ra 183 0.001544829s
[INFO] 10.29.101.97:55579 - 58497 "A IN amazon.com. udp 28 false 512" NOERROR qr,rd,ra 106 0.001595382s
[INFO] 10.29.101.97:55579 - 31875 "AAAA IN amazon.com. udp 28 false 512" NOERROR qr,rd,ra 125 0.00166369s
```

로그를 보면 `amazon.com` 뿐만 아니라 `amazon.com.default.svc.cluster.local.`, `amazon.com.cluster.local` 등의 여러 도메인을 쿼리하는 것을 볼 수 있습니다.

이렇게 동작하는 이유는 각 Pod에 설정된 `resolv.conf`를 보면 알 수 있습니다.

```bash
$ kubectl exec -it <POD_NAME> -- cat /etc/resolv.conf

search default.svc.cluster.local svc.cluster.local cluster.local ap-northeast-2.compute.internal
nameserver 172.20.0.10
options ndots:5
```

resolv.conf에 대한 linux 매뉴얼[linux manual - resolv.conf]의 내용을 참고하면 쿼리에 `.` 이 `ndots` 보다 적게 포함되어있을 경우(FQDN이 아닐경우), search에 있는 서치 도메인을 순서대로 붙여 매치될 때까지 진행하게 된다고 나와있습니다.

여기서 `ndots`는 쿼리 이름에 있는 점의 수를 FQDN(Fully Qualified Domain Name)으로 간주하는 임계값을 나타냅니다.

```
# Search list for host-name lookup.

By default, the search list contains one entry, the local domain name.  It is determined from the local hostname returned by gethostname(2); the local domain name is taken to be everything after the first '.'.  Finally, if the hostname does not contain a '.', the root domain is assumed as the local domain name.

...

Resolver queries having fewer than ndots dots (default is 1) in them will be attempted using each component of the search path in turn until a match is found.
```

따라서 `amazon.com` 이라는 도메인의 `.`의 수가 5보다 작으므로 FQDN이라고 판단되지 않아 `search` 리스트의 모든 검색을 진행하므로 위와 같은 로그가 찍힌 것입니다.

### Query 성능 개선

위의 내용을 보면 Pod 내에서 우리가 일반적으로 사용하는 도메인 이름을 사용한다면 하나의 도메인에 접근할 때 3~4배의 쿼리가 발생한다는 것을 알 수 있습니다.

이는 트래픽이 많아진다면 CoreDNS에 많은 부담이 될 수 있습니다. 그렇다면 이를 어떻게 해결할 수 있을까요?

답은 간단합니다. 바로 FQDN 도메인으로 질의를 하도록 Application 단에서 FQDN 도메인을 사용하도록 하면 됩니다.

즉 `curl amazon.com` 이 아닌 `curl amazon.com.` 으로 제일 뒤에 `.`을 붙여 FQDN으로 명시적으로 만들어주면 다음과 같이 한번에 Query가 성공하는 것을 볼 수 있습니다.

```bash
[INFO] 10.85.0.1:59724 - 59684 "A IN amazon.com. udp 28 false 512" NOERROR qr,rd,ra 106 0.002072165s
[INFO] 10.85.0.1:59724 - 9510 "AAAA IN amazon.com. udp 28 false 512" NOERROR qr,rd,ra 125 0.004880986s
```

### 왜 이러한 설정이 되어있을까?

이렇게 설정되어 있는 이유는 편의성 때문이며 같은 네임스페이스, 혹은 다른 네임스페이스에 있는 서비스를 도메인으로 접근할 때 FQDN이 아닌 짧고 간결한 도메인 명으로 접근할 수 있게 하기 위합입니다. [Namespaces of Services]

이러한 설정 덕분에 우리는 매번 `<SERVICE_NAME>.default.svc.cluster.local` 와 같이 길게 쓸 필요없이 service 이름만으로 연결이 될 수 있는 것입니다.

# 마치며



# Reference
- [coresdns is still labeled as kube-dns](https://github.com/coredns/deployment/issues/116)
- https://jonnung.dev/kubernetes/2020/05/11/kubernetes-dns-about-coredns/
- https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- https://cprayer.github.io/posts/k8s-and-etc-resolv-conf/
- [역방향 DNS](https://powerdmarc.com/ko/what-are-reverse-dns-records/)
- [linux manual - resolv.conf](https://www.man7.org/linux/man-pages/man5/resolv.conf.5.html)
- [Namespaces of Services](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#namespaces-of-services)


이미지 출처
https://h-susu.tistory.com/13