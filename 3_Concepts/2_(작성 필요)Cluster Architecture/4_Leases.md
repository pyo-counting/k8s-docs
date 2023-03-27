아래는 kubectl get -n kube-node-lease lease/${LEASE_NAME} -o yaml 명령어 출력 결과다:

``` yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  creationTimestamp: "2023-03-19T08:52:42Z"
  name: ip-172-31-100-75.ap-northeast-2.compute.internal
  namespace: kube-node-lease
  ownerReferences:
  - apiVersion: v1
    kind: Node
    name: ip-172-31-100-75.ap-northeast-2.compute.internal
    uid: aadf774b-902a-45bf-ba09-f24b4024bbcb
  resourceVersion: "4442214"
  uid: 3f254bdd-c2dc-44a4-9fd3-1461316b9055
spec:
  holderIdentity: ip-172-31-100-75.ap-northeast-2.compute.internal
  leaseDurationSeconds: 40
  renewTime: "2023-03-27T02:00:40.965759Z"
```

- k8s lease 리소스는 no 리소스에 종속되었다.