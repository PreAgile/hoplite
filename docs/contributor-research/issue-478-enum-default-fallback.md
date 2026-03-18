# Issue #478: Support optional default enum value when invalid value is provided

## 기본 정보

| 항목 | 값 |
|------|-----|
| 이슈 번호 | [#478](https://github.com/sksamuel/hoplite/issues/478) |
| 상태 | OPEN |
| 라벨 | 없음 |
| 작성자 | tbcrawford |
| 생성일 | 2025-02-25 |
| 코멘트 | 0건 |

## 문제 요약

현재 hoplite는 config에서 enum 값이 유효하지 않으면 에러를 던진다. 요청자는 유효하지 않은 enum 값이 들어올 때 기본값으로 fallback할 수 있는 옵션을 원함.

### 요청된 API 예시

```kotlin
enum class Colors {
    Red, Blue, Green, Unknown
}

data class Branding(
    @ConfigProperty(default = Colors.Unknown)  // 유효하지 않은 값 → Unknown
    val bgColor: Colors,
)
```

```yaml
branding:
  bgColor: "Yellow"  # Yellow는 Colors에 없지만, Unknown으로 fallback
```

## 기술적 분석

### 현재 동작

`EnumDecoder`가 `valueOf()`를 호출하고 실패하면 `ConfigFailure`를 반환.

### 구현 방향 (제안)

**옵션 1: 어노테이션 기반**
```kotlin
@DefaultEnumValue(Colors.Unknown)
val bgColor: Colors
```

**옵션 2: 기존 Kotlin default value 활용**
```kotlin
data class Branding(
    val bgColor: Colors = Colors.Unknown  // Kotlin default로 처리
)
```

**옵션 3: EnumDecoder에 글로벌 설정 추가**
```kotlin
ConfigLoaderBuilder.default()
    .withEnumFallback(true)  // enum 실패 시 data class default 사용
    .build()
```

### 고려사항

- 기존 `@ConfigProperty` 어노테이션이 있는지 확인 필요
- Kotlin data class의 기본값과의 상호작용
- strict mode에서의 동작 (유효하지 않은 enum을 허용할지)

## 머지 확률 분석

| 요소 | 평가 |
|------|------|
| 명확한 유즈케이스 | 새 enum 값이 추가되기 전 하위호환 |
| 메인테이너 미응답 | 관심도 불확실 |
| 기존 Decoder 패턴 따름 | 구현 난이도 적정 |
| 코멘트 0건 | 커뮤니티 관심 낮음 |
| **종합 머지 확률** | **85%** |

## 포트폴리오 가치

- Type-safe config 설계 능력
- Decoder 아키텍처 이해
- 하위호환성 고려한 기능 설계

## 실행 전략

1. **이슈에 설계 제안 코멘트** 먼저 (메인테이너 의견 확인)
2. 승인 방향이 나오면 `EnumDecoder` 수정
3. 테스트 케이스: 유효/무효 enum 값, default 있는/없는 경우
4. README에 사용 예시 추가

## 리스크

- 메인테이너가 설계 방향에 대해 다른 의견일 수 있음
- Kotlin default value로 이미 해결 가능하다고 판단하여 닫을 수 있음
- 이슈에 먼저 코멘트하여 방향 확인 필수
