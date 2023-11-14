---
title: ! 'Terraform vs Ansible 무엇을 사용할까?'
author: cawcaw253
date: 2022-09-11 21:38:00 +0900
categories: [DevOps, Infrastructure as code]
tags: [devops, infrastructure as code]
---

---
# Terraform과 Ansible 무엇이 더 좋을까?

베어메탈, 클라우드, VM 등의 다양한 환경에서 쿠버네티스 환경 구축을 자동화하는 방법을 고민하던 중, Terraform, Ansible 등 다양한 Iac가 눈에 들어와 조금씩 조사해 보게 되었습니다.
이 글에서는 위의 툴 중, 대표적으로 비교되는 Terraform과 Ansible이 각자 어떠한 특징이 있는지, 또 어떤 작업에 무엇이 더 적절한지 파악해보고자 합니다.

# 특징

## Terraform

> HashiCorp Terraform은 버전을 지정하고 재사용하고 공유할 수 있는 사람이 읽을 수 있는 구성 파일에서 클라우드 및 온프레미스 리소스를 모두 정의할 수 있는 코드형 인프라 도구입니다. 그런 다음 일관된 워크플로를 사용하여 수명 주기 동안 모든 인프라를 프로비저닝하고 관리할 수 있습니다. Terraform은 컴퓨팅, 스토리지 및 네트워킹 리소스와 같은 하위 수준 구성 요소는 물론 DNS 항목 및 SaaS 기능과 같은 상위 수준 구성 요소를 관리할 수 있습니다.
> 
> 
> refer to : [https://www.terraform.io/intro](https://www.terraform.io/intro)
> 

기본적으로 Terraform은 클라우드 기반이며 HCL이라고 하는 HashiCorp에서 제작한 언어로 작성합니다. 다양한 서비스 제공업체 및 시스템의 API를 호출하여 작동하며 따라서 API 가 존재하는 온프레미스 환경 또한 지원합니다. (OpenStack, VMWare vSphere, CloudStack)

Terraform은 Infrastructure Orchestration tool입니다. 비유를 들어 설명하자면 오케스트라의 악기의 수, 배치 등의 전체적인 틀을 잡아주는 툴이며 원하는 결과에 중점을 두어 환경이 특정 상태로 유지되도록 하는 데에 특화되어 있습니다.

## Ansible

> Ansible®은 오픈소스 IT 자동화 툴로서, 프로비저닝, 구성 관리, 애플리케이션 배포, 오케스트레이션, 기타 여러 가지 수동 IT 프로세스를 자동화합니다. 더 단순한 관리 툴과 달리 Ansible 사용자(예: 시스템 관리자, 개발자, 아키텍트)는 Ansible 자동화를 사용해 소프트웨어를 설치하고, 일상적인 태스크를 자동화하고, 인프라를 프로비저닝하고, 보안 및 컴플라이언스를 개선하고, 시스템에 패치를 적용하고, 조직 전체에 자동화를 공유할 수 있습니다.
> 
> 
> refer to : [https://www.redhat.com/ko/technologies/management/ansible/what-is-ansible](https://www.redhat.com/ko/technologies/management/ansible/what-is-ansible)
> 

Ansible은 Yaml 파일로 작성하고 Infrastructure와 app을 자동화하고 구성하는 툴입니다.

Terraform과 비교하자면 Infrastructure의 구성보다는 각 구성에 따라 다르게 동작해야 하는 configuration management tool에 가깝습니다. 비유하자면 오케스트라에서 사용하는 수십개가 넘는 각 악기의 미세 조정을 자동으로 해주는 툴입니다.

# 비교

|         | Terraform                | Ansible                       |
|:--------|:-------------------------|------------------------------:|
| 형태     | Orchestration tool       | Configuration management tool |
| 접근 형태 | Immutable Infrastructure | Mutable Infrastructure        |
| 언어     | Declarative(선언적)        | Procedural(순차적)             |

## 유사점

Terraform과 Ansible은 모두 생성된 가상 머신에서 원격으로 명령을 실행할 수 있습니다.
즉, 두 도구 모두 에이전트가 필요 없으며 따라서 운영 목적으로 에이전트를 배포할 필요가 없습니다.

Terraform은 클라우드 공급자 API를 사용하여 인프라를 생성하고 기본 구성 작업은 SSH를 사용하여 수행합니다.
Ansible도 마찬가지입니다. SSH를 사용하여 필요한 모든 구성 작업을 수행합니다. 두 도구 모두 “상태” 정보를 관리하기 위해 별도의 인프라 세트가 필요하지 않으므로 마스터리스라 할 수 있습니다.

## 차이점

오케스트레이션 대 구성 관리
: 오케스트레이션/프로비저닝은 가상 머신, 네트워크 구성, 데이터베이스 등의 인프라를 만드는 프로세스입니다. 반면에 구성 관리는 버전이 지정된 소프트웨어 구성 요소 설치, OS 구성 작업, 네트워크 및 방화벽 구성 등을 자동화하는 프로세스입니다.

Terraform과 Ansible은 두 가지 작업을 모두 수행할 수 있으나 각자의 강점이 존재합니다.

Terraform은 인프라 관리를 위한 포괄적인 솔루션을 제공하며, 클라우드 공급자 API를 사용하여 선언된 리소스를 기반으로 인프라를 프로비저닝 및 프로비저닝 해제합니다.

반면 Ansible은 클라우드 인프라를 프로비저닝할 수도 있지만 충분히 포괄적이지 않습니다. 주로 응용 프로그램과 종속성을 최신 상태로 유지하는 프로세스, 즉 구성 관리에 대해서 강점을 가지고 있습니다. 이러한 부분은 Ansible이 Terraform과 비교할 때 뛰어난 부분입니다.

# 정리

두 도구 모두 두 종류의 작업을 수행할 수 있습니다. 그러나 Terraform은 구성 관리를 구현하는데에 부족한 점이 존재하고, Ansible은 인프라 자동화를 구현, 복잡한 인프라를 관리하는 경우에 한계가 존재합니다.

인프라 생성을 논리적으로 나눈다면 오케스트레이션을 Day 0 활동으로, 구성 관리를 Day 1 활동으로 식별할 수 있습니다.
여기서 Terraform은 Day 0 활동에 가장 잘 작동하고 Ansible은 Day 1 이상의 활동에 가장 적합합니다.

즉, 정리하자면 대체로 Terraform과 Ansible은 모두 독립 실행형 도구로 작동하거나 함께 작동할 수 있지만 각 도구의 장점을 살려 주어진 작업에 적합한 도구를 사용한다면 효율적으로 인프라의 생성 및 관리를 할 수 있을 것입니다.

# 참고

- [https://npd-58.tistory.com/28](https://npd-58.tistory.com/28)
- [https://selleo.com/blog/terraform-vs-ansible?fbclid=IwAR0Ip2mrSo4NUME1c2004mpPqvxvHwRS5749FYVmbY4sLZ_760MahQQ7038](https://selleo.com/blog/terraform-vs-ansible?fbclid=IwAR0Ip2mrSo4NUME1c2004mpPqvxvHwRS5749FYVmbY4sLZ_760MahQQ7038)
- [https://discuss.hashicorp.com/t/terraform-plan-apply-execute-without-internet/11807](https://discuss.hashicorp.com/t/terraform-plan-apply-execute-without-internet/11807)
- [https://spacelift.io/blog/ansible-vs-terraform](https://spacelift.io/blog/ansible-vs-terraform)