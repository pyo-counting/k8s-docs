kubernetes를 multiple zone에서 운영하는 방법을 설명한다.

## Background
kubernetes는 여러 failure zone에 걸쳐 동작할 수 있도록 설계됐다. 일반적으로 zone은 region이라는 논리적 그룹에 적합(?)하다. 주요 cloud provider는 failure zone(또는 availability zone)의 집합을 region이라고 정의하며 일관된 기능을 제공한다. region 내에서 각 zone은 동일한 API와 서비스를 제공한다.

일반적인 cloud architecture는 한 zone의 장애가 다른 zone의 서비스를 손상시키는 것을 최소화하는 것을 목표로한다.

## Control plane behavior
## Node behavior
### Distributing nodes across zones
## Manual zone assignment for Pods
## Storage access for zones
## Networking
## Fault recovery