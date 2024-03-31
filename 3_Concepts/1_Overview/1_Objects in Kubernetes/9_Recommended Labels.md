kubectl 명령어외 더 많은 툴을 사용해 k8s 리소스를 시각화하고 관리할 수 있다. 공통 label 집합을 사용하면 여러 툴들이 상호 운영이 가능하며 공통된 방식으로 리소스를 다룰 수 있다.

In addition to supporting tooling, the recommended labels describe applications in a way that can be queried.

The metadata is organized around the concept of an application. Kubernetes is not a platform as a service (PaaS) and doesn't have or enforce a formal notion of an application. Instead, applications are informal and described with metadata. The definition of what an application contains is loose.

> **Note**:  
> 권장 label은 애플리케이션을 보다 쉽게 관리할 수 있지만 필수 사항은 아니다.

공유 label, annotation은 `app.kubernetes.io` 공통 접두사를 공유한다. 접두사가 없는 label은 사용자 개인의 것이다. 접두사를 통해 사용자 label과 공유 label을 구분한다.

## Labels
이러한 label을 최대한 활용하기 위해서는 모든 resource object에 해당 label을 적용하면 된다.

| Key                          | Description                                                                      | Example      | Type   |
|------------------------------|----------------------------------------------------------------------------------|--------------|--------|
| app.kubernetes.io/name       | The name of the application                                                      | mysql        | string |
| app.kubernetes.io/instance   | A unique name identifying the instance of an application                         | mysql-abcxzy | string |
| app.kubernetes.io/version    | The current version of the application (e.g., a SemVer 1.0, revision hash, etc.) | 5.7.21       | string |
| app.kubernetes.io/component  | The component within the architecture                                            | database     | string |
| app.kubernetes.io/part-of    | The name of a higher level application this one is part of                       | wordpress    | string |
| app.kubernetes.io/managed-by | The tool being used to manage the operation of an application                    | helm         | string |

아래는 예시다:
``` yaml
# This is an excerpt
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: mysql-abcxzy
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/created-by: controller-manager
```

## Applications And Instances Of Applications

## Examples

### A Simple Stateless Service

### Web Application With A Database