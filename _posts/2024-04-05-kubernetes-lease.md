---
title: "[Kubernetes Deep Dive] Lease"
author: cawcaw253
date: 2024-04-05 20:49:00 +0900
categories:
  - Kubernetes
  - Node
  - Lease
  - Heartbeat
tags:
  - kubernetes
  - deep-dive
  - node
  - heartbeat
  - lease
---
---
# 개요

Kubelet의 로그를 보던 중 지속적으로 ControlPlane과 통신하며 Health Check를 하는 과정에 에러가 발생한 적이 있습니다. 
트러블슈팅 하면서 Node Health Check에서 Lease가 있다는 것은 알았지만 자세히는 몰랐는데, 이번 기회에 공부하게 되었습니다. 
이번 글에서는 Lease가 어떤 기능을 하는지 그리고 어떻게 삭제, 생성되는지에 대해서 정리해 보도록 하겠습니다.

# Lease의 기능

Lease는 일반적으로 분산된 시스템에서 여러 멤버가 협력하여 동작할 때 사용됩니다.
이는 공유 리소스를 lock 하고 각 노드 간의 활동을 조정하는 역할을 합니다.

쿠버네티스에서는 Node Heartbeats를 통해 Node Health를 모니터링할 때 사용하고, 또한 component 레벨의 리더 선출에 사용됩니다.
그리고 애플리케이션을 실행할 때 고가용성 설정의 단일 구성원이 요청을 서비스할 수 있도록 하기 위해 Kubernetes Lease API를 사용합니다.

아래에서 각각의 기능에 대해서 자세히 설명해 보도록 하겠습니다.
## Node Heartbeat

쿠버네티스는 Lease API를 사용하여 kubelet 노드의 하트비트를 쿠버네티스 API Server에 전달합니다.
모든 노드에는 아래와 같이 같은 이름을 가진 Lease 오브젝트가 kube-node-lease namespace에 존재합니다.

```bash
> kubectl get lease -n kube-node-lease
NAME                                               HOLDER                                             AGE
ip-10-29-104-121.ap-northeast-2.compute.internal   ip-10-29-104-121.ap-northeast-2.compute.internal   27h
ip-10-29-69-101.ap-northeast-2.compute.internal    ip-10-29-69-101.ap-northeast-2.compute.internal    27h
ip-10-29-79-205.ap-northeast-2.compute.internal    ip-10-29-79-205.ap-northeast-2.compute.internal    88m


> kubectl get lease -n kube-node-lease ip-10-29-79-205.ap-northeast-2.compute.internal -o yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  creationTimestamp: "2024-04-05T02:48:40Z"
  name: ip-10-29-79-205.ap-northeast-2.compute.internal
  namespace: kube-node-lease
  ownerReferences:
  - apiVersion: v1
    kind: Node
    name: ip-10-29-79-205.ap-northeast-2.compute.internal
    uid: 79d24559-25af-4dd8-8cf8-6b38865926b7
  resourceVersion: "289254"
  uid: 46429704-b2c8-48c9-9d13-c0096dc54fdd
spec:
  holderIdentity: ip-10-29-79-205.ap-northeast-2.compute.internal
  leaseDurationSeconds: 40
  renewTime: "2024-04-05T04:19:33.283190Z"
```

모든 kubelet 하트비트는 해당 Lease 오브젝트에 대한 업데이트를 요청하며, 이 요청은 Lease 오브젝트의 spec.renewTime 필드를 갱신합니다.

> Node에서 직접 API Server를 통해서 상태를 갱신하면 될 텐데 어째서 Leases를 이용하는 걸까요? 
> 
> Reference의 Node Heartbeats를 보면 Heartbeats는 Node의 `.status`를 갱신한다는 것을 알 수 있습니다. 
> kubectl로 node의 status 부분을 확인하면 알겠지만 이는 node에 존재하는 이미지 및 볼륨등의 수에 따라 용량이 매우 커질 수 있습니다. 
> 또한 kubelet에서 10s마다 반복되므로 노드의 수가 많다면 충분히 부담이 될 수 있는 작업입니다. 
> Leases는 Heartbeats의 비용을 줄이고 etcd의 용량을 줄이고자 하는 목적에서 도입이 되었습니다. ([Enhancements - efficient node heartbeats] 참고) 
> 
> 실제로 현재 Heartbeats 기능은 두 가지로 나뉘어 있으며 자주 반복되는 체크는 Lease가 하고 있으며 기존 `. status`의 갱신은 5분마다 체크하게 되었습니다. 
> 더 자세한 갱신 주기에 대한 설명은 Reference의 [Node Heartbeats]를 참고해 주세요

