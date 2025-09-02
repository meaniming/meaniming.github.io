# [KR] TA 입장에서 정리해본 인터체인지 프레임워크

<aside>
💡

TA 기준에서 작성하였으니 플머는 가라 훠이훠이

</aside>

# 이 글의 목적

**인터체인지 프레임워크**가 무엇인지 이해하기

프로젝트에서 어떻게 활용할 수 있는지 살펴보기

TA가 왜 이런 시스템을 만지게 되었는지 공유하기

---

# 예상 독자

**인터체인지 프레임워크**가 뭔지 궁금한 사람

프로젝트에서 임포트 파이프라인을 세팅해야 하는 사람

왜 어째서 TA가 저런 짓을 하게 되었는지 궁금한 사람

---

# 인터체인지 프레임워크

언리얼 엔진의 **새로운 임포트/익스포트 시스템**으로,

예전처럼 엔진이 정해준 방식대로만 가져오는 게 아니라,

데이터 변환과 처리 과정을 모듈 단위로 나눠서 **필요한 부분만 수정하거나 확장**할 수 있게 바뀌었다.

---

# 인터체인지 흐름

인터체인지 흐름은 다음과 같다:

**Source Data → Translator → Pipeline → Factory + Payload → Import**

### 1. Source Data (원본 데이터)

![image.png](image.png)

- 아티스트가 만든 외부 파일
- 언리얼이 직접 사용할 수 없는 원시 형식(raw format)

### 2. Interchange Translator

- 원본 데이터를 읽어서 **Interchange Node Graph**를 생성한다.
- 즉, Translator는 Source Data(FBX, PNG, JSON 등)를 분석해 `UInterchangeBaseNode` 계열 노드를 만들어 컨테이너에 넣는다.
- 이렇게 생성된 Node Graph는 이후 Pipeline과 Factory에서 활용된다.
- 노드 예시:
    - `UInterchangeTexture2DNode` → 텍스처 데이터 표현
    - `UInterchangeMaterialInstanceNode` → 머티리얼 인스턴스 데이터 표현
- 필요하다면 **커스텀 Translator**를 작성해 JSON 같은 별도 포맷을 읽고, 원하는 데이터만 언리얼로 임포트할 수도 있다.

### 3. Interchange Pipeline

- Translator가 만든 **Node Graph를 해석**해서, 실제 임포트 시 어떤 규칙/정책을 적용할지 결정한다.
- 파이프라인 로직 예시:
    - 텍스쳐: 노멀맵일 경우 → sRGB = false, 압축 = BC5
    - 메시: 특정 메시들을 nanite 활성화 / 비활성화 자동화 가능
- 여러 개의 Pipeline을 조합해 사용할 수 있으며, **블루프린트 또는 C++로 커스텀 Pipeline**을 작성할 수 있다.

### 4. Factory Node

- 최종 애셋 생성 청사진 역할을 한다.
- Pipeline에서 Node Graph를 해석한 뒤, 해당 정보를 토대로 Factory Node를 만든다.
- 예시:
    - `UInterchangeTexture2DFactoryNode` → `UTexture2D` 생성
- Factory Node는 어떤 `UObject`를 만들지, 그리고 어떤 설정을 적용할지 정의한다.

### 5. Payload

- 원본 데이터의 실제 바이트/버퍼를 제공한다.
- translator가 원본 데이터를 읽고 Payload Key를 등록하면, Import 단계에서 이 키를 통해 실제 데이터를 로딩한다.
- 예시:
    - PNG → 실제 픽셀 데이터(이미지 버퍼)
    - FBX → 메시의 버텍스/인덱스 버퍼
- Factory Node는 실행 시 Payload를 참조하여 최종 애셋을 생성한다.

### 6. Import

- Import Manager가 순차적으로 Factory를 실행하여 **UObject(에셋)**를 생성한다.
- 결과적으로 Content Browser에 실제 에셋(`UTexture2D`, `UStaticMesh`, `UMaterialInstanceConstant` 등)이 나타난다.

---

# Interchange 설정

**Project Settings - Engine - Interchange - Import Content - Content Import Settings - Pipeline Stacks**

![Interchange raw data 설명](/assets/images/data_image_interchange.png)

**스택 단위로 지정할 수 있는 Asset Pipeline은 1개**만 적용할 수 있다.

다만 필요하다면 **Translator별로 추가 파이프라인을 개별 지정** 할 수도 있다.

- 예를 들어, Texture Translator에는 텍스처 전용 파이프라인을, MaterialX Translator에는 MaterialX 전용 파이프라인을 붙여서 사용할 수 있다.

---

# Interchange 활용법

