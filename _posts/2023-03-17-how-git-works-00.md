---
title: ! 'git은 어떻게 동작할까? | git 톺아보기'
author: cawcaw253
date: 2023-03-17 18:43:00 +0900
categories: [git]
tags: [git, advanced]
published: false
---

---
# 어쩌다 이런글을...

어느날 회사에서 교육 사이트인 https://app.pluralsight.com 를 이용해 보자고 얘기가 나와서 해당 사이트에서 온라인 강의를 들어볼 기회가 생겼습니다.
처음엔 Hands On 관련된 강의가 많아 보여서 이론적인 것들을 좋아하는 저에게는 잘 맞지 않는다는 생각을 했었습니다.
그러다가 우연히 `How Git Works`(https://app.pluralsight.com/library/courses/how-git-works/table-of-contents)라는 강의를 스쳐 지나가듯 보게 되었고, 문득 나는 git이 뭔지 잘 알고 사용하고 있었나 라는 생각이 들었습니다...

생각해보면 아마도 대부분의 사람들이 맨날 사용하는 커맨드들 `git checkout`, `git merge`, `git rebase`등이 실제로 어떻게 동작하는지 전혀 모른체 사용하다가 문제가 생기면 구글링하여 커맨드를 그저 복사, 붙여넣기해 해결하고 있지 않을까 싶습니다.

저 또한 그러한 사람들 중 하나였고요. 이런 생각이 스쳐 지나가고나니 궁금증이 생겨 해당 강의를 듣게 되었고, 강의 내용이 너무나도 좋다고 생각되어 `git은 어떻게 동작할까?` 라는 주제로 글을 몇 개 써보고자 마음을 먹게 되었습니다.

혹시라도 위의 내용을 읽으면서 뜨끔 하셨던 분들은 대충이라도 괜찮으니 간단하게 읽어봐 주시면 감사할것 같습니다 :)

# 개요

Porcelain Commands

- git add
- git commit
- git push
- git pull
- git branch
- git switch
- git merge
- git rebase
- ...

Plumbing Commands

- git cat-file
- git hash-object
- git count-object
- ...

깃을 마스터 하고싶다면 커맨드를 공부하지마라, 모델을 공부해라!!

# git의 기능을 나눠서 하나씩 알아가 봅시다.

깃을 레이어라고 생각하고 단순화하여 이해를 도와보겠습니다.

1. Git Is... a Distributed Revision Control System
좀 더 간단하게 분산된 시스템에서의 관리가 아니라 한 대의 컴퓨터에서만 동작한다고 생각해 봅시다.
2. a Revision Control System
여전히 복잡합니다... 개정 관리 시스템은 History, branch, merge 등의 다양한 기능들이 존재합니다.
조금 더 단순화 해보면 어떨까요?
3. a Stupid Content Tracker
훨씬 간단해졌습니다. 깃이 실제로 하는 일은 결국 콘텐츠를 추적하는 것입니다. 파일이나 폴더등을 말이죠.
실제로 `man git` 커맨드로 Git의 매뉴얼을 본다면 Git이 자신을 설명하고 있는 문구입니다.
여기서 트래킹, 커밋의 개념, 버저닝을 신경쓰지 말고 한번 더 단순화해서 보도록 해봅시다.
4. a Persistent Map
Git의 정말 기본적이고 코어한 부분을 본다면 Persistent Map 이라고 볼 수 있습니다.
즉 Git은 디스크에 저장되는 단순한 Key와 Value의 Map 입니다.





======================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================================

How Git Works

Porcelain Commands

- git add
- git commit
- git push
- git pull
- git branch
- git switch
- git merge
- git rebase
- ...

Plumbing Commands

- git cat-file
- git hash-object
- git count-object
- ...

깃을 마스터 하고싶다면 커맨드를 공부하지마라, 모델을 공부해라!!

========================

깃을 레이어라고 생각하고 단순화하여 이해를 도와보겠습니다.

