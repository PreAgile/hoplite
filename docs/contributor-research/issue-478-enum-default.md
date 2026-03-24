# Issue issue 478: 잘못된 enum 값에 대한 기본값 지원

## 🟢 상태: OPEN — 미클레임, 경쟁 없음

> 코멘트 0개, 13개월간 아무도 작업하지 않음. 깔끔한 기능 추가 후보.

## 기본 정보

| 항목 | 내용 |
|------|------|
| 이슈 | issue 478 |
| 제목 | Support optional default enum value when invalid value is provided |
| 작성자 | 외부 사용자 |
| 작성일 | 2025-02-25 |
| 라벨 | 없음 |
| 클레임 | **없음** (13개월간 미클레임) |
| 코멘트 | 0 |
| 모듈 | `hoplite-core` |

## 문제

config 파일에 잘못된 enum 값이 들어오면 예외가 발생한다. 사용자는 예외 대신 지정된 기본값으로 폴백하는 옵션을 원한다.

```kotlin
enum class LogLevel { DEBUG, INFO, WARN, ERROR }

data class AppConfig(
  val logLevel: LogLevel = LogLevel.INFO  // data class 기본값은 있지만...
)
```

```yaml
# config.yaml
logLevel: VERBOSE  # 잘못된 값 → 현재는 예외 발생
```

사용자가 원하는 동작: `VERBOSE`가 유효한 enum 값이 아니므로 `LogLevel.INFO` 기본값으로 폴백.

## 근본 원인

`EnumDecoder`는 `java.lang.Enum.valueOf()`를 사용하여 문자열을 enum으로 변환하며, 매칭 실패 시 `ConfigFailure`를 즉시 반환한다. data class 생성자의 기본값으로 폴백하는 경로가 없다.

**핵심 파일:**
- `hoplite-core/src/main/kotlin/com/sksamuel/hoplite/decoder/EnumDecoder.kt`

## 수정 방향

### Option A: annotation 기반 기본값

```kotlin
@DefaultOnInvalid
enum class LogLevel { DEBUG, INFO, WARN, ERROR;
  companion object { val default = INFO }
}
```

### Option B: `ConfigLoaderBuilder` 옵션

```kotlin
ConfigLoaderBuilder.default()
  .withEnumFallback(true)  // 잘못된 enum → data class 기본값 사용
  .build()
```

### Option C (권장): `EnumDecoder`에서 실패 시 Undefined 반환

enum 디코딩 실패 시 `ConfigFailure` 대신 `Undefined`를 반환하여, `DataClassDecoder`의 기본값 폴백 로직이 작동하도록 함. 가장 기존 아키텍처에 부합하는 접근.

## 머지 확률 평가

| 요소 | 평가 |
|------|------|
| 메인테이너 관심 | ⚠️ 반응 없음 |
| 경쟁자 | ✅ 없음 |
| 변경 범위 | ✅ 좁음 (EnumDecoder 한 곳) |
| 기존 동작 변경 | ⚠️ breaking change가 될 수 있어 opt-in 방식 필요 |
| **종합** | **높음 (75-80%)** |

## 포트폴리오 가치

**중간** — 깔끔한 기능 추가, 디코더 아키텍처 이해 입증.

## 난이도

**낮음** — EnumDecoder의 에러 처리 분기만 수정.

## 리스크

- 기본 동작을 바꾸면 기존 사용자에게 영향 → opt-in 옵션으로 구현해야 함
- nullable enum 타입과의 상호작용 확인 필요
- data class 기본값이 없는 경우의 폴백 동작 정의 필요

## PR 전략

1. 이슈에 코멘트 — Option C 접근법 제안, 메인테이너 반응 확인
2. `ConfigLoaderBuilder`에 `withEnumFallbackToDefault(Boolean)` 옵션 추가
3. `EnumDecoder`에서 옵션 활성화 시 매칭 실패를 `Undefined`로 반환
4. 테스트: 유효/무효 enum 값 + 기본값 유무 조합 커버
