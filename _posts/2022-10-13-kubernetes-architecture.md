---
title: ! '[Kubernetes Deep Dive] 기본 아키텍처 파헤치기'
author: detectivecrow
date: 2022-10-13 21:24:00 +0900
categories: [Kubernetes]
tags: [devops, kubernetes]
---

---
# 쿠버네티스는 어떻게 구성되어 있을까?

우리가 어떠한 툴이나 프레임워크를 사용하고자 할 때, 사용할 것의 구조나 원리를 잘 파악하고 있으면 생산성 및 트러블슈팅 등에 크게 도움이 될 것입니다.

이 글은 제가 쿠버네티스를 사용하면서 저 스스로의 개념 정리와 동시에 쿠버네티스에 처음 입문하는 사람들에게 전반적인 쿠버네티스의 아키텍처를 설명하여 이해를 돕고자 작성하게 되었습니다.

> 본 글은 기본적으로 udemy의 **[Certified Kubernetes Administrator (CKA) with Practice Tests](https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/)** 의 강의와 개인적으로 찾은 자료들을 기반으로 정리하였습니다.
{: .prompt-info }

# 들어가기에 앞서

쿠버네티스의 목적은 여러분의 어플리케이션을 컨테이너 형태로 호스팅하기 위함입니다.

또한 많은 부분들이 자동화되어서 쉽게 많은 양의 인스턴스를 쉽게 배포하고 인스턴스간의 통신을 손쉽게 설정할 수 있습니다.

밑에서는 쿠버네티스의 구조를 배와 항구의 모습으로 비유하여 전체적인 구조를 설명하고, 요소 하나하나를 자세히 설명해보도록 하겠습니다.

# 쿠버네티스의 구조

