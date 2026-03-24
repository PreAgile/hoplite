# Issue issue 460: data class 기본값이 resolver 이후 적용되지 않음

## 🟡 상태: 부분 수정됨, 근본 원인 미해결

> rocketraman의 PR로 null/undefined 핸들링이 개선되었으나, resolver가 런타임에 Undefined를 반환하는 케이스는 여전히 미처리.

## 기본 정보

| 항목 | 내용 |
|------|------|
| 이슈 | issue 460 |
| 제목 | Data class default values don't apply after resolvers |
| 작성자 | `pschichtel` (외부) |
| 작성일 | 2024-11-05 |
| 라벨 | 없음 |
| 클레임 | **없음** |
| 코멘트 | 6 (pschichtel, rocketraman) |
| 관련 PR | rocketraman의 PR로 부분 수정 (DataClassDecoder null/undefined 핸들링 개선) |
| 모듈 | `hoplite-core` |

## 문제

resolver가 `Undefined` 또는 null을 반환하면, data class 생성자의 기본값으로 폴백하지 않고 디코딩 에러가 발생한다.

```kotlin
data class DbConfig(
  val host: String,
  val port: Int = 5432,  // 기본값 있음
  val password: String? = null
)

// 환경변수 resolver가 PASSWORD를 찾지 못해 Undefined 반환
// → 기대: password = null (기본값)
// → 실제: 디코딩 에러
```

pschichtel이 `NullElementRemovingResolver` 워크어라운드를 공유:

```kotlin
class NullElementRemovingResolver : Resolver {
  override suspend fun resolve(
    node: Node, root: Node, context: DecoderContext
  ): ConfigResult<Node> = when {
    node is NullNode -> Undefined.valid()
    else -> node.valid()
  }
}
```

## 근본 원인

`DataClassDecoder.kt` (라인 117-131)의 파라미터 디코딩 로직:

```kotlin
when {
  // optional이고 Undefined이면 스킵 → Kotlin이 기본값 사용
  param.isOptional && n is Undefined -> null
  else ->
    context.decoder(param).flatMap { decoder ->
      runBlocking {
        context.resolvers.resolve(n, param.name ?: "unknown", kclass, context)
          .flatMap { resolvedNode ->
            // ← resolvedNode가 Undefined/NullNode여도 여기서 디코딩 시도
            decoder.decode(resolvedNode, param.type, context).map { decoded ->
              Arg(param, usedName, decoded, n)
            }
          }
      }
    }
}
```

**문제:** `param.isOptional && n is Undefined` 체크는 resolver 실행 **전**에 수행된다. resolver가 유효한 노드를 받아서 Undefined/NullNode로 변환하는 경우, 이 체크를 통과한 후 decoder가 null을 디코딩하려다 실패한다.

rocketraman의 PR (커밋 87cc90b)이 부분 수정했으나, resolver가 **런타임에** Undefined를 반환하는 케이스는 여전히 미처리.

**핵심 파일:**
- `hoplite-core/src/main/kotlin/com/sksamuel/hoplite/decoder/DataClassDecoder.kt` (라인 40-46, 117-131)

## 수정 방향

resolver 실행 **후**에 optional + Undefined 체크를 추가:

```kotlin
context.resolvers.resolve(n, param.name ?: "unknown", kclass, context)
  .flatMap { resolvedNode ->
    // resolver 실행 후 Undefined/NullNode 체크 추가
    if ((resolvedNode is Undefined || resolvedNode is NullNode) && param.isOptional) {
      return@flatMap null  // mapNotNull에서 제외 → Kotlin 기본값 사용
    }
    decoder.decode(resolvedNode, param.type, context).map { decoded ->
      Arg(param, usedName, decoded, n)
    }
  }
```

## 머지 확률 평가

| 요소 | 평가 |
|------|------|
| 메인테이너 관심 | ⚠️ rocketraman이 부분 수정 PR 제출 이력 |
| 경쟁자 | ✅ 없음 |
| 변경 범위 | ✅ DataClassDecoder 한 곳 |
| 기존 PR 관계 | ⚠️ rocketraman의 수정을 확장하는 형태 — 사전 소통 권장 |
| **종합** | **중간 (60-70%)** |

## 포트폴리오 가치

**높음** — DataClassDecoder는 hoplite의 핵심 디코딩 엔진. 이 영역의 수정은 내부 구조에 대한 깊은 이해를 입증.

## 난이도

**중간** — DataClassDecoder의 파라미터 디코딩 흐름, Resolver 파이프라인, Kotlin 생성자 기본값 메커니즘 이해 필요.

## 리스크

- resolver가 의도적으로 null을 반환하는 케이스(null이 "값"인 경우)와 "값이 없음"을 구분해야 함
- `NullNode` vs `Undefined` 시맨틱 구분이 명확하지 않을 수 있음
- rocketraman의 기존 수정과 충돌하지 않도록 주의
- non-optional 파라미터 + resolver null 반환 시의 동작도 정의 필요

## PR 전략

1. 이슈에 코멘트 — pschichtel의 워크어라운드를 참조하며 근본 수정 제안
2. rocketraman에게 멘션하여 기존 수정과의 호환성 확인
3. `DataClassDecoder`에서 resolver 후 optional 체크 추가
4. 테스트: resolver null 반환 + optional 파라미터, resolver null 반환 + non-optional 파라미터, 정상 케이스 모두 커버
