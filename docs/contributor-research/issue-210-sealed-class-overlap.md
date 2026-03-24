# Issue issue 210: sealed class 감지 오류 (파라미터 오버라이드)

## 🟡 상태: pinned, 5년간 미해결

> 2021년 등록, pinned. meierjan의 PR로 discriminator 방식은 수정되었으나 파라미터 오버랩 근본 문제는 미해결. 신뢰 쌓은 후 도전 권장.

## 기본 정보

| 항목 | 내용 |
|------|------|
| 이슈 | issue 210 |
| 제목 | Sealed class detection is buggy when parameters are overridden |
| 작성자 | 외부 사용자 |
| 작성일 | 2021-05-14 |
| 라벨 | **pinned** |
| 클레임 | **없음** |
| 코멘트 | 7 (다양한 재현 케이스 보고) |
| 관련 PR | meierjan의 PR로 class discriminator 방식은 수정됨 |
| 모듈 | `hoplite-core` |

## 문제

sealed class 역직렬화 시, 서브클래스들의 파라미터 이름이 겹치면 잘못된 서브클래스가 선택된다.

```kotlin
sealed class DataSourceConfig {
  data class Postgres(val host: String, val port: Int, val database: String) : DataSourceConfig()
  data class MySql(val host: String, val port: Int, val schema: String) : DataSourceConfig()
}
```

```yaml
datasource:
  host: localhost
  port: 5432
  database: mydb  # Postgres를 의도
```

`host`와 `port`가 두 서브클래스 모두에 존재하므로, 매칭 알고리즘이 잘못된 서브클래스(MySql)를 선택할 수 있다.

## 근본 원인

`SealedClassDecoder.kt`의 `deriveInstance` 함수 (라인 109-120):

```kotlin
val results = kclass.sealedSubclasses
  .filter { subclass ->
    subclass hasConstructorsWithArgumentsNumberLessOrEqualTo node.expectedNumberOfConstructorArguments
  }
  .sortedWith { subclass1, subclass2 ->
    (
      subclass1.numberOfMandatoryConstructorArguments
        .compareTo(subclass2.numberOfMandatoryConstructorArguments)
        .takeUnless { it == 0 }
        ?: subclass1.numberOfTotalConstructorArguments
          .compareTo(subclass2.numberOfTotalConstructorArguments)
    ) * -1
  }
  .map { DataClassDecoder().decode(node, it.createType(), context) }
```

**문제:** 서브클래스 선택이 **생성자 인자 수**로만 필터/정렬된다. 파라미터 **이름** 매칭은 수행하지 않는다. 인자 수가 같은 서브클래스들은 선언 순서에 따라 순차 디코딩을 시도하고, 첫 번째 성공 결과를 반환한다.

`host`, `port`처럼 공통 파라미터가 있으면, 의도하지 않은 서브클래스도 디코딩에 성공할 수 있다. `database`(Postgres 전용)나 `schema`(MySql 전용)의 존재 여부로 구분해야 하지만, 현재 알고리즘은 이를 고려하지 않는다.

meierjan의 PR (sealed class discriminator)은 명시적 `_type` 필드로 구분하는 방식을 추가했으나, discriminator 없이 파라미터 이름만으로 자동 매칭하는 근본 문제는 미해결.

**핵심 파일:**
- `hoplite-core/src/main/kotlin/com/sksamuel/hoplite/decoder/SealedClassDecoder.kt` (라인 109-120)

## 수정 방향

### Option A (권장): 파라미터 이름 매칭 스코어 도입

```kotlin
val results = kclass.sealedSubclasses
  .map { subclass ->
    val params = subclass.primaryConstructor?.parameters?.map { it.name } ?: emptyList()
    val nodeKeys = (node as? MapNode)?.map?.keys ?: emptySet()
    val matchScore = params.count { it in nodeKeys }
    val mismatchCount = params.filter { !it.isOptional }.count { it !in nodeKeys }
    Triple(subclass, matchScore, mismatchCount)
  }
  .filter { (_, _, mismatch) -> mismatch == 0 }  // 필수 파라미터가 모두 있는 것만
  .sortedByDescending { (_, score, _) -> score }  // 매칭 점수 높은 순
  .map { (subclass, _, _) ->
    DataClassDecoder().decode(node, subclass.createType(), context)
  }
```

### Option B: discriminator 필수화

파라미터 오버랩이 감지되면 discriminator 필드를 요구하도록 에러 메시지를 개선. 자동 매칭의 복잡성을 회피.

## 머지 확률 평가

| 요소 | 평가 |
|------|------|
| 메인테이너 관심 | ✅ **pinned** (5년간 유지) |
| 경쟁자 | ✅ 없음 |
| 변경 범위 | ⚠️ SealedClassDecoder 핵심 알고리즘 변경 |
| 하위 호환성 | ⚠️ 기존 매칭 순서가 변경될 수 있음 |
| **종합** | **중하 (40-50%)** |

## 포트폴리오 가치

**높음** — pinned 이슈, 5년간 미해결. 이 문제를 해결하면 라이브러리 핵심 로직에 대한 깊은 이해를 입증. 다만 하위 호환성 리스크로 리젝 가능성도 있음.

## 난이도

**높음** — sealed class 디코딩 파이프라인 전체 이해 필요. DataClassDecoder와의 상호작용, discriminator 메커니즘, edge case(중첩 sealed class, 3개 이상 서브클래스 등) 처리 필요.

## 리스크

- 파라미터 이름 매칭이 기존의 인자 수 기반 매칭보다 **느릴 수 있음**
- 기존에 "우연히 올바르게 동작하던" config가 깨질 수 있음
- optional 파라미터가 있는 서브클래스와의 매칭 우선순위 복잡
- 다중 상속 구조 (sealed class 안의 sealed class)에서의 동작 정의 필요
- **첫 기여로는 부담** — issue 500, issue 503 등으로 신뢰 쌓은 후 도전 권장

## PR 전략

1. 이슈에 코멘트 — 파라미터 이름 매칭 스코어 접근법 제안, 메인테이너 반응 확인
2. **설계 합의 전 코딩 금지** — 5년간 미해결인 이유가 설계 복잡성에 있을 가능성 높음
3. 기존 테스트 스위트를 모두 통과하면서 오버랩 케이스를 추가하는 방식으로 진행
4. discriminator 우선, 자동 매칭은 폴백으로 설계하면 메인테이너 수용 가능성 높아짐
5. PR 규모가 커지면 "파라미터 이름 매칭 추가"와 "에러 메시지 개선"을 별도 PR로 분리