1. Git Is... a Distributed Revision Control System
좀 더 간단하게 분산된 시스템에서의 관리가 아니라 한 대의 컴퓨터에서만 동작한다고 생각해 봅시다.
2. a Revision Control System
여전히 복잡합니다... 개정 관리 시스템은 History, branch, merge 등의 다양한 기능들이 존재합니다.
조금 더 단순화 해보면 어떨까요?
3. a Stupid Content Tracker
훨씬 간단해졌습니다. 깃이 실제로 하는 일은 결국 콘텐츠를 추적하는 것입니다. 파일이나 폴더등을 말이죠.
실제로 `man git` 커맨드로 Git의 매뉴얼을 본다면 Git이 자신을 설명하고 있는 문구입니다.
여기서 트래킹, 커밋의 개념, 버저닝을 신경쓰지 말고 한번 더 단순화해서 보도록 해봅시다.
4. a Persistent Map
Git의 정말 기본적이고 코어한 부분을 본다면 Persistent Map 이라고 볼 수 있습니다.
즉 Git은 디스크에 저장되는 단순한 Key와 Value의 Map 입니다.

===========================

a Persistent Map

```
	- Values and Keys
		- Values : Any sequence of bytes
		- Keys : SHA1 hash

		"Apple Pie" -> "23991897e13e47ed0adb91a0082c31c82fe0cbe5"

		echo "Apple Pie" | git hash-object --stdin 명령어로 확인할 수 있습니다.

	- Every object in Git has its own SHA1.
	So, what if they collide?

	확률이 굉장히 낮으므로 무시해도 괜찮습니다.

```

Persistant Map 이란?

위의 내용을 통해 우리는 Git이 Key-Value로 정보를 트래킹하고 있다는 것을 알았습니다.
그런데 여기서 Persistant가 의미하는 건 무엇일까요?

아까 위에서 이용했던 커맨드를 조금 더 자세히 살펴보도록 하겠습니다.

echo "Apple Pie" | git hash-object --stdin -w 로 쓰기 옵션을 주도록 하겠습니다.

```
~/lhs/git-test   main ❯ echo "Apple Pie" | git hash-object --stdin -w                                                 base
23991897e13e47ed0adb91a0082c31c82fe0cbe5

```

그런뒤 .git/objects로 이동하면 23으로 시작하는 폴더를 확인할 수 있습니다.

```
~/lhs/git-test   main ❯ cd .git/objects                                                                               base
~/lhs/git-test/.git/objects   main ❯ ls -la                                                                           base

total 20
drwxr-xr-x 5 lhs lhs 4096 Mar  2 13:43 .
drwxr-xr-x 7 lhs lhs 4096 Mar  2 13:42 ..
drwxr-xr-x 2 lhs lhs 4096 Mar  2 13:43 23
drwxr-xr-x 2 lhs lhs 4096 Mar  2 13:42 info
drwxr-xr-x 2 lhs lhs 4096 Mar  2 13:42 pack

~/lhs/git-test/.git/objects   main ❯

```

여기서 23 폴더의 내용물을 살펴보면 낯익은 문자열을 발견할 수 있을겁니다.

```
  ~/lhs/git-test/.git/objects/23   main ❯ ls -la                                                                        base
total 12
drwxr-xr-x 2 lhs lhs 4096 Mar  2 13:43 .
drwxr-xr-x 5 lhs lhs 4096 Mar  2 13:43 ..
-r--r--r-- 1 lhs lhs   26 Mar  2 13:48 991897e13e47ed0adb91a0082c31c82fe0cbe5
  ~/lhs/git-test/.git/objects/23   main ❯

```

잘 관찰해보면 각각 23과 991897e13e47ed0adb91a0082c31c82fe0cbe5 은 아까 hash-object 커맨드를 통해 만들어진 hash값이라는걸 알 수 있습니다.
즉 Git은 Hash된 key값의 앞 두 16진수 값을 폴더로 만들고 나머지 값을 파일 이름으로 지정하여 파일을 구조화하고 분산시킵니다.
이를 통해 한 디렉토리에 파일이 몰리지 않도록 관리하고 blob 형식으로 정보를 저장시킵니다.
이런 blob 데이터는 파일에 직접 접근하여 볼 수 없습니다만 Plumbing Commands를 이용하면 확인 할 수 있습니다.

```
# type 확인
git cat-file 23991897e13e47ed0adb91a0082c31c82fe0cbe5 -t
# print
git cat-file 23991897e13e47ed0adb91a0082c31c82fe0cbe5 -p

```

