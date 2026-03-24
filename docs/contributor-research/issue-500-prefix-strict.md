# Issue issue 500: prefix 사용 시 strict 모드에서 "unused" 오류

## ✅ 상태: PR 제출 완료, 리뷰 대기 중

> **2026-03-23**: PR 517 제출. 이슈에 코멘트 완료. `./gradlew check` 통과 (consul 모듈 제외 — upstream 기존 문제). 리뷰 대기 중.

## 기본 정보

| 항목 | 내용 |
|------|------|
| 이슈 | issue 500 |
| 제목 | Loading a config with a prefix in strict mode leads to bogus "unused" error |
| 작성자 | `sschuberth` (ORT 프로젝트 메인테이너, 정기 기여자) |
| 작성일 | 2025-06-26 |
| 라벨 | `bug` (2025-12-09 추가) |
| PR | PR 517 (2026-03-23 제출, OPEN) |
| 모듈 | `hoplite-core` |

## 문제

`loadConfig`에서 `prefix` 파라미터와 `DecodeMode.Strict`를 함께 사용하면, prefix 키 자체가 "unused"로 보고된다.

```kotlin
val loader = ConfigLoaderBuilder.default()
  .addEnvironmentSource()
  .addPropertySources(sources)
  .withContextResolverMode(ContextResolverMode.SkipUnresolved)
  .withDecodeMode(DecodeMode.Strict)
  .build()

loader.loadConfig<OrtConfiguration>(prefix = "ort")
// → ConfigException: Config value 'ort' at (/home/.../.ort/config/config.yml:1:2) was unused
```

## 근본 원인

`ConfigParser.kt`의 `decode()` (라인 82-106)에서 prefix 처리 흐름:

```kotlin
return configResult
  .map { it.prefixedNode(prefix) }   // (1) prefix 하위 서브트리 추출
  .flatMap {
    val decoded = decoding.decode(kclass, it, decodeMode, context)
    val state = createDecodingState(it, context, secretsPolicy)  // (2) 서브트리 기준 unused 계산
    // ...
  }
```

`Decoding.kt`의 `createDecodingState()` (라인 33-42):

```kotlin
internal fun createDecodingState(root: Node, context: DecoderContext, ...): DecodingState {
  val (used, unused) = root.decodedPaths()
    .filterNot { it.path == DotPath.root }
    .partition { context.usedPaths.contains(it.path) || it.isClassDiscriminator(context) }
  return DecodingState(root, used, unused, ...)
}
```

**핵심:** `prefixedNode(prefix)`로 추출한 서브트리의 root node는 여전히 `ort`라는 path를 가지고 있다. `createDecodingState()`가 이 서브트리의 `decodedPaths()`를 순회할 때, 서브트리 root path인 `ort` 자체가 path 목록에 포함된다. 그런데 디코딩 과정에서 실제로 사용되는 건 `ort` 아래의 자식 노드들이므로, `ort` path는 `context.usedPaths`에 기록되지 않는다. 결과적으로 **prefixed subtree의 root path가 unused 후보에 남는다.**

prefix 없이 전체 트리를 디코딩할 때는 root가 `DotPath.root`라서 `filterNot { it.path == DotPath.root }`로 제외됨. 하지만 prefix로 서브트리를 꺼내면 서브트리 루트의 path가 `ort`(root가 아님)라서 필터에 걸리지 않음.

**핵심 파일:**
- `hoplite-core/src/main/kotlin/com/sksamuel/hoplite/internal/ConfigParser.kt` (라인 82-106, `decode`)
- `hoplite-core/src/main/kotlin/com/sksamuel/hoplite/internal/Decoding.kt` (라인 33-42, `createDecodingState`)

## 실제 수정 내용

`ConfigParser.kt`의 `decode()` 메서드에서, prefixed 노드를 추출한 직후 서브트리 root path를 used로 마킹:

```kotlin
.flatMap {
  // Mark the prefixed subtree root as used so strict mode does not report it as unused.
  // The path is read from the node itself (not the raw prefix string), which ensures
  // compatibility with PathNormalizer.
  if (prefix != null && it !is Undefined) {
    context.usedPaths.add(it.path)
  }
  val decoded = decoding.decode(kclass, it, decodeMode, context)
  // ...
}
```

**설계 포인트:**
- `it.path`를 사용 — `prefixedNode()`가 PathNormalizer 적용 후 반환하는 노드의 path이므로 정규화 호환
- `it !is Undefined` guard — 존재하지 않는 prefix일 때 `DotPath.root` 오염 방지
- subtree root path **만** 마킹 — `prefix.split(".")`으로 모든 상위 segment를 마킹하면 다른 bind에서 genuine unused를 가릴 위험

## 테스트

`PrefixTest.kt`에 5개 회귀 테스트 추가:

| 테스트 | 검증 대상 |
|--------|-----------|
| `strict mode should not report prefix key as unused` | 기본 재현 (issue 500 원본) |
| `strict mode with nested prefix` | `database.primary` 같은 다중 세그먼트 prefix |
| `strict mode should still detect genuinely unused keys` | strict이 진짜 unused는 여전히 잡는지 |
| `ConfigBinder with strict mode` | ConfigBinder로 다중 bind() 호출 시 동작 |
| `nonexistent prefix should not throw` | prefix가 없는 경로 → Undefined 안전성 |

## 빌드 결과

- `PrefixTest` 12/12 PASSED (기존 7 + 신규 5)
- `StrictModeTest` 5/5 PASSED
- `./gradlew check` BUILD SUCCESSFUL (hoplite-watch-consul 제외 — upstream에서도 동일 실패, 외부 Consul 바이너리 다운로드 문제)

## 머지 확률 평가

| 요소 | 평가 |
|------|------|
| 메인테이너 관심 | ⚠️ `bug` 라벨 부착됨 (2025-12-09). 직접 코멘트는 없음 |
| 경쟁자 | ✅ 없음 |
| 변경 범위 | ✅ production 1곳 (`ConfigParser.kt` 3줄) + 테스트 5개 |
| 유지보수자 우선순위 | ⚠️ 불명. 배치 머지 패턴 — 반응까지 수 주 소요 가능 |
| **종합** | **경쟁 낮음, 머지 가능성 있으나 반응 시점은 불확실** |

## 포트폴리오 가치

**중간** — 작은 버그 수정이지만, 첫 PR로 신뢰를 쌓기에 적합. sschuberth가 보고한 이슈를 수정하면 그와의 관계 형성에도 도움.

## 타임라인

| 날짜 | 이벤트 |
|------|--------|
| 2025-06-26 | sschuberth가 이슈 등록 |
| 2025-12-09 | sksamuel이 `bug` 라벨 추가 |
| 2026-03-23 | 이슈에 코멘트 남김 (원인 분석 + PR 링크) |
| 2026-03-23 | PR 517 제출 |
