cm은 데이터를 key-value 쌍으로 저장하는 데 사용되는 API object다. po는 cm을 환경 변수, command-line 인자, volume 내 설정 파일로 사용할 수 있다.

**Caution**: cm은 암호화를 제공하지 않는다. 데이터를 안전하게 저장하기 위해서는 cm이 아닌 secret 또는 third party 툴을 사용하는 것을 권장한다.

## Motivation
cm은 1MiB를 초과할 수 없다.

## ConfigMap object
cm은 다른 object에서 사용할 설정을 저장하기 위한 object다. 대부분의 `.spec` 필드를 갖는 다른 k8s object와 다르게 cm은 `.data`, `.binaryData`, `.immutable` 필드를 갖는다. `.data` 필드는 UTF-8 문자열, `.binaryData` 필드는 base64-encoded string으로 저장될 바이너리 데이터를 위해 사용된다.

## ConfigMaps and Pods
**Note**: static po의 `.spec`에서는 cm 뿐만 아니라 다른 API object를 참조할 수 없다.

``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # property-like keys; each key maps to a simple value
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # file-like keys
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
```

cm을 사용해 po 내 container를 구성하는 네 가지 방법이 있다:

1. container의 command, args
2. container의 환경 변수
3. read-only volume 마운트
4. k8s API를 사용해 cm 사용

네 번째 방식은 continer 내 애플리케이션에서 k8s API를 사용하는 것이므로 잘 사용되지 않지만 다른 ns의 cm에 접근하는 경우에 유용할 수 있다.

## Using ConfigMaps

### Using ConfigMaps as files from a Pod

#### Mounted ConfigMaps are updated automatically
volume에 사용된 cm이 업데이트될 경우, volume내 파일도 결국 업데이트 된다. kubelet은 주기마다 마운트 된 cm이 최신 상태인지 확인한다. 그러나 kbelet은 cm의 현재 값을 얻기 위해 로컬 캐시를 사용한다. 캐시의 타입은 ConfigMapAndSecretChangeDetectionStrategy을 사용해 설정된다. A ConfigMap can be either propagated by watch (default), ttl-based, or by redirecting all requests directly to the API server. 결과적으로 cm이 업데이트되는 순간부터 po내 파일이 업데이트되는 순간까지의 총 지연시간은 kubelet 동기화 시간 + 캐시 propagation 지연만큼 길 수 있다. 여기서 캐시 propagation 지연은 설정된 캐시 타입에 따라 다르다(watch, ttl의 경우 0이라고 간주해도 될 만큼 작음).

환경 변수로 cm을 사용할 경우 자동으로 업데이트되지 않으며 po 재시작이 필요하다.

**Note**: cm을 subPath volume으로 마운트하는 경우 업데이트 되지 않는다.

## Immutable ConfigMaps
secret, cm에 `.immutable` 필드를 제공해 각 object가 변경할 수 없도록 설정할 수 있다. 이를 통한 이점은 다음과 같다:

- 애플리케이션의 중단을 야기할 수 있는 변경으로부터 보호
- immutable로 설정된 cm에 대한 감시를 중단함으로써 API 서버에 대한 부하 감소

이 기능은 ImmutableEphemeralVolumes feature gate에 의해 제어된다.

cm이 immutable로 설정되면 `.immutable`, `.data`, `.binaryData` 필드 내용을 변경할 수 없다. 업데이트를 위해 cm를 삭제 및 재생성해야 한다. 기존 po는 삭제된 cm에 대한 mount point를 유지하므로 po를 재생성하는 것을 권장한다.