## Leader Election

서비스에 속한 리소스가 요청을 처리하도록 보장하기 위해서 사용됩니다.
쿠버네티스에서는 특정 시간 동안 <ins>Component의 인스턴스 하나만 요청을 처리하도록 보장</ins>하는 데에도 사용합니다.

![leader-election](posts/20240405/leader-election.png)

아래와 같은 Component들이 해당되며 `kubectl get lease` 명령어를 통해 확인이 가능합니다.

Components
- kube-scheduler
- kube-controller-manager
- cloud-controller-manager

```bash
> kubectl get lease -A
NAMESPACE         NAME                                               HOLDER                                                                                 AGE
kube-node-lease   ip-10-29-104-121.ap-northeast-2.compute.internal   ip-10-29-104-121.ap-northeast-2.compute.internal                                       27h
kube-node-lease   ip-10-29-69-101.ap-northeast-2.compute.internal    ip-10-29-69-101.ap-northeast-2.compute.internal                                        27h
kube-node-lease   ip-10-29-79-205.ap-northeast-2.compute.internal    ip-10-29-79-205.ap-northeast-2.compute.internal                                        88m
kube-system       apiserver-apb4cp4hnblpcjsgshanwtmkya               apiserver-apb4cp4hnblpcjsgshanwtmkya_69b4bda2-594b-426b-b127-b96f5b4a7722              27h
kube-system       apiserver-bn6uki7bzxisu7rqvlduui37nm               apiserver-bn6uki7bzxisu7rqvlduui37nm_78bcc128-939b-46e9-ace8-849910cebdbe              27h
kube-system       cloud-controller-manager                           ip-172-16-49-75.ap-northeast-2.compute.internal_e0e3cee8-5155-448f-9516-df303df178a5   27h
kube-system       cp-vpc-resource-controller                         ip-172-16-49-75.ap-northeast-2.compute.internal_42db6d33-2127-42ff-8eb6-6d61fbd3bc71   27h
kube-system       eks-certificates-controller                        ip-172-16-49-75.ap-northeast-2.compute.internal                                        27h
kube-system       kube-controller-manager                            ip-172-16-49-75.ap-northeast-2.compute.internal_25b3572b-b4f2-44ed-af75-8599e7a8465b   27h
kube-system       kube-scheduler                                     ip-172-16-49-75.ap-northeast-2.compute.internal_bbeda1bd-4c1e-4b49-b0f5-f90af015bb46   27h
```

## API 서버 인증

Kubernetes v1.26부터, 각 `kube-apiserver`는 Lease API를 이용해 자신의 identity를 시스템에 게시하게 되었습니다. 
2024-04-05 현재 특별히 유용한 기능은 없지만 Client가 `kube-apiserver`의 인스턴스 수를 파악할 수 있는 메커니즘을 제공합니다.

아래의 명령어를 통해 몇 개의 `api-server`가 동작중인지 확인할 수 있습니다.

