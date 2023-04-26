ingress 리소스가 정상적으로 동작하기 위해서는 클러스터 내에 관련 ingress controller가 실행 중이어야 한다.

kube-controller-manager에 의해 동작하는 다른 controller와 다르게 ingress controller는 클러스터에서 자동으로 실행되지 않는다. 아래에서는 사용자 클러스터 환경에 맞는 ingress controller 선택을 위한 내용을 제공한다.

k8s에서는 AWS, GCE, nginx ingress controller를 프로젝트로 관리한다.

## Ingress controllers

## Using multiple Ingress controllers
