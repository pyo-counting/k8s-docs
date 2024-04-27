dynamic volume provision을 통해 필요에 따라 storage volume을 생성할 수 있다. dynamic provision 없이 cluster 관리자는 수동으로 새로운 storage volume을 생성하고 k8s 내 pv 객체를 생성할 수 있다. dynamic provision 기능은 이러한 수동 작업을 없애준다. 대신 사용자 요청에 따라 자동으로 storage를 provision한다.

## Background
dynamic volume provisioning은 storage.k8s.io API group의 sc API object로 구현된다. 클러스터 관리자는 필요한만큼 sc를 정의할 수 있으며 각각은 volume을 provision하는 volume plugin(provisioner) 설정과 provisioning시 필요한 파라미터를 갖는다. 클러스터 관리자는 클러스터 내 여러 storage를 제공할 수 있다. 이를 통해 사용자는 실제 storage에 대한 복잡성을 이해할 필요가 없다.

## Enabling Dynamic Provisioning
dynamic provisioning을 활성화하기 위해 클러스터 관리자는 sc를 먼저 생성해야 한다. sc 객체는 provisioner와 필요한 파라미터를 정의한다. sc의 이름은 DNS subdomain name을 따라야 한다.

아래는 예시다:

``` yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
```

``` yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```

## Using Dynamic Provisioning
사용자는 pvc 내 .spec.storageClassName 필드를 사용해 dynamic provisioning을 요청한다. k8s v1.6 이전 버전에서는 annotation을 이용했지만 v1.9 버전에서 deprecated 됐다.

아래는 예시다:

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim1
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast
  resources:
    requests:
      storage: 30Gi
```

## Defaulting Behavior
sc를 지정하지 않는 경우에도 dynamic provisiong될 수 있도록 설정할 수 있다. 관리자는 아래 동작을 통해 기본 동작을 설정한다:

- 특정 sc를 기본값으로 설정한다.
- API server 내 DefaultStorageClass admission controller를 활성화한다.

storageclass.kubernetes.io/is-default-class annotation을 추가해 기본 sc를 설정할 수 있다. 클러스터 내 기본 sc가 존재하는 경우 사용자가 .spec.storageClassName 필드가 없는 pvc를 생성하면 DefaultStorageClass admission controller가 자동으로 기본 sc를 가리키도록 해당 필드를 추가한다.

Note that there can be at most one default storage class on a cluster, or a PersistentVolumeClaim without storageClassName explicitly specified cannot be created.

## Topology Awareness
[Multi-Zone](https://kubernetes.io/docs/setup/best-practices/multiple-zones/) 클러스터 내에서 Region 내 여러 Zone에 po가 배포될 수 있다. Single-Zone storage backend는 po가 배포된 Zone에 프로비저닝 되어야 한다. This can be accomplished by setting the Volume Binding Mode.