```bash
> kubectl -n kube-system get lease -l apiserver.kubernetes.io/identity=kube-apiserver
NAME                                   HOLDER                                                                      AGE
apiserver-apb4cp4hnblpcjsgshanwtmkya   apiserver-apb4cp4hnblpcjsgshanwtmkya_69b4bda2-594b-426b-b127-b96f5b4a7722   28h
apiserver-bn6uki7bzxisu7rqvlduui37nm   apiserver-bn6uki7bzxisu7rqvlduui37nm_78bcc128-939b-46e9-ace8-849910cebdbe   28h

> kubectl get lease -n kube-system apiserver-apb4cp4hnblpcjsgshanwtmkya -o yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  creationTimestamp: "2024-04-04T00:25:30Z"
  labels:
    apiserver.kubernetes.io/identity: kube-apiserver
    kubernetes.io/hostname: ip-172-16-111-22.ap-northeast-2.compute.internal
  name: apiserver-apb4cp4hnblpcjsgshanwtmkya
  namespace: kube-system
  resourceVersion: "290326"
  uid: 0a719648-23dc-401a-8dc2-d35505ff2263
spec:
  holderIdentity: apiserver-apb4cp4hnblpcjsgshanwtmkya_69b4bda2-594b-426b-b127-b96f5b4a7722
  leaseDurationSeconds: 3600
  renewTime: "2024-04-05T04:26:25.545431Z"

> kubectl get lease -n kube-system apiserver-bn6uki7bzxisu7rqvlduui37nm -o yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  creationTimestamp: "2024-04-04T00:20:18Z"
  labels:
    apiserver.kubernetes.io/identity: kube-apiserver
    kubernetes.io/hostname: ip-172-16-49-75.ap-northeast-2.compute.internal
  name: apiserver-bn6uki7bzxisu7rqvlduui37nm
  namespace: kube-system
  resourceVersion: "290292"
  uid: 3d65bed4-8734-4c75-bc6a-a72fd048f9cb
spec:
  holderIdentity: apiserver-bn6uki7bzxisu7rqvlduui37nm_78bcc128-939b-46e9-ace8-849910cebdbe
  leaseDurationSeconds: 3600
  renewTime: "2024-04-05T04:26:11.994363Z"
```

# Practice
## Node Delete시 어떤 동작이 이루어지는가

### 1. 노드 삭제

먼저 노드를 삭제하고 실제로 내부에서 어떤 변화가 발생하는지 확인해 보겠습니다.

```bash
> kubectl delete node ip-10-29-79-205.ap-northeast-2.compute.internal
node "ip-10-29-79-205.ap-northeast-2.compute.internal" deleted

> kubectl get node
NAME                                               STATUS   ROLES    AGE   VERSION
ip-10-29-104-121.ap-northeast-2.compute.internal   Ready    <none>   29h   v1.29.0-eks-5e0fdde
ip-10-29-69-101.ap-northeast-2.compute.internal    Ready    <none>   29h   v1.29.0-eks-5e0fdde

> kubectl get lease -n kube-node-lease
NAME                                               HOLDER                                             AGE
ip-10-29-104-121.ap-northeast-2.compute.internal   ip-10-29-104-121.ap-northeast-2.compute.internal   29h
ip-10-29-69-101.ap-northeast-2.compute.internal    ip-10-29-69-101.ap-northeast-2.compute.internal    29h
```

이후 CloudWatch를 확인해 보니 `kubectl delete node` api 호출 이후에 `kube-controller-manager`가 lease, csinode, cninode 리소스에 대해 delete api를 호출하는 것을 알 수 있었습니다.

- kube-controller-manager logs
  ```bash
  I0405 07:42:06.595081      11 garbagecollector.go:549] "Processing item" item="[vpcresources.k8s.aws/v1alpha1/CNINode, namespace: , name: ip-10-29-79-205.ap-northeast-2.compute.internal, uid: e0360822-7800-42c0-b655-d99dc015e58e]" virtual=false
  I0405 07:42:06.595402      11 garbagecollector.go:549] "Processing item" item="[coordination.k8s.io/v1/Lease, namespace: kube-node-lease, name: ip-10-29-79-205.ap-northeast-2.compute.internal, uid: c0018b83-f500-4380-a845-a0ef082a0794]" virtual=false
  I0405 07:42:06.595476      11 garbagecollector.go:549] "Processing item" item="[storage.k8s.io/v1/CSINode, namespace: , name: ip-10-29-79-205.ap-northeast-2.compute.internal, uid: aec4bd2d-97e2-444a-a7b7-42b30dcd498a]" virtual=false
  I0405 07:42:06.604563      11 garbagecollector.go:688] "Deleting item" item="[vpcresources.k8s.aws/v1alpha1/CNINode, namespace: , name: ip-10-29-79-205.ap-northeast-2.compute.internal, uid: e0360822-7800-42c0-b655-d99dc015e58e]" propagationPolicy="Background"
  I0405 07:42:06.606122      11 garbagecollector.go:688] "Deleting item" item="[coordination.k8s.io/v1/Lease, namespace: kube-node-lease, name: ip-10-29-79-205.ap-northeast-2.compute.internal, uid: c0018b83-f500-4380-a845-a0ef082a0794]" propagationPolicy="Background"
  I0405 07:42:06.606288      11 garbagecollector.go:688] "Deleting item" item="[storage.k8s.io/v1/CSINode, namespace: , name: ip-10-29-79-205.ap-northeast-2.compute.internal, uid: aec4bd2d-97e2-444a-a7b7-42b30dcd498a]" propagationPolicy="Background"
  I0405 07:42:07.749214      11 node_lifecycle_controller.go:684] "Controller observed a Node deletion" node="ip-10-29-79-205.ap-northeast-2.compute.internal"
  I0405 07:42:07.749250      11 controller_utils.go:173] "Recording event message for node" event="Removing Node ip-10-29-79-205.ap-northeast-2.compute.internal from Controller" node="ip-10-29-79-205.ap-northeast-2.compute.internal"
  I0405 07:42:07.749426      11 event.go:376] "Event occurred" object="ip-10-29-79-205.ap-northeast-2.compute.internal" fieldPath="" kind="Node" apiVersion="v1" type="Normal" reason="RemovingNode" message="Node ip-10-29-79-205.ap-northeast-2.compute.internal event: Removing Node ip-10-29-79-205.ap-northeast-2.compute.internal from Controller"
  ```

