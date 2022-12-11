---
title: ! '컨테이너란 무엇인가'
author: detectivecrow
date: 2022-11-29 18:36:00 +0900
categories: [Container, Kubernetes, Docker]
tags: [container, kubernetes]
---

---

# 컨테이너? VM? 무엇이 다른걸까?

근래 몇 년 사이에 컨테이너에 대한 관심이 급증하였습니다.

컨테이너 기술은 훨씬 전인 2000년도에 이미 존재하였지만 최근에 눈에 띄게 성장하게 된 데에는 2013년에 Killer 툴인 Docker의 등장 그리고 도커 스웜, 쿠버네티스 등의 컨테이너 기반의 마이크로 서비스화 된 인프라 구성이 대세가 되면서 변한 것이 있지 않을까 생각됩니다.

그런데 컨테이너란 정확히 무엇을 말하는 것일까요?

사실 대부분의 사람들이 가벼운 VM 정도로 이해하고 사용하고 있을 것이라고 생각됩니다.
컨테이너가 무엇인지 정확히 말하자면 이러한 인식은 잘못되었다고 볼 수 있습니다.

본 글에서는 컨테이너란 무엇인지, 기존 VM과의 차이점, 컨테이너의 특징 등에 대해서 자세히 알아보려합니다.

# 그래서 컨테이너란 무엇인가

컨테이너를 최대한 단순하게 표현하자면 아래와 같을 것입니다.

**<ins>리눅스 커널 위에서 동작하는 프로세스</ins>**

좀 더 자세하게 설명해보자면 컨테이너는 하나 혹은 복수 개의 어플리케이션이며, 그것들을 실행시키기 위한 모든 종속성을 포함하고 있는 패키지입니다.

또한 컨테이너는 가상 머신이 아닌 런타임으로 컨테이너는 지정된 특정 동작(프로세스)이 종료되면 회수되게 됩니다.



# 기존 VM과의 차이점

아마도 아래와 같은 그림을 자주 보셨을 것입니다.

![Virtual machines vs Containers](/posts/20221129/vm_vs_container.png)
_ref : [https://www.atlassian.com/microservices/cloud-computing/containers-vs-vms](https://www.atlassian.com/microservices/cloud-computing/containers-vs-vms)_

대표적으로 컨테이너와 가상 머신의 차이점을 보여주는 그림입니다. 여기서 좀 더 자세하게 그려보자면 


# 참고

- **[Linux 컨테이너란?]** [https://www.redhat.com/ko/topics/containers/whats-a-linux-container](https://www.redhat.com/ko/topics/containers/whats-a-linux-container)


# 각주

[^Linux 컨테이너란?]: https://www.redhat.com/ko/topics/containers/whats-a-linux-container

개요

근래 몇년 사이에 컨테이너에 대한 관심 급증 (도커, 쿠버네티스)
그런데 컨테이너란 무엇인가 제대로 알고 쓰지 못하고 있다
컨테이너란 무엇인지, 기존의 VM과 다른점, 컨테이너의 특징 등을 알아보겠다

컨테이너란?

기존 VM과의 차이점

컨테이너의 특징

도커를 이용하여 확인해보기

