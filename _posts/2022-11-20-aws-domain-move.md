---
title: ! '[AWS] DNS 변경을 이용한 다운타임 없는 인프라 마이그레이션'
author: detectivecrow
date: 2022-11-20 20:13:00 +0900
categories: [AWS]
tags: [aws]
---

---
# 다운타임 없이 프로덕션 환경 마이그레이션

![Server Migration](/posts/20221120/server-transfer.png)
_ref : [https://icon-icons.com/icon/server-transfer-migration/116623](https://icon-icons.com/icon/server-transfer-migration/116623)_

프로젝트를 진행하다 보면 한 계정에서 다른 aws 계정으로 이동하거나 도메인 밑의 인프라 전체를 새롭게 교체해야 되는 경우가 발생할 수 있습니다.

예를 들어 기존의 인프라를 IaC로 만들어진 새로운 인프라로 교체하는 경우가 있을 것 같습니다.

이러한 경우에 어떠한 방식으로 현재 동작하고 있는 서비스를 옮길 수 있는지 저의 경험을 토대로 설명해 보고자 합니다.

# 고려되어야 될 부분

다운타임 없이 제공 중인 환경을 이전하기 위해서는 생각보다 요구사항이 많습니다.

대표적으로 고려해야 될 점은 아래와 같은 것들이 존재하며 각 항목이 어떤 문제를 가지는지 하나하나 살펴보도록 하겠습니다.

1. 전파시간
2. 어플리케이션 구조
3. 데이터 정합성

## 전파시간

![DNS Propagation](/posts/20221120/dns-propagation.jpg)
_ref : [https://www.xeonbd.com/blog/dns-propagation-take-long/](https://www.xeonbd.com/blog/dns-propagation-take-long/)_

첫째로 전파시간입니다.

프로덕션에서는 기본적으로 도메인을 이용하고 있을 것입니다. 그리고 프로덕션을 새로운 환경으로 이전하게 되면 아무래도 DNS의 레코드를 수정하거나 도메인의 네임서버를 수정하게 될 것입니다.

이러한 부분의 수정은 도메인과 DNS의 전파시간에 영향을 받게 되므로 전반적인 마이그레이션 플랜에 영향을 주게 됩니다.

## 어플리케이션 구조

위의 전파시간이 문제가 되면서 어플리케이션 구조 또한 고려해야 될 부분이 생기게 됩니다.

아무래도 서비스가 강한 의존성을 가지고 있다면 전파시간 중 DNS의 값에 따라서 중간에 다른 인프라로 리퀘스트가 전달되면서 어플리케이션에 문제가 생기거나 사용자 경험에 해가 될 수 있을 겁니다.

또, 각 서비스의 특징에 따라서 어플리케이션에서 특정 동작시 세션이 끊기면 안 된다거나, 데이터 정합성이 발생하면 안 되는 등의 문제가 있을 수 있으므로 어플리케이션 구조 부분은 좀 더 깊게 고려해야 될 부분일 것입니다.

## 데이터 정합성

데이터 정합성 문제 또한 전파시간으로 인해 발생될 수 있습니다.

어플리케이션의 문제보다 더 심각하게 작용할 가능성이 높은 부분으로, 전파 상태에 따라서 특정 지역에서는 OLD로, 다른 지역에서는 NEW로 트래픽을 전달하는 경우가 발생하여 각 환경이 다른 곳에 데이터를 저장한다면 정합성 문제가 발생하게 될 것입니다.

# 어떤 방식으로 마이그레이션 하면 좋을까

제가 진행하던 프로젝트에서는 어플리케이션 구조에는 특별한 문제가 없었고, 데이터 정합성 부분은 백엔드 파트에서 케어하기로 결정을 하였어서 크게 문제 될 부분은 없었습니다.
하지만 기존에 sub-delegation 없이 만들어진 서브 도메인으로 인해서 단순히 도메인의 DNS 값을 변경하는 것뿐만 아니라 추가적인 작업이 필요했었습니다.

아래에서는 제가 진행하려 했던 방법, 그리고 기존 서브 도메인의 sub-delegation 작업을 하나하나 설명하도록 하겠습니다.

## 마이그레이션 계획

처음 생각했던 플랜은 굉장히 심플했습니다.

그 계획은 **<ins>다른 계정에서 기존 환경과 동일한 환경을 구성한 뒤, 도메인의 네임 서버 값을 새로운 hosted zone의 네임 서버 값으로 변경</ins>**하는 것으로, 전파시간동안은 두 환경 모두에게 요청이 갈 수 있지만 전파가 완료된 뒤<ins>(대략 72시간)</ins>에는 새로 만들어진 환경으로 모든 요청이 가게 될 것입니다.

![Migration Plan 001](/posts/20221120/plan_001.png)
_1. 동일한 환경 구성_

![Migration Plan 002](/posts/20221120/plan_002.png)
_2. 도메인의 네임 서버 값을 새로 만들어진 hosted zone의 네임 서버 값으로 변경_

![Migration Plan 003](/posts/20221120/plan_003.png)
_3. 전파시간(72시간)이 지난 후, 트래픽의 흐름_

## 서브 도메인의 sub-delegation 작업

하지만 한가지 문제점이 존재했었습니다. 바로 기존의 개발 환경이 옮겨지지 않는다는 점이었죠.

![Problem 001](/posts/20221120/problem_001.png)
_기존의 개발 환경이 hosted zone에 포함되어있음_

요구사항이 개발환경은 기존 계정에 그대로 있어야 된다는 것이었기에 도메인의 네임 서버 값을 바꾸는 것만으로는 부족했습니다.

물론 새로 생성한 NEW hosted zone에도 개발환경의 레코드를 추가한다면 해결될 일이지만 개발환경의 IaC에서 프로덕션의 리소스를 수정하는 것은 바람직하지 않다고 생각되어 다른 방법을 생각하게 되었습니다.

그렇게 고민하다가 나온 방법이 Sub Delegation을 이용한 방법으로 아래의 다이어그램처럼 `dev.detectivecrow.co.kr`의 hosted zone을 생성한 다음, 각 hosted zone에 추가하는 것으로 개발환경 또한 다운타임 없이 마이그레이션이 가능해 집니다.

![Problem 002](/posts/20221120/problem_002.png)
_Sub Delegation을 이용하여 두 hosted zone에서 바라보게 함_

## 만약 데이터 정합성을 고려한다면

처음에 저의 경우, 데이터 정합성은 백엔드에서 처리해주기로 했다고 했는데요, 여기에 추가적으로 

# 정리

이러한 마이그레이션 작업은 전체적인 흐름을 이해하면 생각보다 엄청 간단한 내용일 수 있습니다.
하지만 인프라의 변경은 언제나 예측하지 못한 문제가 야기 될 수 있으므로 조심해서 나쁠 것은 없다고 생각합니다.

제가 경험했던 부분들이 도움이 되었으면 하며 즐거운 데브옵스 되세요! :)

# 참고

- **[Certified Kubernetes Administrator (CKA) with Practice Tests]** [https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/](https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/)
- **[Kubernetes-Architecture-Diagrams]** [https://phoenixnap.com/kb/understanding-kubernetes-architecture-diagrams](https://phoenixnap.com/kb/understanding-kubernetes-architecture-diagrams)
- **[Kubernetes Components]** [https://kubernetes.io/ko/docs/concepts/overview/components/](https://kubernetes.io/ko/docs/concepts/overview/components/)
