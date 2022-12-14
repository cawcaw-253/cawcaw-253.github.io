---
title: ! '컨테이너란 무엇인가'
author: detectivecrow
date: 2022-11-29 18:36:00 +0900
categories: [Container, Docker]
tags: [container]
---

---

# 컨테이너? VM? 무엇이 다른걸까?

근래 몇 년 사이에 컨테이너에 대한 관심이 급증하였습니다.

컨테이너 기술은 훨씬 전인 2000년도에 이미 존재하였지만 최근에 눈에 띄게 성장하게 된 데에는 2013년에 Killer 툴인 Docker의 등장 그리고 도커 스웜, 쿠버네티스 등의 컨테이너 기반의 마이크로 서비스화 된 인프라 구성이 대세가 되면서 변한 것이 있지 않을까 생각됩니다.

그런데 컨테이너란 정확히 무엇을 말하는 것일까요?

사실 대부분의 사람들이 가벼운 VM 정도로 이해하고 사용하고 있을 것이라고 생각됩니다.
컨테이너가 무엇인지 정확히 말하자면 이러한 인식은 잘못되었다고 볼 수 있습니다.

본 글에서는 컨테이너란 무엇인지, 기존 VM과의 차이점, 컨테이너의 특징 등에 대해서 자세히 알아보려 합니다.

# 컨테이너란?

보통 컨테이너 기술이라고 하면 Docker로 대표되는 LXC(Linux Containers)[^Linux_Containers]를 지칭할 것입니다.

그럼 Linux 컨테이너 기술은 어떠한 기술일까요? 해당 내용을 살펴보면 컨테이너가 무엇인지 더 잘 이해할 수 있을 것입니다.

![Linux Kernel](/posts/20221129/linux_kernel.png){: style="max-width: 75%" }

리눅스는 기본적으로 물리적 자원을 관리하는 **커널 공간**과 사용자 프로세스(어플리케이션)를 실행하는 **사용자 공간**으로 나뉩니다.

컨테이너 기술은 여기서 사용자 공간을 Namespace[^Linux_Namespace]로 논리적으로 나누어 각각의 사용자 프로세스에서 보이는 리소스를 제한하는 기술입니다.

따라서 우리가 생각하는 컨테이너를 정말 간략하게 설명하자면 아래와 같습니다.

**<ins>리눅스 커널 위, Namespace에의해 논리적으로 격리된 환경에서 동작하는 프로세스의 집합</ins>**

> 컨테이너는 단순 프로세스이므로 기본적으로는 런타임[^Runtime]입니다. 따라서 해당 프로세스가 완료되면 컨테이너가 종료되게 됩니다. 
>
> 이것이 우리가 `Docker Run`으로 Ubuntu 이미지를 실행시켰을 때 bash나 어플리케이션을 지정해주지 않으면 바로 종료되는 이유입니다.
{: .prompt-tip }

# 컨테이너의 특징

## 장점

- 가벼운 시작 및 종료

  기존 VM과 비교하면 컨테이너는 OS입장에선 단순 프로세스를 실행시키는 것과 다르지 않기 때문에 굉장히 빠르게 실행될 수 있습니다.

- 높은 집적도

  컨테이너는 실질적으로 커널이 직접 프로세스를 조작하고 격리된 공간을 구성합니다.
  따라서 하드웨어상에서 동작하는 OS는 하나로, 여러 개의 OS가 동작해야 하는 VM 기술에 비하면 소비하는 자원은 훨씬 적습니다.

- 일관성 있는 환경

  개발자는 컨테이너를 이용해, 다른 어플리케이션과 분리된 최적화된 환경을 생성할 수 있습니다.
  따라서 어플리케이션의 설계가 잘 되어있다면 개발자와 운영팀이 버그를 잡고 환경 차이를 진단하던 시간을 줄이고 사용자에게 신규 기능을 제공하는 데에 집중할 수 있게 됩니다.

- 이동성

  컨테이너는 개발자가 자신의 PC에서 개발한 컨테이너를 그대로 퍼블릭 클라우드나 다른 개발자의 PC에 가져가 실행할 수 있습니다.
  동일한 컨테이너 런타임을 이용하는 환경 혹은 OCI(Open Container Initiative) 스펙으로 만들어진 런타임 위에서는 어디서나 동일하게 동작하며 이로 인해 어디서나 손쉽게 개발하고 테스트할 수 있습니다.

## 단점

- 호스트 서버 의존성

  컨테이너는 기본적으로 물리 서버 한 대에 여러대의 서비스가 동작하는 방식으로 많이 이용됩니다. 
  이는 다르게 보면 호스트 서버에 문제가 생기면 모든 컨테이너에 영향을 미친다는 것을 의미합니다.

- 엔지니어들의 전문성 부족

  컨테이너 기술은 비교적 최근에 넓게 사용되기 시작한 기술로 경험이 있는 IT 엔지니어, 개발자들이 많지 않습니다.
  따라서, 기술적 전문성 부족으로 인하여 기존 환경보다 더 쉽게 보안 취약점이 노출될 수 있고 러닝커브등이 문제가 될 수 있습니다,


# 기존 VM과의 차이점

