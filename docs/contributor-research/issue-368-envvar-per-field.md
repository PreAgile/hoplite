# Issue issue 368: 필드별 환경변수 매핑

## 🟡 상태: pinned, 설계 미확정

> 3년간 13개 코멘트. sksamuel이 `EnvVarMappedPropertySource` 방향을 제안했으나 아무도 구현하지 않음. 설계 합의 확인 후 구현 가능.

## 기본 정보

| 항목 | 내용 |
|------|------|
| 이슈 | issue 368 |
| 제목 | Allow changing environment variable per field |
| 작성자 | 외부 사용자 |
| 작성일 | 2023-04-26 |
| 라벨 | **pinned** |
| 클레임 | **없음** |
| 코멘트 | 13 (rocketraman, sksamuel, pschichtel 등) |
| 모듈 | `hoplite-core` |

## 문제

현재 hoplite는 환경변수를 자동 파생된 이름(`DATABASE_HOST`, `DATABASE_PORT`)으로만 매핑한다. Spring의 `@Value("${CUSTOM_ENV_NAME}")`처럼 임의의 환경변수명을 특정 config 필드에 매핑할 수 없다.

```kotlin
data class DbConfig(
  val host: String,      // DATABASE_HOST로만 매핑 가능
  val port: Int,         // DATABASE_PORT로만 매핑 가능
)

// 사용자가 원하는 것:
// PGHOST → host
// PGPORT → port
// (PostgreSQL 표준 환경변수)
```

## 설계 논의 요약

**sksamuel (메인테이너):**
- `EnvVarMappedPropertySource` 구현을 제안했으나 직접 작업하지 않음
- 복잡도가 높아 우선순위에서 밀림

**rocketraman (collaborator):**
- root 노드를 참조하는 방식으로 fallback 제안
- lazy node 평가가 필요하다고 sksamuel이 응답

**pschichtel:**
- 실제 구현이 어떤 모습인지 불확실하다고 코멘트

## 근본 원인

`EnvironmentVariablesPropertySource.kt` (라인 24-36):

```kotlin
override fun node(context: PropertySourceContext): ConfigResult<Node> {
  val map = environmentVariableMap()
    .filterKeys { if (prefix == null) true else it.startsWith(prefix) }
    .mapKeys { if (prefix == null) it.key else it.key.removePrefix(prefix) }

  return map.toNode("env", DELIMITER).transform { ... }.valid()
}
```

환경변수 → 노드 변환이 `DELIMITER` (`"_"`) 기반의 자동 매핑만 지원. 필드 수준의 커스텀 매핑 인터페이스가 존재하지 않음.

**핵심 파일:**
- `hoplite-core/src/main/kotlin/com/sksamuel/hoplite/sources/EnvironmentVariablesPropertySource.kt`
- `hoplite-core/src/main/kotlin/com/sksamuel/hoplite/ParameterMapper.kt`

## 수정 방향

### Option A: 새 PropertySource 구현

```kotlin
class MappedEnvVarPropertySource(
  private val mappings: Map<String, String>  // envVarName → configPath
) : PropertySource {
  override fun node(context: PropertySourceContext): ConfigResult<Node> {
    val entries = mappings.mapNotNull { (envVar, configPath) ->
      System.getenv(envVar)?.let { configPath to it }
    }
    return entries.toNode("mapped-env").valid()
  }
}

// 사용:
ConfigLoaderBuilder.default()
  .addPropertySource(MappedEnvVarPropertySource(
    mapOf("PGHOST" to "database.host", "PGPORT" to "database.port")
  ))
```

### Option B: annotation 기반

```kotlin
@EnvVar("PGHOST")
val host: String

@EnvVar("PGPORT")
val port: Int
```

annotation 기반은 ParameterMapper를 확장해야 하며 기존 아키텍처 변경이 큼.

### Option C (권장): A를 먼저, B를 후속 PR로

Option A는 기존 아키텍처에 부합하며 변경 범위가 좁음. 메인테이너가 제안한 `EnvVarMappedPropertySource`와 정확히 일치.

## 머지 확률 평가

| 요소 | 평가 |
|------|------|
| 메인테이너 관심 | ✅ **pinned** + sksamuel이 직접 구현 방향 제안 |
| 경쟁자 | ✅ 없음 (3년간 미구현) |
| 변경 범위 | ✅ 새 클래스 추가 (기존 코드 수정 최소) |
| 설계 합의 | ⚠️ 여러 방향이 논의되었으나 최종 합의 없음 |
| **종합** | **중간 (50-60%)** |

## 포트폴리오 가치

**매우 높음** — pinned 이슈, 3년간 13개 코멘트, 가장 많이 요청된 기능 중 하나. 이 이슈를 해결하면 hoplite 커뮤니티에서의 인지도가 크게 상승.

## 난이도

**중간** — PropertySource 인터페이스 이해 + 노드 트리 구성 방법 이해 필요.

## 리스크

- 설계 방향에 대한 메인테이너 합의가 확정되지 않음 → 구현 전 이슈에서 방향 확정 필수
- annotation 기반까지 가면 범위가 커짐 → PropertySource 방식만 먼저 제출
- 기존 EnvironmentVariablesPropertySource와의 우선순위/충돌 처리 필요
- **issue 505 (strict + env vars)와 연관** — 두 이슈를 동시에 인지하고 있어야 함

## PR 전략

1. 이슈에 코멘트 — sksamuel의 `EnvVarMappedPropertySource` 제안을 레퍼런스로 하여 구현 의사 표시
2. 설계 방향 합의 확인 (**합의 전 코딩 금지**)
3. `MappedEnvVarPropertySource` 클래스 + `ConfigLoaderBuilder` extension 함수 추가
4. 테스트: 매핑된 환경변수가 올바른 config 필드에 바인딩되는지 검증
5. README에 사용 예시 추가
