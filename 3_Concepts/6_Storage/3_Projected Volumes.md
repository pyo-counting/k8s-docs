## Introduction
projected volume은 여러 volume source를 동일한 디렉터리에 매핑한다.

현재, 아래 volume 타입만 projected volume에 사용될 수 있다:

- secret
- downwardAPI
- configMap
- serviceAccountToken

모든 volume은 po와 동일한 ns에 존재해야 한다.

### Example configuration with a secret, a downwardAPI, and a configMap

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
spec:
  containers:
  - name: container-test
    image: busybox:1.28
    volumeMounts:
    - name: all-in-one
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: mysecret
          items:
            - key: username
              path: my-group/my-username
      - downwardAPI:
          items:
            - path: "labels"
              fieldRef:
                fieldPath: metadata.labels
            - path: "cpu_limit"
              resourceFieldRef:
                containerName: container-test
                resource: limits.cpu
      - configMap:
          name: myconfigmap
          items:
            - key: config
              path: my-group/my-config
```

### Example configuration: secrets with a non-default permission mode set

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
spec:
  containers:
  - name: container-test
    image: busybox:1.28
    volumeMounts:
    - name: all-in-one
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: mysecret
          items:
            - key: username
              path: my-group/my-username
      - secret:
          name: mysecret2
          items:
            - key: password
              path: my-group/my-password
              mode: 511
```

Each projected volume source is listed in the spec under sources. The parameters are nearly the same with two exceptions:

For secrets, the secretName field has been changed to name to be consistent with ConfigMap naming.
The defaultMode can only be specified at the projected level and not for each volume source. However, as illustrated above, you can explicitly set the mode for each individual projection.

## serviceAccountToken projected volumes
sa의 토큰을 po내 특정 위치에 마운트할 수 있다. 아래는 예시다:

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: sa-token-test
spec:
  containers:
  - name: container-test
    image: busybox:1.28
    volumeMounts:
    - name: token-vol
      mountPath: "/service-account"
      readOnly: true
  serviceAccountName: default
  volumes:
  - name: token-vol
    projected:
      sources:
      - serviceAccountToken:
          audience: api
          expirationSeconds: 3600
          path: token
```

The example Pod has a projected volume containing the injected service account token. Containers in this Pod can use that token to access the Kubernetes API server, authenticating with the identity of the pod's ServiceAccount. The audience field contains the intended audience of the token. A recipient of the token must identify itself with an identifier specified in the audience of the token, and otherwise should reject the token. This field is optional and it defaults to the identifier of the API server.

The expirationSeconds is the expected duration of validity of the service account token. It defaults to 1 hour and must be at least 10 minutes (600 seconds). An administrator can also limit its maximum value by specifying the --service-account-max-token-expiration option for the API server. The path field specifies a relative path to the mount point of the projected volume.

**Note**: A container using a projected volume source as a subPath volume mount will not receive updates for those volume sources.

## SecurityContext interactions
The proposal for file permission handling in projected service account volume enhancement introduced the projected files having the correct owner permissions set.

### Linux
linux 환경에서 .spec.securityContext.runAsUser 필드가 설정되어 있고 projected volume이 있는 po의 경우, projected 파일은 container user ownership을 포함한 올바른 ownership set을 갖는다.

When all containers in a pod have the same runAsUser set in their PodSecurityContext or container SecurityContext, then the kubelet ensures that the contents of the serviceAccountToken volume are owned by that user, and the token file has its permission mode set to 0600.

**Note**: Ephemeral containers added to a Pod after it is created do not change volume permissions that were set when the pod was created.

If a Pod's serviceAccountToken volume permissions were set to 0600 because all other containers in the Pod have the same runAsUser, ephemeral containers must use the same runAsUser to be able to read the token.