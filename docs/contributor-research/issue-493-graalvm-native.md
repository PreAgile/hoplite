# Issue issue 493: GraalVM native image에서 KClass 체크 실패

## 🟡 상태: 활발한 조사 중, PR 없음

> sschuberth, sgammon이 조사 중이나 수정 PR은 없음. 2026-03-04 기준 v3.0.0.RC2에서도 여전히 발생 확인. 기여 시 협력 제안 필수.

## 기본 정보

| 항목 | 내용 |
|------|------|
| 이슈 | issue 493 |
| 제목 | GraalVM native image does not run due to failing KClass check |
| 작성자 | `sschuberth` (ORT 프로젝트 메인테이너) |
| 작성일 | 2025-04-05 |
| 라벨 | 없음 |
| 클레임 | **없음** (sschuberth, sgammon이 조사 중이나 수정 PR은 없음) |
| 코멘트 | 7 |
| 관련 이슈 | issue 484 (reachability metadata 기여) |
| 모듈 | `hoplite-core` |

## 문제

GraalVM native image로 빌드된 애플리케이션에서 hoplite의 config 로딩이 실패한다.

```
java.lang.IllegalArgumentException: Only instances of KClass are supported [was ???]
```

`DecoderRegistry.kt`의 `require(type.classifier is KClass<*>)` 체크에서 실패. native image 환경에서 Kotlin 리플렉션의 `type.classifier`가 유효한 `KClass`가 아닌 `???`를 반환.

### 최신 상태 (2026-03-04)

sschuberth 확인: v3.0.0.RC2에서도 여전히 발생. 스택트레이스에서 `???` 타입이 출력됨.

## 근본 원인

`DecoderRegistry.kt` (라인 51):

```kotlin
override fun decoder(type: KType): ConfigResult<Decoder<*>> {
  require(decoders.isNotEmpty()) { "Cannot find decoder in empty decoder registry" }
  require(type.classifier is KClass<*>) {
    "Only instances of KClass are supported [was ${type.classifier ?: type}]"
  }
  val filteredDecoders = decoders.filter { it.supports(type) }
  // ...
}
```

GraalVM native image에서는 AOT(Ahead-of-Time) 컴파일 과정에서 리플렉션 메타데이터가 제거된다. Kotlin의 `KType.classifier`는 런타임 리플렉션에 의존하므로, reflect-config.json에 등록되지 않은 타입은 `classifier`가 유효하지 않은 값을 반환.

**핵심 파일:**
- `hoplite-core/src/main/kotlin/com/sksamuel/hoplite/decoder/DecoderRegistry.kt` (라인 51)
- `example-native/` 모듈 (GraalVM 예제)

## 수정 방향

### Option A: 안전한 폴백 + reflect-config 보강

```kotlin
override fun decoder(type: KType): ConfigResult<Decoder<*>> {
  require(decoders.isNotEmpty()) { "Cannot find decoder in empty decoder registry" }
  val classifier = type.classifier
  if (classifier !is KClass<*>) {
    return ConfigFailure.Generic(
      "Type classifier is not a KClass: ${classifier ?: type}. " +
      "If running in GraalVM native image, ensure reflect-config.json includes this type."
    ).invalid()
  }
  // ...
}
```

### Option B: GraalVM reachability metadata 기여 (issue 484)

Oracle의 [graalvm-reachability-metadata](https://github.com/oracle/graalvm-reachability-metadata) 저장소에 hoplite용 메타데이터를 기여. 등록되면 GraalVM 빌드 시 자동으로 리플렉션 설정 적용.

### Option C (권장): A + B 동시 진행

코드 수정으로 에러 메시지를 개선하고, metadata 기여로 근본 해결.

## 머지 확률 평가

| 요소 | 평가 |
|------|------|
| 메인테이너 관심 | ⚠️ 직접 반응 없으나 example-native 모듈 존재 = GraalVM 지원 의지 |
| 경쟁자 | ⚠️ sschuberth, sgammon이 조사 중이나 PR은 없음 |
| 변경 범위 | ✅ Option A는 DecoderRegistry 한 곳 |
| GraalVM 전문성 | ⚠️ native image 디버깅 경험 필요 |
| **종합** | **중간 (50-60%)** |

## 포트폴리오 가치

**매우 높음** — GraalVM + Kotlin 생태계 전체에 영향. 이 수정이 머지되면 Kotlin/GraalVM 커뮤니티에서의 가시성이 매우 높음.

## 난이도

**높음** — GraalVM native image의 리플렉션 제한 이해, reflect-config.json 작성, native image 빌드 및 테스트 환경 필요.

## 리스크

- GraalVM 버전별로 동작이 다를 수 있음
- `example-native` 모듈의 기존 설정이 불완전할 수 있음
- sschuberth가 이미 조사 중이므로 중복 작업 가능성 → 이슈에 코멘트로 협력 제안 필요
- reflect-config.json에 모든 사용자 정의 data class를 등록해야 하는 제한은 hoplite 수준에서 해결 불가

## PR 전략

1. 이슈에 코멘트 — sschuberth에게 현재 조사 상태 확인 + 협력 제안
2. **Option A**: DecoderRegistry의 require를 ConfigFailure 반환으로 변경 (graceful degradation)
3. **Option B**: 별도로 oracle/graalvm-reachability-metadata에 hoplite 메타데이터 PR
4. 테스트: native image 빌드가 CI에서 어려우므로, unit test로는 classifier가 null인 케이스를 시뮬레이션
