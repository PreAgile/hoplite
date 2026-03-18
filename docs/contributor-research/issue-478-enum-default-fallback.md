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

`EnumDecoder.safeDecode()` (`decoder/enum.kt:28-45`)에서 `klass.java.enumConstants.find{}`로 매칭하고, 없으면 `ConfigFailure.InvalidEnumConstant(node, type, value).invalid()`를 반환한다.

**실제 코드 (enum.kt:28-38):**
```kotlin
fun decode(value: String): ConfigResult<T> {
  val t = klass.java.enumConstants.find {
    it.toString().contentEquals(other = value, ignoreCase = context.config.resolveTypesCaseInsensitive)
  }
  return if (t == null)
    ConfigFailure.InvalidEnumConstant(node, type, value).invalid()
  else
    (t as T).valid()
}
```

수정 지점은 `t == null` 분기에서 data class의 Kotlin default value로 fallback하는 로직을 추가하는 것.

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

### 영향받는 코드 — 실제 진입점

| 파일 | 함수/위치 | 역할 |
|------|-----------|------|
| `hoplite-core/.../decoder/enum.kt:17-46` | `EnumDecoder` 전체 | enum 디코딩 로직 |
| `hoplite-core/.../decoder/enum.kt:28-38` | `decode()` 내부 함수 | `enumConstants.find{}` → null일 때 에러 반환 — **수정 지점** |
| `hoplite-core/.../ConfigFailure.kt` | `InvalidEnumConstant` | 에러 메시지 생성 |

### 고려사항

- 기존 `@ConfigProperty` 어노테이션이 있는지 확인 필요
- Kotlin data class의 기본값과의 상호작용
- strict mode에서의 동작 (유효하지 않은 enum을 허용할지)

## 머지 확률 분석

| 요소 | 평가 |
|------|------|
| 명확한 유즈케이스 | 새 enum 값이 추가되기 전 하위호환 |
| 메인테이너 미응답 | 관심도 불확실 — 유일한 리스크 |
| 기존 Decoder 패턴 따름 | 구현 난이도 적정 |
| 코멘트 0건 | 커뮤니티 관심 낮음 |
| **종합 판단** | **높음 — 유즈케이스 명확하나, 메인테이너 미응답이 변수. 이슈 코멘트로 방향 확인 필수** |

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
