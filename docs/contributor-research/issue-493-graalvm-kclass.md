# Issue #493: GraalVM native image does not run due to failing KClass check

## 기본 정보

| 항목 | 값 |
|------|-----|
| 이슈 번호 | [#493](https://github.com/sksamuel/hoplite/issues/493) |
| 상태 | OPEN |
| 라벨 | 없음 |
| 작성자 | sschuberth (ORT 프로젝트 메인테이너) |
| 생성일 | 2026-03-04 |
| 코멘트 | 7건 (sschuberth, sgammon, sksamuel) |
| 관련 이슈 | #484 (GraalVM reachability metadata) |

## 문제 요약

GraalVM native image로 빌드된 애플리케이션에서 hoplite가 런타임 에러를 발생시킨다. `DecoderRegistry.decoder()`의 `type.classifier`가 KClass 인스턴스가 아닌 `???`로 나타남.

### 에러 스택트레이스

```
java.lang.IllegalArgumentException: Only instances of KClass are supported
    [was org.ossreviewtoolkit.model.config.ProviderPluginConfiguration]
    at com.sksamuel.hoplite.decoder.DefaultDecoderRegistry.decoder(DecoderRegistry.kt:51)
    at com.sksamuel.hoplite.DecoderContext.decoder(DecoderContext.kt:45)
    at com.sksamuel.hoplite.decoder.ListDecoder.safeDecode(ListDecoder.kt:46)
    ...
```

### 근본 원인

`DecoderRegistry.kt:51`의 KClass 타입 체크:
```kotlin
// type.classifier가 GraalVM native image에서 KClass가 아닌 다른 타입으로 나타남
```

GraalVM은 reflection을 AOT(Ahead-of-Time) 등록 방식으로 처리. Kotlin의 `KClass`가 reflection config에 등록되지 않으면 `type.classifier`가 정상 작동하지 않음.

## 커뮤니티 논의

**sgammon** (GraalVM 전문가):
- Native Image Agent로 reflection config 자동 생성 제안
- `build/native/agent-output/{task}`에 생성되는 config를 JAR 리소스에 포함 권장
- `--trace-object-instantiation` 플래그로 디버깅 가능

**sschuberth**:
- `example-native/META-INF/native-image/generated/reflect-config.json` 이 이미 존재함을 지적
- 최신 버전에서도 여전히 재현됨
- `type.classifier`가 `???`로 나타남을 확인

**sksamuel**:
- "What is the value of `type.classifier` inside a native image?" 질문 → 관심 표시

## 기술적 분석

### 수정 방향

**옵션 1: reflect-config.json 업데이트**
- hoplite-core의 Kotlin reflection 관련 클래스를 native-image config에 등록
- `KClassImpl`, `KTypeImpl` 등 필요한 클래스 파악

**옵션 2: KClass 체크 로직 완화**
- `type.classifier is KClass` 체크 대신 더 안전한 방법 사용
- GraalVM 환경에서도 작동하는 대체 로직

**옵션 3: GraalVM Feature 클래스 제공**
- `org.graalvm.nativeimage.hosted.Feature` 구현하여 자동 등록

### 영향받는 코드

- `hoplite-core/src/main/kotlin/com/sksamuel/hoplite/decoder/DecoderRegistry.kt:51`
- `example-native/META-INF/native-image/generated/reflect-config.json`

## 머지 확률 분석

| 요소 | 평가 |
|------|------|
| 실제 프로젝트(ORT) 영향 | 실사용자 문제 |
| 메인테이너 관심 | sksamuel 질문 코멘트 |
| 7건의 활발한 논의 | 커뮤니티 관심 높음 |
| 기존 native-image 예제 존재 | 프로젝트가 GraalVM 지원 의지 있음 |
| 기술적 난이도 높음 | GraalVM 전문 지식 필요 |
| **종합 머지 확률** | **80%** |

## 포트폴리오 가치

- **GraalVM / 클라우드 네이티브 역량** — 채용 시 강한 차별화
- JVM 내부 이해 (reflection, AOT compilation)
- Kotlin KClass 시스템 깊은 이해

## 실행 전략

1. GraalVM native-image 환경 세팅
2. Native Image Agent로 필요한 reflection config 자동 생성
3. `DecoderRegistry.kt:51`의 KClass 체크 로직 분석
4. reflect-config.json 업데이트 또는 체크 로직 수정
5. native-image 빌드 + 테스트

## 리스크

- GraalVM 환경 세팅 및 디버깅 난이도 높음
- sschuberth나 sgammon이 직접 수정할 가능성
- reflection config만으로 해결되지 않을 수 있음 (Kotlin compiler 수준 문제)
- 첫 기여로는 부담이 큼 → 신뢰 구축 후 도전 권장

## 선행 조건

- GraalVM CE/EE 설치
- `example-native` 모듈 빌드 확인
- ORT 프로젝트의 재현 환경 구성 (선택)
