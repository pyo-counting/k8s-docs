**Note**:
psp느ㄴ k8s v1.21에서 deprecated 됐으며 v1.25에서 삭제됐다. psp를 사용하는 대신 아래 방법을 사용해 동일한 제한을 po에 적용할 수 있다.

- psa
- 3rd party admission plugin

For a migration guide, see Migrate from PodSecurityPolicy to the Built-In PodSecurity Admission Controller. For more information on the removal of this API, see PodSecurityPolicy Deprecation: Past, Present, and Future.

If you are not running Kubernetes v1.25, check the documentation for your version of Kubernetes.