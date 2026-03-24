# Issue issue 503: PathNormalizer 소문자 변환 회귀

## 🟢 상태: OPEN — 미클레임, 경쟁 없음

> 코멘트 0개, 8개월간 아무도 작업하지 않음. 명확한 회귀 버그.

## 기본 정보

| 항목 | 내용 |
|------|------|
| 이슈 | issue 503 |
| 제목 | Breaking change from 2.7.5 to 2.8.0 (PathNormalizer lowercases keys) |
| 작성자 | 외부 사용자 |
| 작성일 | 2025-07-22 |
| 라벨 | 없음 |
| 클레임 | **없음** (8개월간 미클레임) |
| 코멘트 | 0 |
| 모듈 | `hoplite-core` |

## 문제

2.7.5에서 2.8.0으로 업그레이드하면 `PathNormalizer`가 모든 키를 소문자로 변환하여, `LookupPreprocessor`의 대소문자 구분 키 참조가 깨진다.

```yaml
# config.yaml
database:
  host: localhost
  connectionString: "jdbc:postgresql://{{database.host}}:5432/mydb"
```

2.7.5에서는 `{{database.host}}`가 정상 치환되지만, 2.8.0에서는 `PathNormalizer`가 키를 `connectionstring`으로 변환하면서 lookup 참조가 실패한다.

## 근본 원인

`PathNormalizer.kt` (라인 23-27)에서 무조건적 소문자 변환:

```kotlin
object PathNormalizer : NodeTransformer {
  override fun transformPathElement(element: String): String = element
    .replace("-", "")
    .replace("_", "")
    .lowercase()  // ← 모든 키를 무조건 소문자로 변환
```

`LookupPreprocessor.kt` (라인 19-33)는 `node.atPath(key)`로 대소문자 구분 lookup을 수행:

```kotlin
object LookupPreprocessor : Preprocessor {
  private val regex1 = "\\$\\{(.*?)\\}".toRegex()
  private val regex2 = "\\{\\{(.*?)\\}\\}".toRegex()

  override fun process(node: Node, context: DecoderContext): ConfigResult<Node> {
    fun lookup(key: String): String? = when (val n = node.atPath(key)) {
      is StringNode -> n.value  // 대소문자 구분 lookup
      else -> null
    }
    // ...
  }
}
```

**핵심 파일:**
- `hoplite-core/src/main/kotlin/com/sksamuel/hoplite/transformer/PathNormalizer.kt` (라인 23-55)
- `hoplite-core/src/main/kotlin/com/sksamuel/hoplite/preprocessor/LookupPreprocessor.kt` (라인 19-33)

## 수정 방향

### Option A (권장): LookupPreprocessor에서 정규화된 키로 lookup

`LookupPreprocessor`가 lookup할 때 `PathNormalizer.transformPathElement()`를 통해 키를 정규화한 후 검색:

```kotlin
fun lookup(key: String): String? {
  val normalizedKey = key.split(".").joinToString(".") { segment ->
    PathNormalizer.transformPathElement(segment)
  }
  return when (val n = node.atPath(normalizedKey)) {
    is StringNode -> n.value
    else -> null
  }
}
```

### Option B: PathNormalizer에서 원본 키도 보존

MapNode의 키를 정규화할 때 원본 키를 별도 필드에 보존하여 lookup 시 양쪽 모두 검색 가능하게 함. 변경 범위가 넓어 비추천.

### Option C: 변환 순서 조정

NodeTransformer 적용 전에 Preprocessor를 실행하도록 파이프라인 순서 변경. 기존 동작에 영향을 줄 수 있어 주의 필요.

## 머지 확률 평가

| 요소 | 평가 |
|------|------|
| 메인테이너 관심 | ⚠️ 직접 반응 없음 |
| 경쟁자 | ✅ 없음 (8개월간 코멘트 0) |
| 변경 범위 | ✅ 좁음 |
| 회귀 버그 | ✅ breaking change이므로 수정 동기 명확 |
| **종합** | **높음 (80-85%)** |

## 포트폴리오 가치

**중간** — 회귀 버그 수정으로 안정성 기여 입증.

## 난이도

**낮음~중간** — NodeTransformer와 Preprocessor의 실행 순서 파이프라인 이해 필요.

## 리스크

- Option A 적용 시 `PathNormalizer`가 비활성화된 환경에서 lookup 동작이 달라질 수 있음
- 정규화 로직이 sealed class discriminator 필드 예외 처리(`normalizePathElementExceptDiscriminator`)와 상호작용할 수 있음
- 테스트에서 PathNormalizer 활성/비활성 양쪽 시나리오를 모두 커버해야 함

## PR 전략

1. 이슈에 코멘트 — 재현 확인 및 Option A 접근 제안
2. `LookupPreprocessor`에서 정규화 적용 후 lookup하도록 수정
3. 테스트: `PathNormalizer` 활성화 상태에서 `{{camelCase.key}}` 치환이 동작하는지 검증
4. PR 본문에 2.7.5 vs 2.8.0 동작 차이 명시
