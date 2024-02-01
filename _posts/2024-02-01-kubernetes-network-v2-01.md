---
title: "[Kubernetes Network] 쿠버네티스 서비스"
author: cawcaw253
date: 2024-02-01 21:55:00 +0900
categories:
  - Kubernetes
  - Network
tags:
  - kubernetes
  - network
  - deep-dive
  - service
---
# 개요

지난번에는 [[Kubernetes Deep Dive] 쿠버네티스 네트워크](https://blog.cawcaw253.com/posts/kubernetes-pod-network/)를 통해서 쿠버네티스 내부에서 어떻게 패킷이 이동하고 서로 통신할 수 있는지를 간단하게 설명했었습니다. 
이번에는 우리가 쿠버네티스를 사용하면서 항상 사용하는 서비스에 대해서 조금 더 깊게 파고들어서 정리해보고자 합니다. 
단순히 서비스가 어떻게 동작하는가를 넘어서 CNI, kube-proxy, iptables 등의 내용과 함께 설명해 보도록 하겠습니다.

# Service의 동작 방식

각 Service Type별 동작 방식을 보기에 앞서 실제로 service를 생성했을 때 클러스터 내부에서 어떠한 동작이 일어나는지 알아보도록 하겠습니다.

우선 간략하게 어떻게 Service가 생성되는지에 대해서 설명하자면 다음과 같습니다.

> 1. Service Resource 생성 (해당 내용을 etcd에 기록)
> 2. kube-apiserver에서 kube-proxy에게 해당 변경사항을 전달
> 3. kube-proxy가 변경사항을 확인하고 변경된 내용을 네트워크 룰에 추가

이러한 방식으로 service에서 설정한 내용을 kube-proxy가 각 노드의 네트워크 룰에 업데이트 함으로써 실제 Service의 IP를 가지고 해당 pod로 패킷이 이동할 수 있는 것입니다.

여기서 계속 언급되는 kube-proxy라는 컴포넌트가 있는데 이 컴포넌트가 정확히 어떤 일을 하는지에 대해서도 설명하도록 하겠습니다.
### kube-proxy
![kube-proxy](posts/20240201/kube-proxy.png)
kube-proxy는 쿠버네티스의 네트워크 뒤편에서 Service를 사용 가능한 네트워크 룰로 변환해 주는 역할을 해주는 에이전트입니다. 

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
- 