- kube-apiserver-audit logs
  - Delete Node
    ```json
    {
        "kind": "Event",
        "apiVersion": "audit.k8s.io/v1",
        "level": "RequestResponse",
        "auditID": "e6d4bc1f-063b-4572-80ca-018ceb7743d2",
        "stage": "ResponseComplete",
        "requestURI": "/api/v1/nodes/ip-10-29-79-205.ap-northeast-2.compute.internal",
        "verb": "delete",
        "user": {
            "username": "kubernetes-admin",
            "uid": "aws-iam-authenticator:0430********:AIDAQUA5FHPOFECTQ53HX",
            "groups": ["system:masters", "system:authenticated"],
            "extra": {
                "accessKeyId": ["AKIA************LOER"],
                "arn": ["arn:aws:iam::0430********:user/terraform"],
                "canonicalArn": ["arn:aws:iam::0430********:user/terraform"],
                "principalId": ["AIDAQUA5FHPOFECTQ53HX"],
                "sessionName": [""]
            }
        },
        "sourceIPs": ["221.154.70.216"],
        "userAgent": "kubectl/v1.29.2 (darwin/arm64) kubernetes/4b8e819",
        "objectRef": {
            "resource": "nodes",
            "name": "ip-10-29-79-205.ap-northeast-2.compute.internal",
            "apiVersion": "v1"
        },
        "responseStatus": {
            "metadata": {},
            "status": "Success",
            "details": {
                "name": "ip-10-29-79-205.ap-northeast-2.compute.internal",
                "kind": "nodes",
                "uid": "17528ac7-96fb-4ff3-9be6-c16aa5f6622d"
            },
            "code": 200
        },
        "requestObject": {
            "kind": "DeleteOptions",
            "apiVersion": "meta.k8s.io/__internal",
            "propagationPolicy": "Background"
        },
        "responseObject": {
            "kind": "Status",
            "apiVersion": "v1",
            "metadata": {},
            "status": "Success",
            "details": {
                "name": "ip-10-29-79-205.ap-northeast-2.compute.internal",
                "kind": "nodes",
                "uid": "17528ac7-96fb-4ff3-9be6-c16aa5f6622d"
            }
        },
        "requestReceivedTimestamp": "2024-04-05T05:58:52.744075Z",
        "stageTimestamp": "2024-04-05T05:58:52.759205Z",
        "annotations": {
            "authorization.k8s.io/decision": "allow",
            "authorization.k8s.io/reason": ""
        }
    }
    ```
  - Delete Lease
    ```json
    {
        "kind": "Event",
        "apiVersion": "audit.k8s.io/v1",
        "level": "Metadata",
        "auditID": "baacc4d1-b4ed-4790-8b14-0a232aa4f1b2",
        "stage": "ResponseComplete",
        "requestURI": "/apis/coordination.k8s.io/v1/namespaces/kube-node-lease/leases/ip-10-29-79-205.ap-northeast-2.compute.internal",
        "verb": "delete",
        "user": {
            "username": "system:serviceaccount:kube-system:generic-garbage-collector",
            "uid": "cb6aff72-484e-4904-9b23-2b1ae737574e",
            "groups": ["system:serviceaccounts", "system:serviceaccounts:kube-system", "system:authenticated"]
        },
        "sourceIPs": ["172.16.49.75"],
        "userAgent": "kube-controller-manager/v1.29.1 (linux/amd64) kubernetes/07600c7/system:serviceaccount:kube-system:generic-garbage-collector",
        "objectRef": {
            "resource": "leases",
            "namespace": "kube-node-lease",
            "name": "ip-10-29-79-205.ap-northeast-2.compute.internal",
            "apiGroup": "coordination.k8s.io",
            "apiVersion": "v1"
        },
        "responseStatus": {
            "metadata": {},
            "status": "Success",
            "details": {
                "name": "ip-10-29-79-205.ap-northeast-2.compute.internal",
                "group": "coordination.k8s.io",
                "kind": "leases",
                "uid": "bd723e95-a78f-4686-8881-13bd46a816cb"
            },
            "code": 200
        },
        "requestReceivedTimestamp": "2024-04-05T05:58:52.774196Z",
        "stageTimestamp": "2024-04-05T05:58:52.792755Z",
        "annotations": {
            "authorization.k8s.io/decision": "allow",
            "authorization.k8s.io/reason": "RBAC: allowed by ClusterRoleBinding \"system:controller:generic-garbage-collector\" of ClusterRole \"system:controller:generic-garbage-collector\" to ServiceAccount \"generic-garbage-collector/kube-system\""
        }
    }
    ```