===========================

a Stupid Content Tracker

  ~/lhs/git-test   main +3 ❯ git st                                                                                     base
On branch main

No commits yet

Changes to be committed:
(use "git rm --cached <file>..." to unstage)
new file:   menu.txt
new file:   recipes/README.txt
new file:   recipes/apple_pie.txt

  ~/lhs/git-test   main +3 ❯ git config user.email "[cawcaw253@gmail.com](mailto:cawcaw253@gmail.com)""                                       base
dquote>
  ~/lhs/git-test   main +3 ❯ git config user.email "[cawcaw253@gmail.com](mailto:cawcaw253@gmail.com)"                                 ✘ INT  base
  ~/lhs/git-test   main +3 ❯ git config [user.name](http://user.name/) "cawcaw253"                                                   base
  ~/lhs/git-test   main +3 ❯ git commit -m "First commit"                                                               base
[main (root-commit) c0ccc7d] First commit
3 files changed, 3 insertions(+)
create mode 100644 menu.txt
create mode 100644 recipes/README.txt
create mode 100644 recipes/apple_pie.txt
  ~/lhs/git-test   main ❯ git st                                                                                        base
On branch main
nothing to commit, working tree clean
  ~/lhs/git-test   main ❯ git log                                                                                       base
  ~/lhs/git-test   main ❯ explorer.exe .git                                                                       14s  base
  ~/lhs/git-test   main ❯ git log                                                                                       base
  ~/lhs/git-test   main ❯ git cat-file -p c0ccc7d82f06884c4ddcc54220c2a3cf8e13c703                                      base
tree 887ac3823f18a2b27c56dcff64fb7cce662da5fc
author cawcaw253 [cawcaw253@gmail.com](mailto:cawcaw253@gmail.com) 1677733681 +0900
committer cawcaw253 [cawcaw253@gmail.com](mailto:cawcaw253@gmail.com) 1677733681 +0900

First commit
  ~/lhs/git-test   main ❯ git cat-file -p 887ac3823f18a2b27c56dcff64fb7cce662da5fc                                      base
100644 blob 23991897e13e47ed0adb91a0082c31c82fe0cbe5    menu.txt
040000 tree 29b3104586f4eb8d4e0b5e445e45c2bdf5030022    recipes
  ~/lhs/git-test   main ❯ git cat-file -p 23991897e13e47ed0adb91a0082c31c82fe0cbe5                                      base
Apple Pie
  ~/lhs/git-test   main ❯ git cat-file -p 29b3104586f4eb8d4e0b5e445e45c2bdf5030022                                      base
100644 blob 67b3562eb527a95ee72c62feb90556698511ff61    README.txt
100644 blob 23991897e13e47ed0adb91a0082c31c82fe0cbe5    apple_pie.txt
  ~/lhs/git-test   main ❯ git cat-file -p 67b3562eb527a95ee72c62feb90556698511ff61                                      base
Put your recipes in this directory, one recipe per file
  ~/lhs/git-test   main ❯

---

---

# Branches

### Branch

이번엔 한 단계 위인  a Revision Control System 그 중에서도 Branches에 대해서 이해해보도록 하겠습니다.

git branch 명령어를 실행해보면 기본적으로 사용하는 main 브랜치가 존재합니다.

```bash
> git log
commit d13ae2d03b3c3e49fa043b2e7ec0935c7335918a (HEAD -> main)
Author: cawcaw253 <cawcaw253@gmail.com>
Date:   Thu Mar 2 14:59:36 2023 +0900

    Add more cake

commit c0ccc7d82f06884c4ddcc54220c2a3cf8e13c703
Author: cawcaw253 <cawcaw253@gmail.com>
Date:   Thu Mar 2 14:08:01 2023 +0900

    First commit

> git branch
* main

```

이러한 브랜치에 대한 정보는 .git/refs/heads 아래에 존재하게 됩니다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c4026da4-7990-426a-b8e5-f68507640047/Untitled.png)

이 main 파일 안에는 어떤 정보가 존재할까요? 아마도 여러분들이 상상한 정보가 포함되어 있을것입니다.

