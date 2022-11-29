---
title: ! 'DevOps 콘셉트: Pets vs Cattle'
author: detectivecrow
date: 2022-08-15 18:23:00 +0900
categories: [DevOps, Infrastructure]
tags: [devops, infrastructure]
---

---
# Pets vs Cattle 이란?

클라우드 컴퓨팅이 발전함에 따라, 비즈니스는 그들의 어플리케이션과 고가용성, 빠른 응답시간을 가진 경쟁력있는 서비스를 원하게 되었습니다.

이러한 요구사항에 따라서 서비스를 구성하는 인프라에 많은 변화가 생겨나기 시작했습니다.

그리고 그러한 인프라의 변화를 비유하는 표현으로 Pets vs Cattle(애완동물 대 가축)이라는 문구가 굉장히 유명해졌죠

![pets vs cattle](/posts/20220815/pets_vs_cattle.png)
_ref : [https://traefik.io/blog/pets-vs-cattle-the-future-of-kubernetes-in-2022/](https://traefik.io/blog/pets-vs-cattle-the-future-of-kubernetes-in-2022/)_

> 이 문구에서 기존의 온프레미스 데이터 센터는 일반적으로 “Pets”이며, 클라우드에 존재하는 서버는 “Cattle”입니다.
Pets(애완동물)는 없어져서는 안되는 서버로 문제가 발생하면 우리들이 직접 설정을 변경할 수 있습니다.
반면, Cattle(가축)은 문제가 있을 경우 언제든지 삭제하고 다시 구축할 수 있는 서버입니다.
> 

이 글에서는 위와 같은 인프라의 형태를 설명하는 Mutable, Immutable Infrastructure에 대해서 설명하고 어째서 Immutable Infrastructure가 기존의 것을 대체하기 시작하였는지 설명하고자 합니다.

> 📢 본 글은 **[DevOps Concepts: Pets vs. Cattle](https://iamondemand.com/blog/devops-concepts-pets-vs-cattle/#:~:text=Servers%20in%20on%2Dpremises%20data,scratch%20in%20case%20of%20failures.)** 의 내용을 공부하며 번역한 글입니다.
{: .prompt-info }

---
# Mutable vs Immutable Infrastructure

온프레미스 데이터 센터와 클라우드 서버는 우리가 어떻게 서버를 취급하는가에 있어서 큰 차이가 존재합니다.

온프레미스 서버는 삭제할 수 없으며 계속 유지해야 되는 존재로 취급하며, AWS, Azure등과 같은 클라우드 프로바이더가 제공하는 클라우드상에 존재하는 서버는 일시적 혹은 임시로 받은 리소스로 취급합니다.

![가변적, 불변적 아키텍처](/posts/20220815/mutable_immutable.png)
_Mutable vs Immutable architecture_

## Mutable Infrastructure

수동으로 직접 조작할 수 있는 서버로, 서버에 로그인하고 구성을 변경, 패키지를 설치/수정할 수 있습니다.

## Immutable Infrastructure

한 번 배포된 서버는 수정할 수 없습니다. 만약 우리가 설정을 변경하고 싶다면 기존에 존재하는 이미지를 업데이트하고 오래된 서버를 새로운 서버로 대체합니다.

즉 코드나 설정의 변경은 서버가 돌아가는 도중에는 허락되지 않습니다.

---
# Benefits of Immutable Infrastructure

현재 분산 시스템은 Immutable Infrastructure 개념을 빠르게 채택하고 있습니다.
아래에서는 시스템 아키텍처에 Immutable Infrastructure 개념을 도입하면 얻을 수 있는 이점에 대해 소개하도록 하겠습니다.

## 작업의 단순화

구성 요소 내에서 드리프트 감지를 처리할 필요가 없고 구성 요소를 수정하는 대신 교체할 수 있으면 작업이 훨씬 간단해집니다.
따라서 많은 수의 프로덕션 서버를 관리할 필요가 없으며 이는 개발 팀이 어플리케이션 개발에 집중하고 인프라의 자동화가 환경 간의 일관성을 보장하도록 할 수 있음을 의미합니다.

> 여기서 드리프트란 “예측하지 못한 변화” 를 의미합니다.
{: .prompt-tip }

## 장애로부터의 빠른 복구

인프라를 다룰 때 동작 실패는 불가피합니다. 하지만 자동화된 방식으로 노드를 교체할 수 있는 기능이 있다면 장애를 신속하게 복구하는데 도움이 됩니다.

## 인프라의 동적 확장

만약 베이스 이미지를 사용하여 인프라를 자동으로 시작하도록 하면, 어플리케이션의 로드에 따라서 환경을 동적으로 확장할 수 있습니다.

짧은 시작 시간으로 동일한 인스턴스를 스핀업할 수 있는 기능은 동적 확장을 위한 전략을 구현하고 인프라 리소스 사용을 최적화 하는데 도움이 됩니다.

그 외에도, 위험 요소의 최소화, 인프라의 보안 강화등의 다른 이점 또한 존재합니다.
이러한 다른 이점은 참고의 **[DevOps Concepts: Pets vs. Cattle]** 에서 자세하게 확인할 수 있습니다.

# 결론

기존 어플리케이션을 클라우드 네이티브 공간으로 마이그레이션을 하려면 인프라를 자동화하여 서비스 가동 시간과 응답 시간을 늘려야 합니다.
이러한 변화에서 Immutable Infrastructure로의 전환은 매우 중요하며 기업이 가동 중지 시간을 최소화하면서 더 나은 서비스를 제공할 수 있도록 돕습니다.

# 참고

- **[DevOps Concepts: Pets vs. Cattle]** [https://iamondemand.com/blog/devops-concepts-pets-vs-cattle/#:~:text=Servers in on-premises data,scratch in case of failures](https://iamondemand.com/blog/devops-concepts-pets-vs-cattle/#:~:text=Servers%20in%20on%2Dpremises%20data,scratch%20in%20case%20of%20failures).
- **[Pets vs. Cattle]** [https://traefik.io/blog/pets-vs-cattle-the-future-of-kubernetes-in-2022/](https://traefik.io/blog/pets-vs-cattle-the-future-of-kubernetes-in-2022/)