삭제된 노드에 접속하여 kubelet의 로그를 확인해 보니 여전히 kube-apiserver에 요청을 보내고 있는 것을 확인할 수 있었습니다.

```bash
# On ip-10-29-79-205
> journalctl -u kubelet -f
Apr 05 06:04:04 ip-10-29-79-205.ap-northeast-2.compute.internal kubelet[477678]: E0405 06:04:04.422460  477678 nodelease.go:49] "Failed to get node when trying to set owner ref to the node lease" err="nodes \"ip-10-29-79-205.ap-northeast-2.compute.internal\" not found" node="ip-10-29-79-205.ap-northeast-2.compute.internal"
Apr 05 06:04:05 ip-10-29-79-205.ap-northeast-2.compute.internal kubelet[477678]: E0405 06:04:05.303006  477678 eviction_manager.go:282] "Eviction manager: failed to get summary stats" err="failed to get node info: node \"ip-10-29-79-205.ap-northeast-2.compute.internal\" not found"
Apr 05 06:04:14 ip-10-29-79-205.ap-northeast-2.compute.internal kubelet[477678]: E0405 06:04:14.072264  477678 kubelet_node_status.go:549] "Error updating node status, will retry" err="error getting node \"ip-10-29-79-205.ap-northeast-2.compute.internal\": nodes \"ip-10-29-79-205.ap-northeast-2.compute.internal\" not found"
Apr 05 06:04:14 ip-10-29-79-205.ap-northeast-2.compute.internal kubelet[477678]: E0405 06:04:14.078210  477678 kubelet_node_status.go:549] "Error updating node status, will retry" err="error getting node \"ip-10-29-79-205.ap-northeast-2.compute.internal\": nodes \"ip-10-29-79-205.ap-northeast-2.compute.internal\" not found"
Apr 05 06:04:14 ip-10-29-79-205.ap-northeast-2.compute.internal kubelet[477678]: E0405 06:04:14.083327  477678 kubelet_node_status.go:549] "Error updating node status, will retry" err="error getting node \"ip-10-29-79-205.ap-northeast-2.compute.internal\": nodes \"ip-10-29-79-205.ap-northeast-2.compute.internal\" not found"
Apr 05 06:04:14 ip-10-29-79-205.ap-northeast-2.compute.internal kubelet[477678]: E0405 06:04:14.087835  477678 kubelet_node_status.go:549] "Error updating node status, will retry" err="error getting node \"ip-10-29-79-205.ap-northeast-2.compute.internal\": nodes \"ip-10-29-79-205.ap-northeast-2.compute.internal\" not found"
Apr 05 06:04:14 ip-10-29-79-205.ap-northeast-2.compute.internal kubelet[477678]: E0405 06:04:14.092890  477678 kubelet_node_status.go:549] "Error updating node status, will retry" err="error getting node \"ip-10-29-79-205.ap-northeast-2.compute.internal\": nodes \"ip-10-29-79-205.ap-northeast-2.compute.internal\" not found"
Apr 05 06:04:14 ip-10-29-79-205.ap-northeast-2.compute.internal kubelet[477678]: E0405 06:04:14.092930  477678 kubelet_node_status.go:536] "Unable to update node status" err="update node status exceeds retry count"
...
```

