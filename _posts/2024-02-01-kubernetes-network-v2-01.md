---
title: "[Kubernetes Deep Dive v2] 쿠버네티스 서비스"
author: cawcaw253
date: 2024-02-01 21:55:00 +0900
categories:
  - Kubernetes
tags:
  - kubernetes
  - network
  - deep-dive
---
# 개요

지난번에는 [[Kubernetes Deep Dive]쿠버네티스 네트워크](https://blog.cawcaw253.com/posts/kubernetes-pod-network/) 를 통해서 쿠버네티스 내부에서 어떻게 패킷이 이동하고 서로 통신할 수 있는지를 간단하게 설명했었습니다.
이번에는 우리가 쿠버네티스를 사용하면서 항상 사용하는 서비스에 대해서 조금 더 깊게 파고들어서 정리해보고자 합니다.
단순히 서비스가 어떻게 동작하는가를 넘어서 CNI, kube-proxy, iptables등의 내용과 함께 설명해보도록 하겠습니다.

# Service의 동작 방식

각 Service Type별 동작 방식을 보기에 앞서 실제로 service를 생성했을 때 클러스터 내부에서 어떠한 동작이 일어나는지 알아보도록 하겠습니다.


## kube-proxy

# Cluster IP

> 서비스를 클러스터-내부 IP에 노출시킨다. 이 값을 선택하면 클러스터 내에서만 서비스에 도달할 수 있다. 이것은 서비스의 `type`을 명시적으로 지정하지 않았을 때의 기본값이다.
> 
> ref : https://kubernetes.io/ko/docs/concepts/services-networking/service/#publishing-services-service-types

Cluster IP 방식은 가장 기본적인 방식으로 클러스터 내부에서 해당 service 
## Single Node


## Multi Node

