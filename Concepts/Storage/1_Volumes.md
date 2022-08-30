## Background
k8s는 다양한 유형의 volume 타이블 지원한다. po는 여러 volume 타입을 동시에 사용할 수 있다. ephemeral volume 타입은 po와 같은 lifetime을 갖지만 persistent volume은 po의 lifetime과 관계없다. po가 삭제되면 ephemeral volume도 삭제되지만 persistent volume은 삭제되지 않는다.

기본적으로 volume은 디렉터리로 po 내 container에서 접근할 수 있다.

volume은 다른 volume 내에 마운트될 수 없다. 또한 volume은 다른 volume 내 content에 대한 hard link를 포함할 수 없다.

## Types of Volumes

### configMap
cm은 설정 데이터를 po내 주입(inject)하는 방법을 제공한다. cm에 저장된 데이터는 configMap 타입의 voolume에서 참조되고 po내 실행되는 container에서 접근할 수 있다.

**Note**:
- volume 사용 이전에 cm을 먼저 생성해야 한다.
- cm을 subPath volume으로 사용할 때 cm에 대한 업데이트가 반영되지 않는다.
- 텍스트 데이터는 UTF-8 인코딩을 사용한 파일로 저장된다. 다른 문자 인코딩의 경우 binaryData를 이용해야 한다.

### downwardAPI
volume 내에서 노출된 데이터를 일반 텍스트 형식의 읽기 전용 파일로 사용할 수 있다.

**Note**: downward API를 subpath volume으로 사용할 때 업데이트가 반영되지 않는다.

### emptyDir
emptyDir volume은 po가 no에 할당될 때 처음 생성되며, po가 실행되는 동안에만 존재한다. emptyDir volume은 초기에 비어있다.

**Note**: container가 crash 될 때 po는 no에서 삭제되지 않는다. 그렇기 때문에 container crash로부터 emptyDir volume의 데이터는 안전하다.

emptyDir.medium 필드를 "Memory"로 설정하면 k8s는 tmpfs(RAM 기반 파일시스템)를 사용한다.

**Note**: SizeMemoryBackedVolumes feature gate가 활성화되면 memory 기반 volume에 대해 크기를 지정할 수 있다. 크기를 지정하지 않으면 리눅스 호스트 메모리의 50%로 조정된다.

### hostPath
**Warning**: hostPath volume에는 많은 보안 결함이 있기 때문에 가능하면 사용하지 않는 것이 좋다. hostPath volume을 사용해야 하는 경우 ReadOnly로 마운트하는 것을 권장한다.

AdmissionPolicy를 사용해 특정 디렉토리의 hostPath 접근을 제한하는 경우, readOnly 마운트를 사용하는 정책이 유효하기 위해 volumeMounts 필드가 반드시 지정되어야 한다.

### local
local volume은 디스크와 같은 로컬 스토리지 장치를 나타낸다.

local volume은 정적으로 생성된 pv로만 사용할 수 있다. 동적 provision은 지원하지 않는다.

### persistentVolumeClaim

### projected
projected volume은 여러 기존 volume을 동일한 디렉터리에 매핑한다.

### secret

### 