CloudWatch에서 `kube-apiserver-audit`의 로그를 확인해 보면 lease에 대한 get 요청이 지속적으로 들어오고 있으나, 위에서 `kube-controller-manager`가 삭제했으므로 404 Not Found를 보내고 있다는 것을 확인할 수 있습니다.

```
# On CloudWatch
# kube-apiserver-audit에서 leases 리소스에 대한 get 요청이 오지만 삭제되었으므로  
fields @logStream, @timestamp, @message
| filter @message like "ip-10-29-79-205"
| sort @timestamp desc
```

```json
{
    "kind": "Event",
    "apiVersion": "audit.k8s.io/v1",
    "level": "Metadata",
    "auditID": "8bf8dca8-3fa4-4286-9469-6c5bfb32193e",
    "stage": "ResponseComplete",
    "requestURI": "/apis/coordination.k8s.io/v1/namespaces/kube-node-lease/leases/ip-10-29-79-205.ap-northeast-2.compute.internal?timeout=10s",
    "verb": "get",
    "user": {
        "username": "system:node:ip-10-29-79-205.ap-northeast-2.compute.internal",
        "uid": "aws-iam-authenticator:0430********:AROAQUA5FHPODRA52EIMH",
        "groups": ["system:bootstrappers", "system:nodes", "system:authenticated"],
        "extra": {
            "accessKeyId": ["ASIA********EOHO"],
            "arn": ["arn:aws:sts::0430********:assumed-role/ToyBox-1-29-EKSNodeGroupRole/i-04ccfd26e5c668ef8"],
            "canonicalArn": ["arn:aws:iam::0430********:role/ToyBox-1-29-EKSNodeGroupRole"],
            "principalId": ["AROAQUA5FHPODRA52EIMH"],
            "sessionName": ["i-04ccfd26e5c668ef8"]
        }
    },
    "sourceIPs": ["10.29.79.205"],
    "userAgent": "kubelet/v1.29.0 (linux/amd64) kubernetes/3034fd4",
    "objectRef": {
        "resource": "leases",
        "namespace": "kube-node-lease",
        "name": "ip-10-29-79-205.ap-northeast-2.compute.internal",
        "apiGroup": "coordination.k8s.io",
        "apiVersion": "v1"
    },
    "responseStatus": {
        "metadata": {},
        "status": "Failure",
        "message": "leases.coordination.k8s.io \"ip-10-29-79-205.ap-northeast-2.compute.internal\" not found",
        "reason": "NotFound",
        "details": {
            "name": "ip-10-29-79-205.ap-northeast-2.compute.internal",
            "group": "coordination.k8s.io",
            "kind": "leases"
        },
        "code": 404
    },
    "requestReceivedTimestamp": "2024-04-05T06:04:14.665138Z",
    "stageTimestamp": "2024-04-05T06:04:14.668822Z",
    "annotations": {
        "authorization.k8s.io/decision": "allow",
        "authorization.k8s.io/reason": ""
    }
}
```

추가적으로 위의 `kube-apiserver-audit`을 통해 user.username 혹은 user.groups이 가진 권한으로 lease에 대한 api 요청을 처리하는 것을 볼 수 있습니다.
따라서 지금처럼 404 이슈가 아닌 401 등의 에러가 발생할 경우, `clusterrole`, `clusterrolebinding`등을 확인해 보면 도움이 될 수 있습니다.

[Kubernetes Reference - Lease] 링크를 참고하여 어떠한 요청이 있는지 발생할 수 있는지 참고하여 Control Plane의 로그를 조회하는 것 또한 도움이 될 것 같습니다.

