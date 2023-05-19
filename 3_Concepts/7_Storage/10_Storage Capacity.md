storage 용량은 제한적이녀 po가 실행되는 no에 따라 달라질 수 있다. network-attached storage에 모든 no가 접근할 수 없거나 storage가 처음부터 no의 로컬에 존재할 수 있다.

k8s가 storage 용량을 추적하는 방법과 스케줄러가 이 정보를 사용해 누락된 나머지 volume에 충분한 storage 용량에 접근할 수 있는 no로 po를 스케줄링하는 방법을 설명한다. storage 용량을 추적하지 않으면 스케줄러가 volume을 provisioning하기에 충분한 용량이 없는 no를 선택할 수 있으며 이로인해 여러 번의 스케줄링 재시도가 필요하다.

## Before you begin
k8s v1.27은 storage 용량 추적을 위한  cluster-level API 지원이 포함되어 있다. 이를 사용하기 위해 용량 추적을 지원하는 CSI driver를 사용해야 한다. 이 지원을 사용할 수 있는지 여부와 사용 방법을 확인하기 위해 사용하는 CSI driver를 참고한다. k8s v1.27을 실행하지 않는 경우 해당 버전의 k8s 문서를 참고한다.

## API
CSIStorageCapacity object: these get produced by a CSI driver in the namespace where the driver is installed. Each object contains capacity information for one storage class and defines which nodes have access to that storage.
The CSIDriverSpec.StorageCapacity 필드: true로 설정되면 k8s 스케줄러는 CSI driver를 사용하는 volume을 위한 storage 용량을 고려한다.

## Scheduling

## Rescheduling

## Limitations