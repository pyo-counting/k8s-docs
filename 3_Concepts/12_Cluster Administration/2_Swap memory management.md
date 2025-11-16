## Swap memory management
no에 swap을 활성화하기 위해 kubelet의 `NodeSwap` feature gate 활성화(기본 값 true), kubelet의 `.failSwapOn`이 false(기본 값 true)어야한다. po가 swap을 사용하기 위해서는 kubelet의 `.swapBehavior`이 NoSwap (기본 값)이면 안된다.

swap은 cgroup v2에서만 지원한다.