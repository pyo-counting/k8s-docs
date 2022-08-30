## Introduction
projected volume은 여러 volume source를 동일한 디렉터리에 매핑한다.

현재, 아래 volume 타입만 projected volume에 사용될 수 있다:

- secret
- downwardAPI
- configMap
- serviceAccountToken

모든 volume은 po와 동일한 ns에 존재해야 한다.

### Example configuration with a secret, a downwardAPI, and a configMap