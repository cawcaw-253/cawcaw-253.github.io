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

Kubelet의 로그를 보던 중 지속적으로 ControlPlane과 통신하며 Health Check를 하는 과정에 에러가 발생한 적이 있습니다.

트러블슈팅 하면서 Node Health Check에서 Lease가 있다는 것은 알았지만 자세히는 몰랐는데, 이번 기회에 공부하게 되었습니다. 

이번 글에서는 Lease가 어떤 기능을 하는지 그리고 어떻게 삭제, 생성되는지에 대해서 정리해 보도록 하겠습니다.

# Lease의 기능

Lease는 일반적으로 분산된 시스템에서 여러 멤버가 협력하여 동작할 때 사용됩니다.
이는 공유 리소스를 lock 하고 각 노드 간의 활동을 조정하는 역할을 합니다.

쿠버네티스에서는 Node Heartbeats를 통해 Node Health를 모니터링할 때 사용하고, 또한 component 레벨의 리더 선출에 사용됩니다.
그리고 애플리케이션을 실행할 때 고가용성 설정의 단일 구성원이 요청을 서비스할 수 있도록 하기 위해 Kubernetes Lease API를 사용합니다.

아래에서 각각의 기능에 대해서 자세히 설명해 보도록 하겠습니다.
## Node Heartbeat

쿠버네티스는 Lease API를 사용하여 kubelet 노드의 하트비트를 쿠버네티스 API Server에 전달합니다.
모든 노드에는 아래와 같이 같은 이름을 가진 Lease 오브젝트가 kube-node-lease namespace에 존재합니다.