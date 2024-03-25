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

Kubernetes 클러스터를 생성하고 Helm 차트를 통해 다양한 어플리케이션을 설정하다보면 볼륨에 대한 설정이 항상 등장합니다. 이러한 상황에서 볼륨에 대해서 자세히 모르다보면 이 부분에서 헤메는 경우가 종종 발생하게되죠. 급하게 많이 사용하는 csi-driver 나 longhorn 등을 설치해서 문제를 해결할때가 있습니다.
본 글에서는 여기서 CSI 는 무엇이고 어떤 동작을 하는지에 대해서 CNI 때와 마찬가지로 알아보도록 하겠습니다.

# CSI

Kubernetes v1.9부터 컨테이너 스토리지 인터페이스(Container Storage Interface, CSI)가 알파 버전으로 지원되기 시작했습니다. v1.10에서 베타 버전으로 지원되었고, v1.13에서 GA가 되었습니다.

Kubernetes의 스토리지 관련 기능은 Kubernetes 소스에 직접 내장된 구현("in-tree")으로만 제공되었습니다. 따라서 외부 스토리지를 개발하는 3rd party 벤더는 Kubernetes 소스 코드에 업스트림을 해야 했고, Kubernetes 개발팀과 릴리스 시기를 맞춰야 한다는 문제가 있었습니다.
중간에 v1.8에서 추가된 Flex-volume이라는 플러그인 기반 솔루션이 API를 노출하여 문제를 해결하려고 시도했습니다. 하지만 몇몇 문제점들이 존재하여 넓게 이용되지 못했습니다.

> Flex-volume 솔루션은 k8s 바이너리를 분리한다는 측면에서 CSI와 유사한 원칙에 따라 동작하지만, 이 접근 방식에는 여러 가지 문제가 있었습니다.
> 첫째, 드라이버 파일을 배포하려면 마스터 및 호스트 파일 시스템에 대한 루트 액세스 권한이 필요했습니다.
> 둘째, 호스트에서 사용할 수 있다고 가정한 운영 체제 종속성 및 전제 조건이 상당히 부담스러웠습니다.

이러한 문제를 해결하고자 Kubernetes에서는 CSI를 지원함으로써 Kubernetes 소스에 포함하지 않고 3rd party 벤더가 자체적으로 구현할 수 있는("out-of-tree") 방식으로 제공할 수 있게 되었습니다.
또한, Kubernetes에서 기존에 제공하던 StorageClass, PersistentVolume, PersistentVolumeClaim을 CSI에서도 계속 사용할 수 있어, 컨테이너를 배포하고자 하는 사용자 입장에서는 v1.8 이전과 동일한 조작으로 CSI를 통해 외부 스토리지를 이용할 수 있게 되었습니다.

이 CSI는 Kubernetes, Mesos, Docker, CloudFoundry 등 다양한 컨테이너 환경에서 사용할 수 있도록 Kubernetes와 독립적으로 [Container Storage Interface 커뮤니티](https://github.com/container-storage-interface)에서 사양을 수립하고 있습니다.

![container-storage-interface_diagram](posts/20240322/container-storage-interface_diagram.png)
# Service



# Reference
- [[Kubernetes Deep Dive] 쿠버네티스 네트워크](https://blog.cawcaw253.com/posts/kubernetes-pod-network/)
- [kube-proxy](https://kodekloud.com/blog/kube-proxy/)
- [iptables-or-ipvs](https://www.tigera.io/blog/comparing-kube-proxy-modes-iptables-or-ipvs/)
- [introduction-cni](https://kube.academy/courses/kubernetes-in-depth/lessons/an-introduction-to-cni)
- [Kube-Proxy and CNI: The Hidden Components of Kubernetes Networking](https://medium.com/@seifeddinerajhi/kube-proxy-and-cni-the-hidden-components-of-kubernetes-networking-eb30000bf87a)
- [Endpoint란](https://yoo11052.tistory.com/193)


