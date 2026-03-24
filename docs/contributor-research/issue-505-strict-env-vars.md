# Issue issue 505: strict() 모드에서 환경변수가 "unused"로 보고됨

## 🟡 상태: 설계 논의 완료, 구현 대기 중

> rocketraman이 sksamuel과 Slack에서 3단계 strictness 설계를 논의. 아직 아무도 구현하지 않음. 설계 합의 확인 후 구현 가능.

## 기본 정보

| 항목 | 내용 |
|------|------|
| 이슈 | issue 505 |
| 제목 | `strict()` mode fails because of environment variables |
| 작성자 | `TwoClocks` (외부) |
| 작성일 | 2025-10-14 |
| 라벨 | `bug` |
| 클레임 | **없음** |
| 코멘트 | 3 (rocketraman, TwoClocks, sksamuel 간접 참여) |
| 모듈 | `hoplite-core` |

## 문제

`strict()` 모드를 활성화하면, OS의 모든 환경변수(PATH, HOME, SHELL 등)가 "unused" 키로 감지되어 에러가 발생한다. 사실상 환경변수 PropertySource와 strict 모드를 함께 사용할 수 없는 상태.

```kotlin
ConfigLoaderBuilder.default()
  .strict()  // EnvironmentVariablesPropertySource가 기본 포함됨
  .build()
  .loadConfigOrThrow<MyConfig>()
// → 수백 개의 환경변수가 "unused" 에러로 보고됨
```

### 추가 보고 (2026-03-03)

TwoClocks가 v3.0.0.RC2에서도 여전히 발생 확인. sealed class 리스트 + HOCON substitution과 결합 시 추가 실패 패턴 보고.

## rocketraman의 설계 논의 (Slack 기반)

rocketraman이 sksamuel과 Slack에서 논의한 내용을 이슈에 공유. 3가지 strictness 유스케이스 정리:

1. **Decode errors only**: 디코딩 실패만 에러로 보고 (현재 Lenient와 동일)
2. **Unused from config files**: 명시적 config 파일의 미사용 키만 감지 (환경변수 제외)
3. **Unused from all sources**: 모든 소스의 미사용 키 감지 (현재 Strict)

## 근본 원인

`Decoding.kt`의 `createDecodingState()` (라인 33-42):

```kotlin
internal fun createDecodingState(
  root: Node,
  context: DecoderContext,
  secretsPolicy: SecretsPolicy?
): DecodingState {
  val (used, unused) = root.decodedPaths()
    .filterNot { it.path == DotPath.root }
    .partition { context.usedPaths.contains(it.path) || it.isClassDiscriminator(context) }
  return DecodingState(root, used, unused, createNodeStates(root, context, secretsPolicy))
}
```

`DecodeModeValidator.kt` (라인 14-26):

```kotlin
class DecodeModeValidator(private val mode: DecodeMode) {
  fun <A : Any> validate(a: A, state: DecodingState): ConfigResult<A> {
    return when (mode) {
      DecodeMode.Strict -> ensureAllUsed(state).map { a }
      DecodeMode.Lenient -> a.valid()
    }
  }

  private fun ensureAllUsed(result: DecodingState): ConfigResult<DecodingState> {
    return if (result.unused.isEmpty()) result.valid() else {
      val errors = NonEmptyList.unsafe(result.unused.map { ConfigFailure.UnusedPath(it) })
      ConfigFailure.MultipleFailures(errors).invalid()
    }
  }
}
```

**문제:** unused 키 계산 시 PropertySource 유형별 필터링이 없다. `EnvironmentVariablesPropertySource`가 가져오는 수백 개의 환경변수가 config 파일의 키와 동일한 수준으로 unused 체크에 포함됨.

**핵심 파일:**
- `hoplite-core/src/main/kotlin/com/sksamuel/hoplite/internal/Decoding.kt` (라인 33-42)
- `hoplite-core/src/main/kotlin/com/sksamuel/hoplite/internal/DecodeModeValidator.kt` (라인 14-26)
- `hoplite-core/src/main/kotlin/com/sksamuel/hoplite/ConfigLoaderBuilder.kt` (라인 445-450, defaultPropertySources)

## 수정 방향

### 접근법: DecodeMode를 3단계로 확장

```kotlin
enum class DecodeMode {
  Lenient,          // 기존: 에러 무시
  StrictFiles,      // 신규: config 파일의 미사용 키만 감지
  Strict,           // 기존: 모든 소스의 미사용 키 감지
}
```

`DecodeModeValidator`에서 `StrictFiles` 모드일 때 `PropertySource` 유형 정보를 기반으로 환경변수 키를 필터링:

```kotlin
DecodeMode.StrictFiles -> {
  val fileOnlyUnused = state.unused.filter { it.source?.isFileSource() == true }
  if (fileOnlyUnused.isEmpty()) a.valid()
  else ConfigFailure.MultipleFailures(
    NonEmptyList.unsafe(fileOnlyUnused.map { ConfigFailure.UnusedPath(it) })
  ).invalid()
}
```

이를 위해 `DecodedPath`에 source 정보를 추가해야 함. Node의 `pos` (Position) 필드에 이미 소스 정보가 포함되어 있으므로 이를 활용.

## 머지 확률 평가

| 요소 | 평가 |
|------|------|
| 메인테이너 관심 | ✅ rocketraman이 sksamuel과 설계 논의 완료 |
| 경쟁자 | ✅ 없음 (논의만 있고 구현 없음) |
| 변경 범위 | ⚠️ DecodeMode enum 변경 + Validator + Decoding |
| API 변경 | ⚠️ 기존 `strict()` 동작 변경 여부 합의 필요 |
| **종합** | **중상 (65-75%)** |

## 포트폴리오 가치

**높음** — 설계 논의에 참여하고 실제 구현까지 완료하는 패턴. 라이브러리 코어 이해 입증.

## 난이도

**중간** — DecodeMode, DecodeModeValidator, Decoding 세 파일의 상호작용 이해 필요. PropertySource별 소스 추적 메커니즘 이해도 필요.

## 리스크

- rocketraman/sksamuel의 설계 방향과 다른 구현을 하면 리젝될 수 있음
- `strict()` 기존 호출자의 기대 동작이 변경될 수 있음 → 기존 `strict()`를 `StrictFiles`로 매핑하고, 새 `strictAll()`을 추가하는 것이 안전
- sealed class + HOCON substitution 추가 버그(TwoClocks 보고)는 별도 이슈로 분리해야 할 수 있음

## PR 전략

1. 이슈에 코멘트 — rocketraman의 3단계 제안에 동의하며 구현 의사 표시
2. 설계 합의 확인 후 구현 시작 (**설계 합의 전 코딩 금지**)
3. `DecodeMode` enum 확장 + `DecodeModeValidator` 수정 + 빌더에 `strictFiles()` 메서드 추가
4. 테스트: 환경변수만 있는 경우, config 파일만 있는 경우, 혼합 경우 모두 커버
5. PR 본문에 rocketraman의 3단계 제안을 레퍼런스로 명시