![Virtual machines vs Containers](/posts/20221129/vm_vs_container.png)
_ref : [https://www.atlassian.com/microservices/cloud-computing/containers-vs-vms](https://www.atlassian.com/microservices/cloud-computing/containers-vs-vms)_

![Virtual machines vs Containers (Kernel)](/posts/20221129/vm_vs_container_kernel.png){: style="max-width: 50%" }

대표적으로 컨테이너와 가상 머신의 차이점을 보여주는 그림입니다. 

**VM(가상 머신)**은 머신을 프로비저닝 해주는 하이퍼바이저가 각 VM을 프로비저닝 하면서 Kernel(Guest OS)를 생성하여 어플리케이션이 해당 Kernel에 시스템 호출을 하도록 구성하는 반면

**Container**는 Host의 커널에 여러 컨테이너가 시스템 호출로 상호작용하며 동작하는 구조입니다.

따라서 컨테이너는 OS입장에선 프로세스를 실행시키는 것이기에 기존 VM보다 가볍고 빠르게 프로비저닝 될 수 있고, 직접 프로세스를 조작하기에 적은 리소스로 높은 집적도를 이끌어낼 수 있으며, 낮은 사양의 환경에서도 더욱 활용도가 높다는 장점이 있습니다.

하지만 Host OS에 종속적이라는 단점 또한 존재하므로 요구사항에 따라서는 무조건 컨테이너가 좋다고는 할 수 없을 것입니다.

# 도커의 컨테이너 격리 확인

1. 우선 아래의 명령어로 `sleep 1d` 커맨드를 실행하는 컨테이너를 실행하고, ps 커맨드를 통해서 컨테이너 안의 프로세스 리스트를 확인합니다.
  ![Run simple container](/posts/20221129/explain_0.jpeg)

    ```sh
    # 테스트 컨테이너 1 실행
    docker run --name container-1 -d ubuntu sh -c 'sleep 1d'
    
    # ps 커맨드로 테스트 컨테이너 1의 프로세스 리스트를 확인
    docker exec container-1 ps aux 
    ```

2. 그리고 `sleep 999d` 커맨드를 실행하는 두번째 컨테이너를 실행하고, 마찬가지로 컨테이너 안의 프로세스 리스트를 확인합니다.
  ![Run simple container](/posts/20221129/explain_1.jpeg)
  
    ```sh
    # 테스트 컨테이너 2 실행
    docker run --name container-2 -d ubuntu sh -c 'sleep 999d'
    
    # ps 커맨드로 테스트 컨테이너 1의 프로세스 리스트를 확인
    docker exec container-2 ps aux 
    ```

3. 이제 호스트에서 프로세스 리스트를 확인해봅니다.
  ![Run simple container](/posts/20221129/explain_2.jpeg)

    ```sh
    # sleep 을 실행중인 프로세스 리스트를 확인
    # 이 명령어를 통해서 컨테이너는 OS, 커널에서 프로세스를 실행하고 논리적으로 격리하고 있다는 것을 확인할 수 있습니다.
    ps aux | grep sleep
    ```

4. 여기서 기존 테스트 컨테이너 2를 지우고, pid 옵션으로 기존 컨테이너와 네임스페이스를 공유하게 해주고 다시 컨테이너의 프로세스 리스트를 확인해봅니다. 
  ![Run simple container](/posts/20221129/explain_3.jpeg)
  
    ```sh
    # 테스트 컨테이너 1과 네임스페이스를 공유하는 컨테이너 2를 생성합니다.
    docker run --name container-2 --pid=container:container-1 -d ubuntu sh -c 'sleep 999d'
    
    # ps 커맨드로 테스트 컨테이너 1, 2의 프로세스 리스트를 확인
    # 이 명령어를 통해서 동일한 네임스페이스를 사용하면 서로의 프로세스를 볼 수 있다는 것을 확인할 수 있습니다.
    docker exec container-1 ps aux
    docker exec container-2 ps aux
    ```

# 마치며

컨테이너는 최근 굉장히 다양한 용도로 사용되고 연구되어 점점 인프라의 필수 구성요소로 다가오고 있습니다.

제가 정리한 내용은 제가 이해한 것들을 바탕으로 컨테이너 기술의 기본, 근본적인 부분을 간략하게 정리해 본 것입니다. 제가 정리한 내용보다 다양하고 심도 깊은 내용들은 검색하면 얼마든지 찾을 수 있으니 이 글을 읽으시고 컨테이너 기술에 흥미가 생기셨으면 한 번 찾아보시는 것도 좋을 것 같습니다.

글에 대해서 애매하거나 설명이 부족한 부분이 있다면 언제든지 댓글 부탁드립니다.

제가 경험하고 공부했던 부분들이 도움이 되었으면 하며 즐거운 데브옵스 되세요! :)

# 참고

- **[Linux 컨테이너란?]** [https://www.redhat.com/ko/topics/containers/whats-a-linux-container](https://www.redhat.com/ko/topics/containers/whats-a-linux-container)
- **[LXC 와 Docker 그리고 Linux...]** [http://www.digistory.co.kr/?p=733](http://www.digistory.co.kr/?p=733)
- **[리눅스 네임스페이스란?]** [https://www.44bits.io/ko/keyword/linux-namespace](https://www.44bits.io/ko/keyword/linux-namespace)
# 각주

[^Linux_Containers]: https://en.wikipedia.org/wiki/LXC
[^Linux_Namespace]: https://en.wikipedia.org/wiki/Linux_namespaces
[^Runtime]: https://en.wikipedia.org/wiki/Runtime_(program_lifecycle_phase)
<!-- 
개요

근래 몇년 사이에 컨테이너에 대한 관심 급증 (도커, 쿠버네티스)
그런데 컨테이너란 무엇인가 제대로 알고 쓰지 못하고 있다
컨테이너란 무엇인지, 기존의 VM과 다른점, 컨테이너의 특징 등을 알아보겠다

컨테이너란?

기존 VM과의 차이점

컨테이너의 특징

도커를 이용하여 확인해보기 -->

