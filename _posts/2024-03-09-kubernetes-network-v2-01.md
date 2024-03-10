---
title: "[Kubernetes Deep Dive] Kube-Proxy"
author: cawcaw253
date: 2024-03-09 21:55:00 +0900
categories:
  - Kubernetes
  - Network
tags:
  - kubernetes
  - network
  - deep-dive
  - service
---

개요
service 동작 방식
kube-proxy에 대해서
how kube-proxy work
정리

# 개요

지난번에는 [[Kubernetes Deep Dive] 쿠버네티스 네트워크](https://blog.cawcaw253.com/posts/kubernetes-pod-network/)를 통해서 쿠버네티스 클러스터 내부에서 어떻게 패킷이 이동하고 서로 통신할 수 있는지를 간단하게 설명했었습니다.
이번에는 우리가 쿠버네티스를 사용하면서 항상 보고있지만 정확히는 모르겠는 kube-proxy라는 컴포넌트가 무엇인지, 어떻게 동작하는지에 대해서 정리해보고자 합니다.

# Service

kube-proxy에 설명하기에 앞서서 Service는 무엇인지에 대해서 설명해보도록 하겠습니다.

우리는 쿠버네티스에서 Pod란 언제든지 종료될 수 있고 재생성 될 수 있기에 일시적인 존재라는 것을 알고있습니다. 새로운 Pod가 생성되면 이전 Pod의 IP와는 별개의 IP를 할당받게 되며 이로 인해서 우리는 Pod의 IP에 의존할 수 없었습니다.

이러한 문제를 해결하고자 Service가 등장하였습니다. Service는 여러 Pod에게 Stable한 IP와 DNS 이름을 제공하는 역할을 하며, 이로 인해 Pod는 종료되거나 재생성 되어도 어플리케이션이 잘 동작할 수 있었던 것입니다.

또한 Service는 Pod 간의 트래픽을 분산하여 요청이 균등하게 분산되고 개별 Pod가 과부하되지 않도록 보장해줍니다. 우리는 이를 통해서 고가용성 및 확장성 애플리케이션을 만들 수 있게 되었습니다.

그런데 Service와 Pod의 관계는 어떻게 유지될 수 있는걸까요?
바로 여기서 kube-proxy가 등장하게 됩니다. 

kube-proxy는 Service IP 주소를 해당 Service에 속한 Pod의 IP 주소에 매핑하는 네트워크 라우팅 테이블을 유지함으로써 Service-Pod 간 매핑을 지원해줍니다. 새로운 Service에 대한 요청이 오면, kube-proxy는 이 정보를 매핑하여 패킷이 Service에 속한 Pod로 전달될 수 있도록 해줍니다.

# Kube-Proxy

![kubernetes-node](posts/20240309/kubernetes-node.png)

위에서 설명했듯 kube-proxy는 뒤편에서 Service나 Endpoint(Pod IP)에 변경이 발생하면 변경 사항을 노드안에서 실제 사용 가능한 네트워크 룰로 변환해 주는 역할을 해주는 에이전트입니다.

이 에이전트는 클러스터내의 모든 노드에 설치되며 주로 <ins>**DaemonSet**</ins>으로 동작합니다. 물론 이 방법뿐만 아니라 노드에 프로세스로 직접 설치할 수 도 있습니다.

# Kube-Proxy는 어떻게 동작하는가

Kube-Proxy가 설치되면 API Server와 인증을 설정합니다.

kube-proxy는 쿠버네티스에 새로운 Service나 Endpoint(Pod IP)가 생성 및 삭제 되었을때, 이러한 변경 사항을 API Server로부터 전달받아 네트워크 룰로 변환하여 Service-Pod 간의 매핑을 해줍니다.

동작 방식을 조금 더 풀어서 설명해보자면 다음과 같습니다.



이 컴포넌트는 각 노드에서 동작하며 일반적으로 DaemonSet 형태로 실행됩니다. 
kube-proxy는 Service object와 그 endpoint의 변화를 모니터링하고 있으며, 변경이 발생했을 경우 노드 내부의 실제 네트워크 룰로 변환시킵니다. 

이 노드 내부의 네트워크 룰이란 NAT를 의미하는데 kube-proxy는 변경사항을 내부 iptables의 NAT 규칙으로 적용시켜 단순히 Service IP를 Pod IP에 매핑시키는 것입니다. 

따라서 특정 Service IP에 대한 패킷은 Linux의 netfilter기능을 통해서 iptables의 규칙을 기반으로 Pod IP로 리디렉션 되는 것입니다. 


그러면 간단하게 Service가 어떻게 동작하는지 알아보았으니 Service의 대표적인 세 타입 Cluster IP, NodePort, LoadBalancer에 대해서 그림과 예시를 통해 자세하게 설명해 보도록 하겠습니다.

# Cluster IP

> 서비스를 클러스터-내부 IP에 노출시킨다. 이 값을 선택하면 클러스터 내에서만 서비스에 도달할 수 있다. 이것은 서비스의 `type`을 명시적으로 지정하지 않았을 때의 기본값이다.
> 
> ref : https://kubernetes.io/ko/docs/concepts/services-networking/service/#publishing-services-service-types

Cluster IP 방식은 가장 기본적인 방식으로 클러스터 내부에서 해당 service 
## Single Node


## Multi Node



# Reference
- [[Kubernetes Deep Dive] 쿠버네티스 네트워크](https://blog.cawcaw253.com/posts/kubernetes-pod-network/)
- [kube-proxy](https://kodekloud.com/blog/kube-proxy/)



https://medium.com/@seifeddinerajhi/kube-proxy-and-cni-the-hidden-components-of-kubernetes-networking-eb30000bf87a
https://kube.academy/courses/kubernetes-in-depth/lessons/an-introduction-to-cni
https://kodekloud.com/blog/kube-proxy/#how-kube-proxy-works