![쿠버네티스 구조](/posts/20221013/kubernetes-model-architecture.png)
_ref : [https://phoenixnap.com/kb/understanding-kubernetes-architecture-diagrams](https://phoenixnap.com/kb/understanding-kubernetes-architecture-diagrams)_

## 마스터 노드와 워커 노드

쿠버네티스 클러스터는 기본적으로 노드의 집합입니다.

아래에 두 종류의 선박이 존재합니다. 하나는 화물선(Cargo ships)으로 실질적으로 여러 컨테이너들을 싣고 운용하는 선박이고, 다른 하나는 운용선(Control ships)으로 화물선들에 대한 관리와 모니터링에 의무를 가지고 있는 선박입니다.

![마스터 노드, 워커 노드](/posts/20221013/master-worker-node.png){: style="max-width: 70%"}
_마스터 노드의 역할을 하는 운용선 (Control ships) 과 워커 노드의 역할을 하는 화물선 (Cargo ships)_

- 워커 노드는 여러 어플리케이션을 컨테이너 형태로 호스트하는 노드로 실질적으로 어플리케이션이 동작하는 노드입니다.
- 마스터 노드는 여러 구성 요소와 상호작용하며 워커 노드에 컨테이너를 실행시키고 실행시키기 위한 계획을 하며 노드의 상태를 모니터링하는 노드입니다.

마스터 노드는 이러한 동작을 control plane component 라고 불리우는 컴포넌트의 집합을 이용하여 관리합니다. 

# ETCD Cluster

화물선에는 매일 많은 컨테이너들이 적재, 하역됩니다. 따라서 이러한 컨테이너들의 관리를 위해서는 어떤 배에 어떤 컨테이너가 적재되어있는지, 언제 적재되었는지 등의 정보를 가지고 있어야 합니다.

- ETCD는 key-value 형식의 데이터베이스로 클러스터 관리에 필요한 정보들을 보존하고 있습니다.

![etcd 클러스터](/posts/20221013/etcd-cluster.png){: style="max-width: 70%"}

# kube-scheduler

화물선이 도착하면 여러분은 크레인을 이용하여 컨테이너들을 해당 선박에 적재할 것입니다.
크레인은 화물선에 적재되어야 할 컨테이너를 식별하고, 또한 적재하기 알맞은 화물선인지 배의 용량과 현재 적재되어있는 컨테이너의 수, 배의 상태 등을 고려하여 판단합니다.

- kube-scheduler는 이처럼 컨테이너의 리소스 요구량, 워커 노드의 공간, taint, tolerations, node affinity 등과 같은 정책과 제약 등을 확인하여 실행하기 적절한 노드인지 판단하는 역할을 합니다.

![kube 스케줄러](/posts/20221013/kube-scheduler.png){: style="max-width: 70%"}

# controller-manager

또한 다른 배들과 커뮤니케이션하며 서비스를 정상적인 상태로 동작하게 하는 요소들이 존재합니다.

첫째로 운용선에는 화물선들의 상태를 체크하고 트래픽 컨트롤을 담당하는 오퍼레이션팀이 존재하며

둘째로는 화물선에 적재되어있는 컨테이너의 상태를 관리하며 컨테이너가 손상되었거나 문제가 있으면 컨테이너를 새로운 것으로 교체하는 등의 일을 하는 화물팀이 존재합니다.

- 이처럼 서로 다른 영역을 담당하는 node-controller와 replication-controller 가 존재합니다.
- node-controller는 아래와 같이 노드에 관한 작업들을 담당합니다.
    - 클러스터에 새로운 노드를 추가하는 작업
    - 노드가 unavailable 상태일 때 제거
- replication-controller는 여러분이 요청한 replication 컨테이너의 수를 보장하는 역할을 합니다.

![컨트롤러 매니저](/posts/20221013/controller-manager.png){: style="max-width: 70%"}

# kube-apiserver

지금까지 쿠버네티스에 각각의 역할이 있는 컴포넌트들, 즉 여러 배들, 데이터 스토어, 크레인 등이 존재한다는 것을 알아보았습니다.

하지만 이런 요소들이 어떻게 서로 커뮤니케이션 할까요? 또 어떻게 한 관리자가 높은 레벨에서 모든 것들을 관리하게 할 수 있을까요?

- kube-apiserver는 쿠버네티스의 주된 관리 컴포넌트로 클러스터의 모든 동작을 오케스트레이팅하는 역할을 합니다.
- 이것은 외부에서 관리자가 관리 작업을 할 수 있도록 Kubernetes API를 외부로 노출시킵니다.

![kube api서버](/posts/20221013/kube-apiserver.png){: style="max-width: 70%"}

# Container Runtime Engine

그리고 모든 노드에게 컨테이너의 규격을 정해주고 동작시키기 위한, 즉 컨테이너를 동일하게 실행시킬 수 있는 엔진이 필요합니다.

- 대표적인 예시는 도커이며 그 외에도 로켓이나 크라이오 등 다양한 CRI(Container Runtime Interface) 규격을 만족시키는 컨테이너 런타임 엔진이 존재합니다.

![컨테이너 런타임](/posts/20221013/container-runtime-engine.png){: style="max-width: 70%"}

# kubelet

모든 배에는 선장이 존재합니다. 선장은 배에서 일어나는 모든 활동에 대해 책임지고 관리하는 역할을 합니다.
또한 운용선에 컨테이너와 관련된 정보(컨테이너의 상태, 적재되었다, 컨테이너에 문제가 있다 등)를 보고할 의무를 가지고 있습니다.

- 쿠버네티스에서는 kubelet이 이러한 역할을 합니다. kubelet은 클러스터내의 각 노드에서 동작하는 에이전트로, kube-apiserver로부터의 명령을 듣고 필요하다면 노드에 컨테이너를 생성하고 제거합니다.
- kube-apiserver는 주기적으로 kubelet으로부터 노드와 컨테이너의 상태를 보고 받아 모니터링 합니다.

![kubelet](/posts/20221013/kubelet.png){: style="max-width: 70%"}

# kube-proxy

그런데 이러한 구조에서 한 노드에 존재하는 컨테이너가 다른 노드에 있는 컨테이너와 통신하려면 어떻게 해야 될까요?

노드 간의 커뮤니케이션은 노드에서 동작하는 또 다른 컴포넌트인 kube-proxy에 의해서 가능하게 됩니다.

kube-proxy는 노드에 존재하는 컨테이너가 다른 노드에 있는 컨테이너로 도달할 수 있게 필요한 규칙들을 보장해주는 역할을 합니다.

+ 좀 더 깊게 들어가 CNI와 kube-proxy의 관계에 대해서 보고자 한다면 아래의 링크를 참고하면 좋을 것 같습니다.

[https://stackoverflow.com/questions/53534553/kubernetes-cni-vs-kube-proxy](https://stackoverflow.com/questions/53534553/kubernetes-cni-vs-kube-proxy) 

# 참고

- **[Certified Kubernetes Administrator (CKA) with Practice Tests]** [https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/](https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/)
- **[Kubernetes-Architecture-Diagrams]** [https://phoenixnap.com/kb/understanding-kubernetes-architecture-diagrams](https://phoenixnap.com/kb/understanding-kubernetes-architecture-diagrams)