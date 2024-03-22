---
title: "[Kubernetes Deep Dive] csi-driver"
author: cawcaw253
date: 2024-03-22 17:55:00 +0900
categories:
  - Kubernetes
  - Storage
  - Volume
tags:
  - kubernetes
  - deep-dive
  - storage
  - volume
---
---
# 개요

지난번에는 [[Kubernetes Deep Dive] 쿠버네티스 네트워크](https://blog.cawcaw253.com/posts/kubernetes-pod-network/)를 통해서 쿠버네티스 클러스터 내부에서 어떻게 패킷이 이동하고 서로 통신할 수 있는지를 간단하게 설명했었습니다.
이번에는 우리가 쿠버네티스를 사용하면서 항상 보고있지만 정확히는 모르겠는 kube-proxy라는 컴포넌트가 무엇인지, 어떻게 동작하는지에 대해서 정리해보고자 합니다.

# Service



# Reference
- [[Kubernetes Deep Dive] 쿠버네티스 네트워크](https://blog.cawcaw253.com/posts/kubernetes-pod-network/)
- [kube-proxy](https://kodekloud.com/blog/kube-proxy/)
- [iptables-or-ipvs](https://www.tigera.io/blog/comparing-kube-proxy-modes-iptables-or-ipvs/)
- [introduction-cni](https://kube.academy/courses/kubernetes-in-depth/lessons/an-introduction-to-cni)
- [Kube-Proxy and CNI: The Hidden Components of Kubernetes Networking](https://medium.com/@seifeddinerajhi/kube-proxy-and-cni-the-hidden-components-of-kubernetes-networking-eb30000bf87a)
- [Endpoint란](https://yoo11052.tistory.com/193)
