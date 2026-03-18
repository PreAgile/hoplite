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

strict mode의 unused 체크가 prefix로 사용된 루트 노드를 "사용되지 않은 값"으로 오인한다. prefix는 config 트리를 루팅하기 위한 용도인데, 검증 로직에서 이를 고려하지 않음.

### 수정 방향

- strict mode의 unused 체크 시 `prefix` 파라미터로 전달된 경로는 제외
- prefix 하위 노드만 unused 체크 대상으로

### 영향받는 코드 (추정)

- `ConfigParser.decode()` 또는 `Decoding.decode()` 내 strict 검증
- prefix 기반 노드 필터링 로직

## 머지 확률 분석

| 요소 | 평가 |
|------|------|
| `bug` 라벨 | 명확한 버그 |
| 재현 방법 명확 | 코드와 에러 메시지 제공 |
| 실제 프로젝트(ORT) 영향 | 실사용자 문제 |
| 수정 범위 작음 | prefix 필터링 로직 추가 |
| **종합 머지 확률** | **95%** |

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
