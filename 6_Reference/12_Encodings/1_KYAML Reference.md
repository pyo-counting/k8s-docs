KYAML은 더 안전하고 모호성이 적은 YAML의 부분집합(subset)으로, k8s v1.34에서 처음 도입(alpha)되었으며 v1.35에서 기본적으로 활성화(beta)됐다. k8s를 위해 특별히 설계된 KYAML은 기존 YAML 파서 및 도구와의 완벽한 호환성을 유지하면서 공백 민감성(whitespace sensitivity) 및 암시적 타입 강제 변환(implicit type coercion)과 같은 일반적인 YAML의 문제점들을 해결한다.

이 문서는 KYAML 구문에 대해 설명한다.

## Getting started with KYAML
들여쓰기와 암시적 타입 강제 변환에 의존하는 YAML의 특성은 종종 설정 오류를 유발하며, 이는 특히 CI/CD pipeline이나 Helm과 같은 템플릿 시스템에서 두드러진다. KYAML은 명시적인 구문과 구조를 강제함으로써 이러한 문제들을 제거해 더 신뢰할 수 있고 디버깅하기 쉽게 만들어준다.

### Basic Structure
KYAML은 객체에 `{}`를 사용하고 배열에 `[]`를 사용하는 플로우 스타일(flow style) 구문을 사용한다. 모든 문자열 값은 반드시 큰따옴표(double-quoted)로 묶어야 한다.
``` yaml
---
{
  apiVersion: "v1",
  kind: "Pod",
  metadata: {
    name: "my-pod",
    labels: {
      app: "demo"
    },
  },
  spec: {
    containers: [{
      name: "nginx",
      image: "nginx:1.20"
    }]
  }
}
```

주요 규칙은 다음과 같다.
- 값은 항상 큰따옴표로 감싼 문자열을 사용
- 키는 기본적으로 따옴표 없이 작성 가능하지만, 모호성이 있을 경우 따옴표 사용
- 중괄호 `{}`를 사용해 객체 표현
- 대괄호 `[]`를 사용해 배열 표현
- JSON과 달리 주석 지원
- JSON과 달리 마지막 원소 뒤에 쉼표(,) 허용
- 공백과 들여쓰기에 민감하지 않음