# Issue #500: Prefix + strict mode bogus "unused" error

## 기본 정보

| 항목 | 값 |
|------|-----|
| 이슈 번호 | [#500](https://github.com/sksamuel/hoplite/issues/500) |
| 상태 | OPEN |
| 라벨 | `bug` |
| 작성자 | sschuberth (활발한 기여자) |
| 생성일 | 2025-12-09 |
| 코멘트 | 0건 |
| 관련 이슈 | #505 (strict mode + 환경변수) |

## 문제 요약

`prefix`를 사용하여 config를 로딩할 때 strict mode(`DecodeMode.Strict`)가 prefix 자체를 "unused" 값으로 잘못 보고한다.

### 재현 코드

```kotlin
val loader = ConfigLoaderBuilder.default()
    .addEnvironmentSource()
    .addPropertySources(sources)
    .withContextResolverMode(ContextResolverMode.SkipUnresolved)
    .withDecodeMode(DecodeMode.Strict)
    .build()

loader.loadConfig<OrtConfiguration>(prefix = "ort")
```

### 에러 메시지

```
ConfigException: Failed to load ORT configuration:
    Config value 'ort' at (/home/sebastian/.ort/config/config.yml:1:2) was unused
```

## 기술적 분석

### 근본 원인

`ConfigParser.decode()` (`internal/ConfigParser.kt:82-106`)에서 `prefixedNode(prefix)`로 서브트리를 추출하지만, 이후 `createDecodingState()`는 **추출 전의 전체 노드 트리**에서 unused를 계산한다. 따라서 prefix 노드(`ort`) 자체가 "사용되지 않은 값"으로 보고됨.

### 수정 방향

- `createDecodingState()` 호출 시 prefix 경로를 전달하여 해당 경로와 그 상위 노드를 unused 체크에서 제외
- 또는 `prefixedNode()` 적용 후의 서브트리만으로 unused를 계산하도록 변경

### 영향받는 코드 — 실제 진입점

| 파일 | 함수/위치 | 역할 |
|------|-----------|------|
| `hoplite-core/.../internal/ConfigParser.kt:82-106` | `decode()` | `prefixedNode(prefix)` 호출 후 `createDecodingState()` 호출 — **수정 지점** |
| `hoplite-core/.../internal/ConfigParser.kt:141-145` | `prefixedNode()` | prefix로 서브트리 추출 |
| `hoplite-core/.../internal/Decoding.kt:33-42` | `createDecodingState()` | unused 파티셔닝 — prefix 경로 필터 추가 필요 |
| `hoplite-core/.../internal/DecodeModeValidator.kt:21-26` | `ensureAllUsed()` | unused 리스트 기반 에러 반환 |

### 수정 후보 테스트

| 테스트 파일 위치 (신규 작성) | 검증 내용 |
|------------------------------|-----------|
| `hoplite-core/src/test/kotlin/.../StrictModePrefixTest.kt` | prefix 사용 시 prefix 노드가 unused로 잡히지 않는지 |

## 머지 확률 분석

| 요소 | 평가 |
|------|------|
| `bug` 라벨 | 메인테이너가 버그로 분류 |
| 재현 방법 명확 | 코드와 에러 메시지 제공 |
| 실제 프로젝트(ORT) 영향 | 실사용자 문제 |
| 수정 범위 작음 | `createDecodingState()`에 prefix 필터 추가 |
| **종합 판단** | **매우 높음 — bug 라벨 + 재현 명확 + 수정 범위 좁음** |

## 포트폴리오 가치

- Config prefix binding 메커니즘 이해 증명
- Strict mode validation 로직 수정 경험

## #505와의 관계

두 이슈 모두 strict mode의 unused 체크 로직 결함. 근본 원인이 다를 수 있지만:
- #500: prefix 노드 자체가 unused로 잡힘
- #505: 환경변수가 unused로 잡힘

하나의 PR에서 strict mode 검증 로직을 전체적으로 개선하면 두 이슈 동시 해결 가능.

## 실행 전략

1. #505와 함께 strict mode 코드 분석
2. prefix가 unused 체크에서 제외되는 테스트 케이스 작성
3. #505 수정과 동시에 진행하여 하나의 PR로 제출
4. PR 본문에 `Closes #500, Closes #505` 명시

## 리스크

- sschuberth가 직접 수정할 가능성 (ORT 프로젝트 메인테이너)
- #505와 별도 PR로 분리해야 할 수 있음 (메인테이너 판단)
