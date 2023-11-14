---
title: ! 'Git 브랜치 전략 (git workflow)'
author: cawcaw253
date: 2022-07-26 20:34:00 +0900
categories: [Git]
tags: [git]
---

---
# Git 브랜치 전략이란? (git workflow)

## 간단한 설명

Vincent Driessen 이라는 사람이 고안한 git을 이용한 버전 관리의 방법론입니다.
여기에서 버전 관리의 방법론이란, 특정 브랜치를 만들어 역할을 부여하고 규칙을 만듦으로써 프로젝트의 관리를 체계화하는 방법을 의미합니다.

## 방법론의 종류

우선 크게 유명한 3가지 템플릿이 존재합니다.
각각 Git Flow, GitHub Flow, GitLab Flow 으로 불리며 이것은 반드시 이렇게 해라 라는 것보다는 Vincent Driessen가 말한 것처럼 개발 환경에 따라 수정, 변형해서 사용하라고 한 것처럼 참고 정도로 하는 것이 좋습니다.

# Git Flow

## 간단한 설명

먼저 기본적인 Git Flow입니다.

![git flow](posts/20220726/git-flow.png)

Git Flow 는 아래와 같이 5개의 브랜치로 이루어져 있습니다.

- **master** : 중심이 되는 브랜치로 제품을 배포하는 브랜치입니다.
- **develop** : 개발 브랜치로 프로젝트의 구성원들이 이 브랜치를 기준으로 각자 작업한 기능들을 Merge 합니다.
- **feature** : 단위 기능을 개발하는 브랜치로 개발이 완료되면 develop 브랜치에 Merge 합니다.
- **release** : 배포를 위해 master 브랜치로 merge 하기 전에 테스트를 하기 위한 브랜치입니다.
- **hotfix** : master 브랜치에 배포된 제품에 버그가 발생했을 때 긴급 수정하는 브랜치입니다.

## 장/단점

### 장점

- 명령어가 직접적으로 나와있으며 브랜치 이름이 직관적입니다.
- 가장 기본적이고 널리 알려져 있기에 다양한 에디터와 IDE에서 플러그인으로 존재합니다.

### 단점

- 브랜치가 많아서 복잡한 구성을 갖고 있습니다.
- 프로젝트에 따라서 사용하지 않는 브랜치가 존재할 수 있습니다.

# GitHub Flow

## 간단한 설명

GitHub Flow는 **Scott chacon**가 Git Flow 는 좋은 방식이지만 GitHub 에서는 사용하기에 복잡하다고 하며 소개한 방식입니다.
룰이 굉장히 단순하며 `master` 브랜치에 대한 각 브랜치의 `role` 만 정확하다면 다른 브랜치에는 관여를 하지 않습니다.
그리고 `pull request` 기능을 사용하도록 권장을 합니다.

![github flow](posts/20220726/github-flow.png)

## 장/단점

### 장점

- 브랜치 전략이 단순합니다.
- 방법이 단순하기에 처음 접하는 사람이 사용하기 쉽습니다.
- 자연스럽게 코드 리뷰를 사용할 수 있습니다.

### 단점

- 명확한 룰이 없기 때문에 그룹 내에서 명확하게 룰을 정할 필요성이 있습니다.
- 룰이 존재하지 않거나 없으면 시간이 지날수록 버전 관리의 복잡도가 올라가며 효율이 떨어지게 됩니다.

# GitLab Flow

## 간단한 설명

`GitHub Flow` 에서는 flow가 너무 간단하여 각 작업 (배포, 테스트 등)에 대한 이슈를 남겨둔 것이 많았습니다.
그것을 보안하기 위해 `GitLab Flow` 에서는 관련 내용들을 추가적으로 덧붙여 설명하였습니다.

브랜치는 `GitHub Flow` 를 베이스로 `production` 브랜치가 추가되어 커밋된 내용들을 배포하는 형태를 가지고 있습니다.
또한 별도로 `development` 브랜치를 통한 테스트 환경 작성, `pre-production` 혹은 `staging` 브랜치를 통한 스테이징 환경을 만드는 것도 권장됩니다.

![gitlab flow](posts/20220726/gitlab-flow.png)

# 마치며

전체적으로 간단하게 git workflow 를 정리해보았습니다.
위의 방법론은 처음에 설명한 것처럼 반드시 이렇게 해야 된다 라기보다는 자신, 혹은 팀의 작업에 따라서 취사선택을 하여 이용하는 것이 가장 좋으며, 전체적인 효율이 올라간다고 생각됩니다.
요약을 한 부분들이 많아 부족하지만 도움이 되었으면 좋겠습니다.

# 참고자료

- [https://blog.programster.org/git-workflows](https://blog.programster.org/git-workflows)
- [https://ujuc.github.io/2015/12/16/git-flow-github-flow-gitlab-flow](https://ujuc.github.io/2015/12/16/git-flow-github-flow-gitlab-flow)
- [https://shoma2da.hatenablog.com/entry/2015/11/04/233534](https://shoma2da.hatenablog.com/entry/2015/11/04/233534)