Interchange는 단순히 새로운 임포트 시스템을 넘어, **프로젝트 상황에 맞춰 파이프라인을 자동화**할 수 있다:

- **Interchange 자동화 활용**
    
    아티스트가 FBX나 PNG를 드래그 앤 드롭하면, Interchange Pipeline이 자동으로 규칙을 적용한다.
    
    - 예를 들어 노멀맵은 자동으로 sRGB가 꺼지고 BC5 압축으로 임포트되듯이, 다양한 텍스처 애셋들의 임포트 옵션을 원하는 대로 제어할 수 있다,
    - 또한 프로젝트가 모바일, PC, 콘솔 등 여러 플랫폼을 지원해야 할 경우, 플랫폼별 임포트 규칙을 나눠서 적용할 수도 있다.
        - 모바일: 텍스처 압축을 ETC2로 자동 변환
        - PC/콘솔: Nanite 자동 활성화
- **커스텀 데이터 기반 확장**
    
    커스텀 Translator를 작성하면, JSON/CSV 같은 사내 파이프라인 데이터를 읽어서 그대로 Interchange Node로 만들 수 있다.
    
    예를 들어 데이터테이블에 기록된 네이밍 규칙이나 머티리얼 매핑 정보를 Translator가 읽어 들여, Import 시점에 자동 적용하게 만들 수 있다.
    

정리하면, 
Interchange는 **자동화와 확장을 통해 프로젝트 워크플로우를 크게 개선**할 수 있는 강력한 도구이다.

---

# TA(Technical Artist)의 관점에서 본 Interchange

### 왜 TA가 Interchange에 손을 대게 되었는가

- **제작 파이프라인 구축 필요성**
    
    해외 개발 환경에서는 아티스트가 반복 작업이나 불필요한 수작업을 하지 못하도록 **제작 파이프라인을 먼저 세우는 문화**가 강하다는 인상을 받았다.
    
    이 과정을 겪으면서 나 역시 **자동화와 규칙화의 필요성**을 절실히 느꼈다.
    
- **휴먼 에러 방지**
    
    예를 들어 Material Instance에 파라미터 값을 직접 적용하다 보면, 사람이 하는 작업이니 작은 실수가 발생하기 쉽다.
    
    Interchange를 활용하면 이런 부분을 시스템적으로 보완할 수 있다.
    

### Interchange의 장점 (개인 의견)

- 프로젝트에 맞는 **임포트/익스포트 시스템을 손쉽게 구축**할 수 있다.
- 단순한 파이프라인은 **블루프린트만으로도 구현 가능**하기 때문에 접근성이 좋다.
- 데이터 변환/처리 과정을 **모듈 단위로 커스터마이즈**할 수 있어 확장성이 높다.

### Interchange의 단점  (개인 의견)

- 애셋 구조나 네이밍 규칙이 확정되지 않은 상태에서 시스템을 구축하면, **유지보수 비용**이 커질 수 있다.
- 블루프린트로 만들면 수정이 비교적 쉽지만, **C++로 작성한 경우에는 코드 수정 → 코드 리뷰 → 반영**까지 시간이 더 들어간다.
    - 그래서 나는 이 문제를 줄이기 위해 **네이밍 같은 규칙은 데이터 테이블로 관리**할 수 있도록 설계했다.

---

# 마무리

정리하자면, 

- **Interchange**는 언리얼 엔진의 새로운 임포트/익스포트 프레임워크
- 외부 데이터를 Node Graph라는 중간 표현으로 변환한 뒤, Pipeline과 Factory + Payload 단계를 거쳐 최종 에셋을 생성하는 구조
- 무엇보다 프로젝트 방향에 맞게 **커스터마이징**이 가능하다는 점이 큰 장점
- 간단한 자동화는 블루프린트로도 구현할 수 있고, 더 깊은 수준의 제어가 필요하다면 C++로 확장 가능

즉, **Interchange는 단순히 임포트 도구 같은 느낌이 아니라, 프로젝트 워크플로우와 파이프라인을 최적화할 수 있는 시스템**이라고 생각한다

# 레퍼런스

[언리얼 엔진의 인터체인지 프레임워크 | 언리얼 엔진 5.6 문서 | Epic Developer Community](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/interchange-framework-in-unreal-engine)

[인터체인지 개발 가이드 | 언리얼 엔진 5.6 문서 | Epic Developer Community](https://dev.epicgames.com/documentation/ko-kr/unreal-engine/interchange-development-guides)

[Import Customization through the Interchange Framework | Unreal Fest 2024](https://www.youtube.com/watch?v=Oq6KbrqkGnw)
