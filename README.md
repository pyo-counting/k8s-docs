## about docs
- v1.30 버전 기준 작성

## configure cluster
### general
- k8s의 모든 구성 요소를 container로 관리하는 것을 권장한다. 하지만 container 실행을 담당하는 kubelet은 container로 실행할 수 없다. ([Getting started](https://kubernetes.io/docs/setup/))
- resource 마다 object naming 규칙에 대한 제약 사항이 더 많을 수도 있다. 가장 보수적으로 RFC 1035를 준수하는 것이 편하다. ([Object Names and IDs](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#rfc-1035-label-names))
- ns에 대해 `kube-` 접두사는 k8s 시스템을 위해 예약됐기 때문에 사용하지 않는다. ([Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/%2523working-with-namespaces))
- ns의 이름을 사용해 cluster 내에서 domain을 생성하기 때문에 제한된 사용자만 ns를 만들 수 있도록 제한한다. ([Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/#namespaces-and-dns))
- ns의 limit, hierarchical limit을 고려한다. ([Production environment](https://kubernetes.io/docs/setup/production-environment/#set-limits-on-workload-resources))
- `kubernetes.io/`, `k8s.io/` 접두사는 k8s core system에서 사용되도록 예약됐다. ([Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/#syntax-and-character-set))
- owner reference는 k8s resource 간의 종속 관계를 나타낸다. k8s는 object 삭제 시 label이 아닌 owner reference를 사용해 종속 관계에 대한 cascading deletion(background 또는 foreground)을 수행한다 ([Owners and Dependents](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/))

### control plane
- control plane 구성 요소는 3개 이상(raft 알고리즘을 위해)의 failure zone에서 실행하는 것을 권장한다 ([Production environment](https://kubernetes.io/docs/setup/production-environment/#production-control-plane))
- control plane 구성 요소(예를 들어 kube-scheduler, kube-controller-manager)의 여러 replica는 kube-node-lease ns에 저장된 lease object를 사용해 reader를 관리한다 ([Leases](https://kubernetes.io/docs/concepts/architecture/leases/#leader-election))
- kube-apiserver는 kube-node-lease ns에 저장된 lease object를 통해 k8s 전체 시스템에 kube-apiserver에 대한 정보를 제공한다 ([Leases](https://kubernetes.io/docs/concepts/architecture/leases/#api-server-identity))
- encryption at rest 고려 ([Security](https://kubernetes.io/docs/concepts/security/#control-plane-protection))
- kube-controller-manager
  - no의 non-graceful shutdown 처리를 위한 `NodeOutOfServiceVolumeDetach` feature gate 활성화 여부 확인. ([Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/#non-graceful-node-shutdown))
  - no의 non-graceful shutdown 처리 대신 po의 삭제가 6분동안 실패할 경우 강제로 volume mount를 해제하는 `disable-force-detach-on-timeout` 설정 확인. ([Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/#storage-force-detach-on-timeout))
- kube-apiserver
  - kube-apiserver -> kubelet 통신 시, 기본적으로 kube-apiserver는 kubelet의 server certificate를 검증하지 않는다. 검증을 위해 `--kubelet-certificate-authority` flag에 kubelet의 ca certificate를 설정할 수 있다.
  - authorization 방식 설정을 위해 `--authorization-mode` flag를 사용한다.

### node
- no당 최대 po 실행 갯수는 110개 ([Considerations for large clusters](https://kubernetes.io/docs/setup/best-practices/cluster-large/))
- no가 k8s cluster의 no에 join하기 위한 요구 사항을 검증을 위해 node conformance test를 수행할 수 있다. ([Validate node setup](https://kubernetes.io/docs/setup/best-practices/node-conformance/))
- no 모니터링을 위한 node problem detector 설치 ([Production environment](https://kubernetes.io/docs/setup/production-environment/#production-worker-nodes))
- no의 graceful/non-graceful shutdown 설정 고려 ([Node Shutdowns](https://kubernetes.io/docs/concepts/cluster-administration/node-shutdown/))
- kublet과 container runtime 모두 control group을 통해 po, container에 대한 리소스 관리를 수행한다. control group은 cgroup driver를 통해 사용하며 kubelet과 container runtime이 동일한 cgroup driver를 사용하는 것이 중요하다 ([Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cgroup-drivers))
- no의 cgroup v2 설정 고려 ([About cgroup v2](https://kubernetes.io/docs/concepts/architecture/cgroups/))
- kubelet의 대부분 flag는 deprecated이며 대신 config file을 통해 설정하는 것을 권장한다. control plane 구성 요소의 경우 아직은 flag를 이용한 설정을 사용하는 것 같다.
- kubelet에서 container runtime에 접근하기 위한 endpoint 설정 고려 ([Container Runtime Interface (CRI)](https://kubernetes.io/docs/concepts/architecture/cri/#api))
- kubelet에서 image 용량에 따른 gc 설정 고려 ([Garbage Collection](https://kubernetes.io/docs/concepts/architecture/garbage-collection/#container-image-lifecycle))
- kubelet에서 image age에 따른 gc 설정 고려 ([Garbage Collection](https://kubernetes.io/docs/concepts/architecture/garbage-collection/#image-maximum-age-gc))
- kubelet이 특정 label을 마음대로 수정할 수 없도록 NodeRestriction admission plugin 설정 고려 ([Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-isolation-restriction))

### addon
- addon의 기본 limit은 일반적으로 작은 크기의 cluster에서의 경험을 통해 수집한 데이터를 기반으로 하기 때문에 조정 필요 ([Addon resources](https://kubernetes.io/docs/setup/best-practices/cluster-large/#addon-resources))
- cluster의 사이즈가 커짐에 따라 자동으로 addon의 스케일링을 위해 addon resizer 고려 ([Considerations for large clusters](https://kubernetes.io/docs/setup/best-practices/cluster-large/#what-s-next))
- k8s object의 status를 노출하는 kube-state-metrics 설치 고려 ([Metrics for Kubernetes Object States](https://kubernetes.io/docs/concepts/cluster-administration/kube-state-metrics/))

## 요약
- k8s의 autoscaling 옵션: pod hpa, vpa, cluster autoscaler, addon resizer
- linux container는 격리를 위해 namespace, cgroup(control group) 기술을 사용한다.
  - namespace:
    - `mnt`: mount points
    - `user`: user and group ids
    - `uts(unix time sharing)`: hostname, nis domain name
    - `pid`: process ids
    - `network`: network devices, stacks, ports, etc
    - `ipc(inter process communication)`: system V IPC, POSIX message queues
    - `control groups`: control group root directory
  - cgroup
    - 메모리, cpu, 네트워크 대역폭 등
- 이론적으로 container는 모든 리눅스 시스템에서 실행될 수 있지만 호스트의 커널을 사용하기 때문에 커널 버전에 영향을 받는다. 뿐만 아니라 특정 하드웨어 아키텍쳐용(x86, ARM 등)으로 만들어진 image는 해당 아키텍쳐에서만 실행될 수 있다.
- docker 기준 k8s는 po내 모든 container가 동일한 linux namespace를 공유하도록 docker를 설정한다. 파일 시스템의 경우 image에 저장되어 있어 기본적으로 완전히 분리된다.
  - 동일한 namespace: network, uts, ipc
  - 다른 namespace: uid, pid, mnt
- po의 manifest 중 spec.containers[\*].ports[\*].containerport는 container가 노출하는 포트 정보를 명시하기만 할 뿐 다른 기능은 없다. 이를 생략한다고 해서 포트를 통해 po에 연결할 수 있는 여부에 영향을 미치지 않는다.
- po는 k8s에서 배포할 수 있는 가장 작은 단위로, 이미 실행 중인 po 내에 container를 배포할 수 없다. 물론 po에 포함된 container가 po 내에서 재시작 될 수 있다.
- static po는 다른 po들(control plane이 담당)과 다르게 해당 no의 kubelet이 직접 관리한다. 그리고 static po에 대한 object를 API server에 게시한다(API server를 통해 object를 변경하는 것은 불가능). 즉 static po가 특정 kubelet에 종속되어 있지만 API server를 통해 조회가 가능하다. static po의 .spec에서는 다른 API object를 참조할 수 없다(예를 들어 sa, cm, secret 등).
- container 로그는 하루 단위, 10MB 크기 기준으로 롤링(rolling)된다.
- k8s namespace는 리소스 이름의 범위를 제공한다. 뿐만 아니라 리소스를 격리하는 것외에도 특정 사용자가 지정된 리소스에 접근할 수 있도록 허용하고, 개별 사용자가 사용할 수 있는 컴퓨팅 리소스를 제한하는 데에도 사용된다. 하지만 리소스간 격리는 제공하지 않는다. 즉 서로 다른 namepsace에 존재하는 리소스더라도 통신할 수 있다. 네트워크 격리는 k8s와 함께 배포되는 네트워킹 솔루션에 따라 다르다.
- container의 생명주기와 관련해 hook을 제공하며, handler를 구현함으로써 hook에 대한 이벤트를 처리할 수 있다.
  - PostStart: container와 비동기적으로 실행된다. 하지만 PreStart가 완료되지 않으면 container는 running state에 도달할 수 없다.
  - PreStop: container에 TERM 시그널이 전송되기 전에 실행된다. po의 terminationGracePeriodSeconds 설정보다 PreStop, TERM signal 전송 및 처리 시간이 더 오래걸이면 container는 비정상 종료 될 수도 있다. po가 terminated 상태가 되고 난 후, prestop hook -> container TERM sinal 전송된다.
- k8s no resource는 해당 호스트의 image 목록도 저장한다. 이는 k describe no 명령어를 사용해 조회되지 않는 정보다.
- svc의 .spec.selctor는 .spec.type이 ClusterIP, NodePort, LoadBalancer일 떄만 적용됨. ExternalName일 때는 무시된다고 documentation에 명시되어 있지만, ep는 생성됨을 확인(물론 프록시는 되지 않음). 
- svc의 .spec.selector가 없는 svc는 일반적인 svc와 동일하게 동작한다. 하지만 ep를 자동으로 생성하지 않으며 사용자가 직접 생성/관리해야 하는 책임이 있다.
- svc의 .spec.type을 ExternalName으로 바꾸거나 반대로 바꾸지 않는 이상 svc가 수정되더라도 clusterIP는 고정 값이다.
- svc의 .spec.clusterIP는 .spec.type이 NodePort, ClusterIP, LoadBalancer일 때 자동 할당되거나 사용자가 설정할 수 있다.
- svc의 .spec.clusterIP가 None: headless svc를 의미(cluster ip 미할당). kube-proxy가 해당 svc를 다루지 않기 때문에 프록시, 로드밸런싱되지 않는다. 즉 .spec.ports[*]를 명시해도 의미가 없다.
  - .spec.selector가 있으면 svc DNS lookup 시 pod들의 ip 목록(A 레코드)가 조회됨. ep도 생성됨. .spec.clusterIP: "None"이 아니면 오류. 조회된 pod의 ip와 po의 노출 po로 접근할 수 있다.
  - .spec.selector가 없으면 svc DNS lookup 시 아무것도 조회 안됨
- svc의 .spec.type이 ClusterIP: \<cluster ip>:\<port>로 접근 가능
- svc의 .spec.type이 NodePort: \<node ip>: \<node port>, \<cluter ip>:\<port>로 접근 가능 
- svc의 .spec.type이 ExternalName: .spec.externalName 필드 필수로 설정 필요. .spec.clusterIP 핃드를 명시하지 않거나 값을 ""으로 설정 필요. cluster ip, port 할당되지 않음. svc DNS lookup 시 CNAME 레코드 조회됨.
- volume은 po의 수명주기와 같은 ephemeral, po의 수명주기와 상관없는 persistent volume을 사용할 수 있다. po 내 .spec.volumes[*] 필드에서는 persistent volume을 사용하기 위해 pvc를 사용해 pv resource를 요청한다. 즉, 직접 pv를 명시하는 것은 아니다. generic ephemeral pv의 경우에는 inline pvc를 사용하기 때문에 reclaim policy가 Retain이면 po의 수명주기와 상관없이 pv가 삭제되지 않는다. generic ephemeral pv의 경우 pvc의 이름이 po에 따라 다르게 생성되기 때문에 동일 static po에 대한 claim 시 1개 po의 pvc에 대해서만 bound 된다.
- 동일 pvc를 사용한다면 여러 po에서 동일 pv를 바인딩할 수 있다.
- 사용자는 pvc의 .spec.storageClassName 필드를 사용해 pv를 dynamic provisioning할 sc를 명시한다. 이 때 pvc는 먼저 요청에 매칭되는 static pv를 찾고 없으면 dynamic pv를 생성한다. 해당 필드가 ""라면 dynamic provisioning을 비활성화한다는 의미다. pv는 pvc, pvc는 po에 바운딩 되며 관련 finalizer를 갖는다. pv가 pvc로부터 release된 이후, reclaim policy에 따라 삭제 또는 보존될 수 있다. pvc내 .spec.volumeName, .spec.claimRef 필드를 통해 특정 pvc와 특정 pv를 바인딩할 수 있다.
- 1개의 pv는 1개의 pvc와 바운딩된다는 것을 명심해야 한다. 대신 1개의 pvc는 여러 po에서 사용할 수 있다.
- pv는 아래 state를 갖는다.
  - Available: claim에 대해 아직 바운드 되지 않은 리소스
  - Bounded: claim에 바운딩된 volume
  - Released: claim이 삭제되었지만, 리소스는 아직 cluster가 reclaim하지 않음
  - Failed: volume이 dynamic reclaim에 대해 실패함

---
## 명령어
- `kubectl get ${RESOURCE} -l`: label selector
- `kubectl get ${RESOURCE} -L`: 출력 column에 추가할 label
- `kubectl get ${RESOURCE} --field-selector`: field selector
- `kubectl get ${RESOURCE} --as ${USER} --as-group {GROUP}`: impersonation
- `kubectl auth whoami`: 사용자 인증에 대한 정보 확인(selfsubjectreview resource)
- `kubectl auth can-i`: 사용자의 동작에 대한 인가 정보 확인(selfsubjectaccessreview resource)
- `kubectl certificate (approve|deny)`: csr에 대한 승인/거부
- `kubectl delete ${RESOURCE}/${NAME} --cascade`: cascade deletion. 기본값은 background
- `kubectl drain ${NODE}`: API-initiated eviction

---
## 체크리스트
- po내 ports[*].hostPort에 사용된 port는 호스트 netstat 조회 시 보이지 않음. 하지만 type=ClusterIP svc로 expose 시 netstat에 조회됨
- svc externalIPs 설정 시, no의 IP로 svc 접근 가능
- local pv의 경우, pv 생성 시 디렉토리가 호스트 내 존재해야 함. 자동 생성되지 않음 확인
- hpa의 cpu 사용률 계산 시 po를 선택하는 방법 알아보기
- container의 liveness, readiness, startup probe들도 back-off 재시작이 있는지

---
## k8s addon(plugin)
- aws EFS csi driver
  - elastic file system에 대해 access point를 생성하면 실제 file system의 root directory를 숨길 수 있다. 생성한 access point를 통해 접근하면 실제 file system의 루트 디렉토리가 아닌 access point의 루트 디렉토리에 접근한다.
  - access point를 생성할 때 설정 가능한 옵션은 다음과 같다.
    - root directory path: access point의 루트 디렉토리로 사용할 경로(file system 기준 절대 경로).
    - udi, gid, secondary gid: acccess point를 통해 접근하는 사용자의 uid, gid를 강제한다. 기본적으로 efs의 경우 root user(UID 0)만 rwx 권한을 갖는다. 다른 유저에 대해서는 직접 권한을 부여해야한다.
    - OwnerUid, OwnerGiD, Permissions: root directory path를 생성할 떄 사용할 디렉토리의 소유자, 그룹 권한 정보
  - ```
    EFS CSI driver supports dynamic provisioning and static provisioning. Currently Dynamic Provisioning creates an access point for each PV. This mean an AWS EFS file system has to be created manually on AWS first and should be provided as an input to the storage class parameter(parameters.fileSystemId). For static provisioning, AWS EFS file system needs to be created manually on AWS first. After that it can be mounted inside a container as a volume using the driver.
    ```
      - dynamic provisioning
        - AWS 상에 file system이 먼저 생성되어 있어야 한다.
        - pv에 따라 해당 file system 내에서 access point를 생성한다.
        - access point를 사용해 file system에 접근할 경우 동일 file system 내에서도 특정 경로를 투르 디렉토리로 인식하도록 만든다.
        - access point를 생성 시 설정 가능한 [파라미터](https://github.com/kubernetes-sigs/aws-efs-csi-driver/tree/master#storage-class-parameters-for-dynamic-provisioning)
        - access point의 root directory 경로는 \${.parameters.basePath}/\${.parameters.subPathPattern}가 된다.
      - static provisioning
        - AWS 상에 file system이 먼저 생성되어 있어야 한다.
        - access point 없이 file system에 접근이 가능하다.
        - pv를 생성할 때 [`.csi.volumeHandle`](https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/examples/kubernetes/access_points/README.md#efs-access-points) 필드를 통해 file system id, sub-path, access point 설정이 가능하다. 포맷은 `[FileSystemId]:[Subpath]:[AccessPointId]`
      - efs csi driver 설치 시, `delete-access-point-root-dir` 파라미터를 통해 dynamic provisioning을 통해 생성된 pv가 삭제될 떄 관련 access point의 삭제 여부도 제어할 수 있다.
  - efs csi driver에서 지원하는 accessmode는 소스코드 상 ReadWriteOnce(RWO), ReadWriteMany(RWX)를 지원하는 것으로 보인다(https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/7d0b76bca26f7e0c258f3cdc68286aacffbfe5b3/pkg/driver/node.go#LL36C3-L37C3).
    - efs csi driver의 경우 CSI ephemeral volumes은 지원하지 않는다.
- aws lbc(load balancer controller)
  - aws application elb의 경우 k8s ing resoucre, network elb의 경우 k8s svc resourcew를 통해 provision된다.
    - network elb(svc)
      - svc의 .spec.ports[*].port가 aws network elb 상에서 listener로 등록되며, 각 listener의 모든 target은 svc의 .spec.selector에 매칭되는 po의 집합이다. 각 listener의 target group은 k8s 상에서 TargetGroupBinding(crd)로 구현된다.
      - ing와 같이 ingress group을 사용해 여러 svc에서 1개의 aws network elb를 공유하는 것은 불가능하다. 단지 1개의 svc에서 정의된 .spec.ports[*] 필드를 사용해 여러 listener를 만들고 실제 target group은 해당 svc의 .spec.selector에 매칭되는 po들의 집합이다.
    - application elb(ing)
      - annotation을 통해 listener port를 지정할 수 있으며, 기본적으로 .spec.rules[*]에 설정된 rule 별로 설정된 svc가 listener의 target group으로 등록된다. target group은 k8s 상에서 TargetGroupBinding(CRD)로 구현된다.
      - annotation을 사용해 1개의 rule에 대한 target group에 대한 condition, action을 설정할 수 있다. 하지만 target group에 대한 health check, order, protocol, attributes 등은 ing에서 한 번만 지정되기 때문에 해당 설정을 공유하게 된다.