### 2. 노드 재등록

이런 상황에서 kubelet을 재실행하면 어떻게 될지 확인해 보겠습니다.

```bash
# On ip-10-29-79-205
> sudo systemctl restart kubelet
```

이후 CloudWatch에서 Lease Create에 대한 작업을 Kubelet에서 요청했고 잘 생성된 것을 확인할 수 있었습니다.

```json
{
    "kind": "Event",
    "apiVersion": "audit.k8s.io/v1",
    "level": "Metadata",
    "auditID": "33a800c7-1d7c-4882-b42b-73ea4893892a",
    "stage": "ResponseComplete",
    "requestURI": "/apis/coordination.k8s.io/v1/namespaces/kube-node-lease/leases?timeout=10s",
    "verb": "create",
    "user": {
        "username": "system:node:ip-10-29-79-205.ap-northeast-2.compute.internal",
        "uid": "aws-iam-authenticator:0430********:AROAQUA5FHPODRA52EIMH",
        "groups": ["system:bootstrappers", "system:nodes", "system:authenticated"],
        "extra": {
            "accessKeyId": ["ASIA************G2HA"],
            "arn": ["arn:aws:sts::0430********:assumed-role/ToyBox-1-29-EKSNodeGroupRole/i-04ccfd26e5c668ef8"],
            "canonicalArn": ["arn:aws:iam::0430********:role/ToyBox-1-29-EKSNodeGroupRole"],
            "principalId": ["AROAQUA5FHPODRA52EIMH"],
            "sessionName": ["i-04ccfd26e5c668ef8"]
        }
    },
    "sourceIPs": ["10.29.79.205"],
    "userAgent": "kubelet/v1.29.0 (linux/amd64) kubernetes/3034fd4",
    "objectRef": {
        "resource": "leases",
        "namespace": "kube-node-lease",
        "name": "ip-10-29-79-205.ap-northeast-2.compute.internal",
        "apiGroup": "coordination.k8s.io",
        "apiVersion": "v1"
    },
    "responseStatus": {
        "metadata": {},
        "code": 201
    },
    "requestReceivedTimestamp": "2024-04-05T09:38:34.242033Z",
    "stageTimestamp": "2024-04-05T09:38:34.265363Z",
    "annotations": {
        "authorization.k8s.io/decision": "allow",
        "authorization.k8s.io/reason": ""
    }
}
```

restart kubelet 이후, kubelet 및 `kube-apiserver-audit`의 로그를 확인해 보니 정상적으로 Sync가 되는 것을 확인할 수 있습니다.

```bash
# On ip-10-29-79-205
> journalctl -u kubelet -f
Apr 05 09:38:42 ip-10-29-79-205.ap-northeast-2.compute.internal kubelet[502508]: I0405 09:38:42.566068  502508 kubelet.go:2528] "SyncLoop (probe)" probe="readiness" status="" pod="kube-system/aws-node-kx4hk"
Apr 05 09:38:42 ip-10-29-79-205.ap-northeast-2.compute.internal kubelet[502508]: I0405 09:38:42.566204  502508 prober_manager.go:312] "Failed to trigger a manual run" probe="Readiness"
Apr 05 09:38:42 ip-10-29-79-205.ap-northeast-2.compute.internal kubelet[502508]: I0405 09:38:42.601195  502508 kubelet.go:2528] "SyncLoop (probe)" probe="readiness" status="ready" pod="kube-system/aws-node-kx4hk"
```

```bash
# kubectl
> kubectl get node
NAME                                               STATUS   ROLES    AGE     VERSION
ip-10-29-104-121.ap-northeast-2.compute.internal   Ready    <none>   30h     v1.29.0-eks-5e0fdde
ip-10-29-69-101.ap-northeast-2.compute.internal    Ready    <none>   30h     v1.29.0-eks-5e0fdde
ip-10-29-79-205.ap-northeast-2.compute.internal    Ready    <none>   2m51s   v1.29.0-eks-5e0fdde

> kubectl get lease -n kube-node-lease
NAME                                               HOLDER                                             AGE
ip-10-29-104-121.ap-northeast-2.compute.internal   ip-10-29-104-121.ap-northeast-2.compute.internal   30h
ip-10-29-69-101.ap-northeast-2.compute.internal    ip-10-29-69-101.ap-northeast-2.compute.internal    30h
ip-10-29-79-205.ap-northeast-2.compute.internal    ip-10-29-79-205.ap-northeast-2.compute.internal    2m57s
```

