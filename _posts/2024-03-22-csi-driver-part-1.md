---
title: "[Kubernetes Deep Dive] CSI Driver - Part 1"
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
  - csi
  - csi-driver
---
---
# 개요

Kubernetes 클러스터를 생성하고 Helm 차트를 통해 다양한 애플리케이션을 설정하다 보면 볼륨에 대한 설정이 항상 등장합니다.
이러한 상황에서 볼륨에 대해서 자세히 모르다 보면 이 부분에서 헤매는 경우가 종종 발생하게 되고 급하게 많이 사용하는 csi-driver 나 longhorn 등을 설치해서 문제를 해결하곤 합니다.

그런데 여기서 csi-driver는 실제로 어떻게 구성되고 동작할까요? 우리는 보통 설치하고 문제없이 동작하는 것을 확인하지만 내부에서 어떤 방식을 이용해서 동작하는지는 문제가 생겼을 때 확인해 보곤 합니다.
좀 더 자세히 공부해 보자는 의미에서 본 글에선 여기서 CSI는 무엇이고 어떤 구성요소들이 있는지에 대해서 알아보도록 하겠습니다.

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

# CSI Driver 메커니즘

먼저 전반적으로 CSI 드라이버들이 어떻게 동작하는지 알기 위해서 Kubernetes에서 권장하는 메커니즘에 대해서 설명하도록 하겠습니다.

![container-storage-interface_diagram](posts/20240322/container-storage-interface_diagram.png)

Kubernetes는 3rd party 벤더들이 CSI 볼륨 드라이버를 만들 때 다음과 같은 사항을 따르는 것을 권장합니다.
- 볼륨 플러그인 동작을 구현하고 CSI 사양(컨트롤러, 노드 및 신원 서비스 포함)에 정의된 대로 유닉스 도메인 소켓을 통해 gRPC 인터페이스를 노출하는 "<ins>CSI volume driver</ins>" 컨테이너를 만듭니다.
- "<ins>CSI volume driver</ins>"에 Kubernetes 팀이 지원할 헬퍼 컨테이너를 번들합니다. (external-attacher, external-provisioner, node-driver-registrar, cluster-driver-registrar, external-resizer, external-snapshotter, livenessprobe).
  이러한 헬퍼 컨테이너는 드라이버가 Kubernetes 시스템과 상호작용 하는 것을 돕습니다. 이러한 헬퍼 컨테이너에 대한 자세한 내용은 아래에서 설명하도록 하겠습니다.
- 클러스터 관리자가 위 다이어그램의 StatefulSet과 DaemonSet을 배포하여 클러스터에 스토리지 시스템에 대한 지원을 추가하도록 합니다.

