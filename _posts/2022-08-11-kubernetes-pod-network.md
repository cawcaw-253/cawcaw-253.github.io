---
title: Kubernetes Pod Network
author: detectivecrow
date: 2022-08-11 21:55:00 +0800
categories: [Kubernetes]
tags: [kubernetes, network]
---

# 개요

쿠버네티스의 네트워크는 컨테이너의 네트워크를 기반으로 구성되어 있습니다. 따라서 docker와 비슷한데, 완전히 같지는 않고 차이점들이 존재합니다. 이번에는 컨테이너의 네트워킹을 이해하고 그것을 넓혀서 쿠버네티스의 파드 네트워킹을 이해해 보고자 합니다.

# Docker의 네트워킹

![[https://docs.docker.com/engine/tutorials/networkingcontainers/](https://docs.docker.com/engine/tutorials/networkingcontainers/)](/posts/20221007/01_docker_networking_1.png)

![docker_networking_2.png](/posts/20221007/02_docker_networking_2.png)

위의 그림은 도커의 기본적인 `bridge` 타입 네트워크의 구조입니다.

컨테이너가 외부와 연결을 해야할 경우 호스트에 `veth (Virtual )` 이라는 **네트워크 네임스페이스** 를 생성하고 컨테이너의 `eth` 와 연결됩니다. 

또한 `docker0` 라는 가상 브릿지도 존재하는데, 이는 `veth` 인터페이스와 바인딩 되어 호스트의 eth 인터페이스와 연결해줍니다. 

# Pod의 네트워크 인터페이스

쿠버네티스는 도커와 달리 파드 단위로 컨테이너들을 관리합니다. 여기서 파드는 여러개의 컨테이너로 구성될 수 있는데, 파드에 속한 컨테이너들은 모두 같은 가상 네트워크 인터페이스를 공유합니다.

여기서 어떻게 파드는 가상 네트워크 인터페이스를 공유할까요?  바로 pause 컨테이너의 존재 덕분입니다.

![03_pod_network_interface](/posts/20221007/03_pod_network_interface.png)

[https://medium.com/google-cloud/understanding-kubernetes-networking-pods-7117dd28727](https://medium.com/google-cloud/understanding-kubernetes-networking-pods-7117dd28727)

파드가 실행될 때 pause라는 컨테이너가 자동으로 실행되고, 이 pause 컨테이너의 리눅스 네임스페이스, 네트워크 인터페이스를 파드에 속한 모든 컨테이너들이 공유해서 사용하게 됩니다.

좀 더 자세한 설명은 아래의 링크를 통해서 확인할 수 있습니다.

- [https://medium.com/google-cloud/understanding-kubernetes-networking-pods-7117dd28727](https://medium.com/google-cloud/understanding-kubernetes-networking-pods-7117dd28727)
- [https://velog.io/@200ok/Kubernetes-K8S-Flannel-CNI-PAUSE-컨테이너-동작원리-이해하기](https://velog.io/@200ok/Kubernetes-K8S-Flannel-CNI-PAUSE-%EC%BB%A8%ED%85%8C%EC%9D%B4%EB%84%88-%EB%8F%99%EC%9E%91%EC%9B%90%EB%A6%AC-%EC%9D%B4%ED%95%B4%ED%95%98%EA%B8%B0)

# Pod간의 통신

기본적으로 쿠버네티스는 kubenet이라는 네트워크 플러그인을 제공해줍니다. 하지만 이 플러그인은 `노드간 네트워킹` 이나 `네트워크 정책설정` 과 같은 기능이 구현되어 있지 않아 CNI 플러그인을 따로 사용해 통신을 돕습니다.

![단일 노드에서의 Pod간 통신](/posts/20221007/04_pod_networking_single_node.png)

단일 노드에서의 Pod간 통신

CNI 플러그인을 사용하면 위의 그림과 같이 각 파드가 고유한 IP를 가지게 되고 이를 통해서 각 파드는 CNI를 통하여 고유한 IP 주소로 통신할 수 있게 됩니다.

![멀티 노드에서의 Pod간 통신](/posts/20221007/05_pod_networking_multi_node.png)

멀티 노드에서의 Pod간 통신

여러 노드를 가진 환경에서 Pod에서 다른 노드에 있는 Pod에 요청을 보내면 여러 단계를 걸쳐서 통신하게 됩니다.

Pod → veth → CNI(Flannel) → NAT → eth0 → Router → eth1 → NAT → CNI(Flannel) → veth → Pod

이러한 복잡한 과정을 Pod Network, CNI 플러그인에서는 네트워크 모델을 구성해 위의 과정을 추상화시켜 간단하게 사용할 수 있도록 만들어줍니다.

# CNI 플러그인이란 무엇인가

> CNCF(Cloud Native Computing Foundation)의 프로젝트 중 하나인 CNI는 컨테이너 간의 네트워킹을 제어할 수 있는 플러그인을 만들기 위한 표준입니다.
컨테이너의 발전이 가속화 됨에 따라 다양한 형태로 컨테이너 런타임과 오케스트레이터 사이의 네트워크 계층을 구현하는 방식을 각자의 방식으로 발전하게 되는 것을 피하기 위해 공통된 인터페이스를 제공하려는 목적으로 만들어졌다.

refer to : [https://github.com/containernetworking/cni#what-is-cni](https://github.com/containernetworking/cni#what-is-cni)
> 

쿠버네티스 뿐만 아니라 Amazon ECS, Apache Mesos, rks 등의 다양한 플랫폼들 또한 네트워크 인터페이스로 CNI를 사용하고 있습니다.
여기서 쿠버네티스는 기본적으로 `kubenet` 이라는 자체적인 CNI 플러그인을 제공하지만 기능이 매우 제한적이어서 보통은 Flannel, Calico, Weavenet 등의 다양한 3rd-party CNI 플러그인들을 사용합니다.

## 왜 CNI 플러그인이 필요한가

![06_why_cni_plugin_1.png](/posts/20221007/06_why_cni_plugin_1.png)

위 그림과 같이 UI Container, Login Container, Cart Container등의 컨테이너 기반 애플리케이션이 멀티 호스트 구성으로 동작한다고 가정합니다.

여기서 UI Container(172.17.0.2) 에서 Login Container(172.17.0.2) 로 통신을 하고자 하면 일반적으로 요청이 docker0라는 브릿지를 통해 외부로 전달되어 통신이 될 것입니다. 

하지만 위의 상황에서는 두 컨테이너의 IP가 동일하기 때문에 요청은 자기 자신에게 돌아와 기대하는 동작과는 다르게 움직일 것입니다.

이러한 멀티 호스트 환경에서 컨테이너간의 통신을 하기 위해서는 CNI 플러그인이 필수적으로 필요하게 됩니다.

![07_why_cni_plugin_2.png](/posts/20221007/07_why_cni_plugin_2.png)

CNI 플러그인은 위처럼 **Overlay Network를 구성**하고 **컨테이너 네트워크 대역대를 나눠주며**, **라우팅 테이블을 생성**하여 한 컨테이너가 다른 컨테이너로 통신하는데 문제가 없도록 도와줍니다.

위와 관련된 내용은 아래에서 더욱 자세히 알아보도록 하겠습니다.

## 네트워크 모델

CNI 플러그인들은 본질적 기능인 컨테이너 및 노드간의 통신을 중개하기 위해 네트워크 모델을 사용하는데, 대표적으로 크게 두 가지 형식의 네트워크 모델을 사용합니다.

- `VXLAN(Virtual Extensible LAN)` 혹은 `IP-in-IP` 프로토콜을 사용하는 Overlay Network 모델
- `BGP(Border Gateway Protocol)` 을 사용하는  모델

이는 Encapsulated Network, Unencapsulated Network 라고도 불립니다.

여기서는 기본적인 Overlay Network에 대해서만 정리해보도록 하겠습니다.

### Overlay Network (Encapsulated Networks)

오버레이 네트워크의 목적은 실제로 복잡할 수 있는 엔드포인트 간의 네트워크 구조를 추상화하여 네트워크의 통신 경로를 단순화 하는 것입니다.

![08_overlay_network_1.png](/posts/20221007/08_overlay_network_1.png)

![09_overlay_network_2.png](/posts/20221007/09_overlay_network_2.png)

이 모델은 기존 레이어 3 위에 구축된 네트워크 간에 있는 엔드포인트의 노드간 통신이 일어날 때 패킷을 캡슐화하여 레이어 2 (같은 LAN) 에서 통신이 일어나는 것처럼 통신할 수 있도록 해줍니다.

기본적으로 아래와 같은 동작을 통해 통신을 추상화 합니다.

1. 통신 노드간에 가상 Tunnel을 생성
2. 통신 패킷을 encapsulation
3. 캡슐화 된 패킷을 생성한 가상 Tunnel의 Endpoint를 통해 전달
- **여기서 이러한 처리를 위해 라우팅 테이블처럼 사용되는 것이 바로 key value 스토어인 ETCD 입니다.**

![10_encapsulated_network.png](/posts/20221007/10_encapsulated_network.png)

Overlay Network를 사용하면 거의 대부분의 환경에서 기존 네트워크 환경에 영향 없이 CNI 환경을 구성할 수 있지만, Unencapsulated Network 방식에 비해 캡슐화 할 때 CPU 등의 자원도 소모하고 패킷 당 송수신 할 수 있는 데이터의 양이 줄어들어 상대적으로 비효율 적인 단점이 있습니다.

# 정리

위에서 컨테이너, 파드의 네트워크 인터페이스에 대한 이해, CNI 플러그인이 하는 역할, 네트워크 모델을 통한 노드 간 통신의 추상화 등의 내용을 정리해 보았습니다.

내용이 많아서 저 또한 아직 제대로 이해하고 있지 못한 부분이 많고 틀린 부분이 있을 수 있지만 전체적인 그림을 이해하는 데에 도움이 되었으면 하는 마음으로 작성해보았습니다.

다음에는 쿠버 네티스의 Service와 외부에서 Service에 접근할 수 있게 하는 방식에 대해서 정리해보도록 하겠습니다.

# 참고

- **[CNI 란]** [https://captcha.tistory.com/78#:~:text=CNCF](https://captcha.tistory.com/78#:~:text=CNCF)
- **[쿠버네티스 파드 네트워크 정리]** [https://jonnung.dev/kubernetes/2020/02/24/kubernetes-pod-networking/](https://jonnung.dev/kubernetes/2020/02/24/kubernetes-pod-networking/)
- **[쿠버네티스 네트워크 구성도]** [https://velog.io/@seunghyeon/쿠버네티스-네트워크-구성도](https://velog.io/@seunghyeon/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-%EA%B5%AC%EC%84%B1%EB%8F%84)
- **[Kubernetes CNI Explained]** [https://www.tigera.io/learn/guides/kubernetes-networking/kubernetes-cni/](https://www.tigera.io/learn/guides/kubernetes-networking/kubernetes-cni/)
- **[Networking with Kubernetes]** [https://www.youtube.com/watch?v=WwQ62OyCNz4](https://www.youtube.com/watch?v=WwQ62OyCNz4)
- **[Kubernetes Services networking]** [https://www.youtube.com/watch?v=NFApeJRXos4&t=256s](https://www.youtube.com/watch?v=NFApeJRXos4&t=256s)
- **[Kubernetes Ingress in 5 mins]**(https://www.youtube.com/watch?v=NPFbYpb0I7w)