```
# On CloudWatch
fields @logStream, @timestamp, @message
| filter @message like "ip-10-29-79-205"
| sort @timestamp desc
```

```json
{
    "kind": "Event",
    "apiVersion": "audit.k8s.io/v1",
    "level": "Metadata",
    "auditID": "550cfc75-64d3-4096-bc88-a645b983d561",
    "stage": "ResponseComplete",
    "requestURI": "/apis/coordination.k8s.io/v1/namespaces/kube-node-lease/leases/ip-10-29-79-205.ap-northeast-2.compute.internal?timeout=10s",
    "verb": "update",
    "user": {
        "username": "system:node:ip-10-29-79-205.ap-northeast-2.compute.internal",
        "uid": "aws-iam-authenticator:0430********:AROAQUA5FHPODRA52EIMH",
        "groups": ["system:bootstrappers", "system:nodes", "system:authenticated"],
        "extra": {
            "accessKeyId": ["ASIA************CTFF"],
            "arn": ["arn:aws:sts::0430********:assumed-role/ToyBox-1-29-EKSNodeGroupRole/i-04ccfd26e5c668ef8"],
            "canonicalArn": ["arn:aws:iam::0430********:role/ToyBox-1-29-EKSNodeGroupRole"],
            "principalId": ["AROAQUA5FHPODRA52EIMH"],
            "sessionName": ["i-04ccfd26e5c668ef8"]
        }
    },
    "sourceIPs": ["10.29.79.205"],
    "userAgent": "kubelet/v1.29.0 (linux/amd64) kubernetes/3034fd4",
    "objectRef": {
        "resource": "leases",
        "namespace": "kube-node-lease",
        "name": "ip-10-29-79-205.ap-northeast-2.compute.internal",
        "uid": "f41aaee7-ebab-49cb-844f-0a1ab5ae43fe",
        "apiGroup": "coordination.k8s.io",
        "apiVersion": "v1",
        "resourceVersion": "308422"
    },
    "responseStatus": {
        "metadata": {},
        "code": 200
    },
    "requestReceivedTimestamp": "2024-04-05T06:26:01.081550Z",
    "stageTimestamp": "2024-04-05T06:26:01.087545Z",
    "annotations": {
        "authorization.k8s.io/decision": "allow",
        "authorization.k8s.io/reason": ""
    }
}
```

### 3. 정리

위의 예시를 통해서 ControlPlane에서 Node 리소스가 삭제되면 `kube-controller-manager`의 `garbagecollector`가 Lease 리소스를 삭제하고, Kubelet이 Node를 등록할 때에는 Kubelet이 자체적으로 Lease 리소스를 생성하는 것을 알 수 있었습니다.

# 마치며

이번에는 Lease가 무엇인지, 어떻게 생성되고 어떻게 삭제되는지 확인해 보았습니다.

그리고 그 과정에서 DataPlane Node가 어떤 과정을 통해서 ControlPlane에 HealthCheck를 하는지 확인할 수 있었습니다.

다음에 기회가 된다면 실제로 Kubelet이 어떤 과정으로 동작하고, kube-controller-manager의 동작이 어떻게 이루어지는지 알아보도록 하겠습니다.

# Reference
- [Lease API - Kubernetes](https://www.youtube.com/watch?v=ttPYCQ922mo)
- [리스 - Lease](https://alive-wong.tistory.com/70)
- [Node Heartbeats](https://kubernetes.io/docs/reference/node/node-status/#heartbeats)
- [Enhancements - efficient node heartbeats](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/589-efficient-node-heartbeats)
- [Kubernetes Reference - Lease](https://dev-k8sref-io.web.app/docs/cluster/lease-v1/)
- [Replica 레벨의 리더 선출 예시](https://www.pulumi.com/ai/answers/sztyN56FbxgbsaWRehCrZk/implementing-high-availability-financial-systems-on-kubernetes)