> 여기서 모든 구성 요소(`external-provisioner` 와 `external-attacher` 를 포함한)를 단일 Pod에 배치하여 배포를 단순화할 수도 있습니다.
> 하지만 이렇게 구성할 경우, 더 많은 리소스가 소모되고 `external-provisioner` 및 `external-attacher` 구성 요소에 리더 선출 프로토콜(예 : https://github.com/kubernetes-retired/contrib/tree/master/election)이 필요합니다.

# Components of CSI Driver

## Controller Component

컨트롤러 플러그인은 `Deployment` 또는 `StatefulSet`으로 배포되며 클러스터 내의 모든 노드에 마운트 할 수 있습니다. 이는 CSI Controller 서비스를 구현하는 CSI 드라이버와 사이드카 컨테이너로 구성됩니다.
컨트롤러 사이드카 컨테이너는 일반적으로 쿠버네티스 오브젝트와 상호 작용하고 CSI Controller 서비스를 호출합니다.

컨트롤러는 호스트에 직접 액세스 할 필요가 없습니다.(외부 컨트롤 플레인 서비스와 쿠버네티스 API를 통해 모든 작업을 수행할 수 있습니다.)
고가용성(HA)을 위해 컨트롤러 구성 요소의 복사본을 여러 개 배포할 수 있지만, 리더 선출을 구현하여 특정 시간에 하나의 컨트롤러만 활성화되도록 해야 합니다.
### Sidecar Containers

CSI 사이드카 컨테이너는 쿠버네티스에서 CSI 드라이버의 개발 및 배포를 간소화하는 것을 목표로 하는 표준 컨테이너 세트입니다.
이러한 컨테이너에는 Kubernetes API를 감시하고, "CSI 볼륨 드라이버" 컨테이너에 대해 적절한 작업을 트리거하고, 적절하게 Kubernetes API를 업데이트하는 공통 로직이 포함되어 있습니다.

여기서 설명하는 사이드카 컨테이너는 다음과 같은 컨테이너들이 포함되어 있습니다.
- [external-provisioner](https://kubernetes-csi.github.io/docs/external-provisioner.html)
- [external-attacher](https://kubernetes-csi.github.io/docs/external-attacher.html)
- [external-snapshotter](https://kubernetes-csi.github.io/docs/external-snapshotter.html)
- [external-resizer](https://kubernetes-csi.github.io/docs/external-resizer.html)
- [node-driver-registrar](https://kubernetes-csi.github.io/docs/node-driver-registrar.html)
- [cluster-driver-registrar](https://kubernetes-csi.github.io/docs/cluster-driver-registrar.html) (deprecated)
- [livenessprobe](https://kubernetes-csi.github.io/docs/livenessprobe.html)

아래는 추가적인 설정 없이 `ebs-csi-driver`를 설치했을 때 Deployment로 생성된 `ebs-csi-controller`의 내용입니다.
위의 provisioner, attacher, snapshotter, resizer 및 liveness-probe가 포함되어 있습니다.

```bash
> k get po -n kube-system ebs-csi-controller-67bbddfc9-6f8vt -o yaml | yq e '.spec.containers.[].name'

ebs-plugin
csi-provisioner
csi-attacher
csi-snapshotter
csi-resizer
liveness-probe
```

## Per-node Component

노드 플러그인은 DaemonSet을 통해 클러스터의 모든 노드에 배포해야 합니다. 이는 CSI Node Service를 구현하는 CSI Driver와 node-driver-registrar 역할을 하는 사이드카 컨테이너로 구성됩니다.

이 컴포넌트는 모든 노드에서 실행되며 `kubelet`과 통신하며 CSI Node service에 대한 요청을 처리합니다. 이러한 호출은 스토리지 시스템에서 스토리지 볼륨을 Mount 혹은 Unmount 하며 그것들을 Pod가 이용할 수 있도록 해줍니다.
`kubelet`은 호스트의 HostPath 볼륨을 통해 공유되는 UNIX domain socket을 사용하여 CSI driver를 호출합니다. 추가적인 UNIX domain socket은 node-driver-registrar이 드라이버를 kubelet에 등록하는 데 사용됩니다.

아래는 `ebs-csi-driver`를 설치했을 때 DaemonSet으로 생성된 `ebs-csi-node`의 내용입니다.
위의 설명처럼 `node-driver-registrar`에 HostPath 볼륨이 연결되어 있으며 `csi.sock`를 사용하는 것을 확인할 수 있습니다.
```bash
> k describe po -n kube-system ebs-csi-node

Volumes:
  kubelet-dir:
    Type:          HostPath (bare host directory volume)
    Path:          /var/lib/kubelet
    HostPathType:  Directory
  plugin-dir:
    Type:          HostPath (bare host directory volume)
    Path:          /var/lib/kubelet/plugins/ebs.csi.aws.com/
    HostPathType:  DirectoryOrCreate
  registration-dir:
    Type:          HostPath (bare host directory volume)
    Path:          /var/lib/kubelet/plugins_registry/
    HostPathType:  Directory
  device-dir:
    Type:          HostPath (bare host directory volume)
    Path:          /dev
    HostPathType:  Directory
  probe-dir:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     
    SizeLimit:  <unset>
...
Containers:
  ...
  node-driver-registrar:
    ...
    Requests:
      cpu:     10m
      memory:  40Mi
    Liveness:  exec [/csi-node-driver-registrar --kubelet-registration-path=$(DRIVER_REG_SOCK_PATH) --mode=kubelet-registration-probe] delay=30s timeout=15s period=90s #success=1 #failure=3
    Environment:
      ADDRESS:               /csi/csi.sock
      DRIVER_REG_SOCK_PATH:  /var/lib/kubelet/plugins/ebs.csi.aws.com/csi.sock
    Mounts:
      /csi from plugin-dir (rw)
      /registration from registration-dir (rw)
      /var/lib/kubelet/plugins/ebs.csi.aws.com/ from probe-dir (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-kz76w (ro)
```

노드 플러그인은 드라이버 볼륨을 마운트 하기 위해 호스트에 직접 액세스해야 합니다. 파일 시스템 마운트와 Block Device를 kubelet에서 사용할 수 있게 하려면, CSI 드라이버는 드라이버 컨테이너가 생성한 마운트를 kubelet이 볼 수 있도록 하는 양방향 마운트 포인트를 사용해야 합니다.

> Unix Domain Socket(UDS)란 IPC socket(inter-process communication socket) 이라고도 하며 TCP의 소켓과 동일한 API로 데이터를 주고받을 수 있는 local file 기반의 소켓입니다.
> 더 자세한 내용은 Reference의 [유닉스 도메인 소켓(Unix Domain Socket) 이란?]을 참고해 주세요. 

# 마치며

이번 글에서는 시간 관계상 CSI 가 무엇인지, CSI Driver는 어떤 구성 요소를 가지고 있는지에 대해서 간단하게 알아보았습니다.

다음에 기회가 된다면 좀 더 자세한 CSI Driver가 어떻게 동작하는지, 어떠한 과정을 거쳐서 볼륨을 관리하는지에 대해서 알아보도록 하겠습니다. 

# Reference
- [Basics of csi volumes and how to build a csi driver](https://bluexp.netapp.com/blog/cvo-blg-kubernetes-csi-basics-of-csi-volumes-and-how-to-build-a-csi-driver)
- [CSI Volume Plugins in Kubernetes Design Doc](https://github.com/kubernetes/design-proposals-archive/blob/main/storage/container-storage-interface.md#cluster-level-deployment)
- [Kubernetes CSI Sidecar Containers](https://kubernetes-csi.github.io/docs/sidecar-containers.html)
- [KubernetesにおけるContainer Storage Interface (CSI)の概要と検証](https://qiita.com/ysakashita/items/4b56c2577f67f1b141e5)
- [유닉스 도메인 소켓(Unix Domain Socket) 이란?](https://www.lesstif.com/linux-core/unix-domain-socket)