```bash
> cat .git/refs/heads/main                                                                      base
d13ae2d03b3c3e49fa043b2e7ec0935c7335918a
```

파일은 한줄의 hash 값이 존재합니다. 그렇습니다 이 hash값은 위의 git log에서 확인한 commit의 hash 값입니다.

**즉 branch란 단순히 commit의 참조(just a reference to a commit) 입니다.**

** 그러면 .git/refs/heads 디렉토리 밑에 파일의 이름을 바꾸거나 추가하면 브랜치가 수정, 생성될까요? → 그렇습니다! 단순히 파일을 만들고 참조할 commit의 hash값을 넣어주면 해당 commit을 참조하는 브랜치가 생성됩니다!

```bash
> echo c0ccc7d82f06884c4ddcc54220c2a3cf8e13c703 > .git/refs/heads/test                          base
> ls .git/refs/heads                                                                            base
main  test
> git branch
* main
  test
```

### HEAD

이제 branch의 정보가 어디에 있는지 알게 되었습니다. 그런데 git은 현재 작업하는 branch가 어느 branch인지 어떻게 알고 있을까요?

git을 사용해 보셨던 분들은 아마 알고 계실텐데 바로 `.git/HEAD` 에 적혀있습니다.

```bash
> cat .git/HEAD                                                                                 base
ref: refs/heads/main
```

HEAD는 파일의 위치를 참조하고 있습니다.

**즉 HEAD는 branch의 참조(just a reference to a branch)입니다.**

### Change branch

branch가 무엇인지, 어떻게 생성하는지 그리고 HEAD가 무엇인지 확인해 보았습니다.

그렇다면 우리가 자주 사용하는, 사용하게 될 branch를 변경하는 작업은 어떠한 과정을 거치는지 상상해 볼 수 있을것입니다.

우선 branch를 변경하는 코드에는 두 가지가 존재합니다.

- `git switch test`
- `git checkout test`

**둘의 차이는 무엇일까요?**

두 command는 각자 다른 옵션이 존재하고 서로 다른 상황에서 자주 사용됩니다.

하지만 우리가 간단하게 branch를 바꾸는 작업만 한다면 둘은 동일한 동작을 합니다.

다만 한 가지 다른 점이 존재한다면, `git switch` 커맨드가 상대적으로 최근에 등장했다는 점입니다.

따라서 git의 버전에 따라서 `git switch` 커맨드가 동작하지 않는다면, `git checkout` 을 사용하면 됩니다.

정리해서 말하자면 **Branch의 변경은**

switch는 정말로 단순한 동작을 합니다.

1. HEAD를 옮기고
2. working area를 업데이트 합니다.

```bash
# 커맨드를 통해서 실제로 HEAD 파일이 바뀐 것을 확인할 수 있습니다.
main ❯ cat .git/HEAD
ref: refs/heads/main
main ❯ git switch test
Switched to branch 'test'
test ❯ cat .git/HEAD
ref: refs/heads/test
```

### Merge

main, test branch에 각각 새로운 commit을 생성해 보겠습니다. 

```bash
동일한 파일을 수정
```

동일한 파일을 수정한 commit이 있고, 그 commit을 가리키는 두 개의 branch가 존재하고 있습니다.

여기서 test 브랜치에 있는 변경 사항을 main 브랜치에 merge시켜보도록 하겠습니다.

```bash
both modified 표시 
```

저희가 자주 보는 git 혹은 다른 버전 관리 시스템의 메시지가 표시되었습니다.

해당 파일을 열어보면 현재의 변경사항과 merge하려 시도한 branch의 변경사항을 확인 할 수 있습니다.

```bash
<<<<<<< HEAD
test1
=======
test2
>>>>>>> test
```

이 confilict 를 해결하고 스테이징에 추가하고 commit 해보겠습니다.

```bash

```

commit의 key를 가지고 내용을 한 번 볼까요?

```bash
# parent가 두개인 것을 확인 가능
parent ***
parent ***
```

parent 가 두 개인것을 확인 할 수 있습니다.

즉 Merge를 간단하게 설명하자면

두 commit을 parent로 가진 commit을 생성하고 HEAD를 새로 생성한 commit으로 옮기는 것이라고 할 수 있습니다.