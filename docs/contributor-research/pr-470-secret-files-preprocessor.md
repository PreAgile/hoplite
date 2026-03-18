# PR #470: SecretFilesPreprocessor — 테스트 수정 후 머지 가능

## 기본 정보

| 항목 | 값 |
|------|-----|
| PR 번호 | [#470](https://github.com/sksamuel/hoplite/pull/470) |
| 상태 | OPEN |
| 작성자 | Bengreen |
| 생성일 | 2025-01-19 |
| 변경 규모 | +60줄, -0줄 |
| 메인테이너 코멘트 | "fix the test and I can merge and release" |
| 마지막 활동 | 2025-03-16 (sksamuel 코멘트) |

## PR 내용

Kubernetes/Docker 환경에서 마운트된 시크릿 파일로부터 config 값을 로드하는 `SecretFilesPreprocessor` 추가.

### 유즈케이스

K8s에서 시크릿은 kv pair로 저장되고 컨테이너 마운트 경로에 파일로 노출됨. 이 프리프로세서는 `${key}` 패턴을 시크릿 디렉토리의 파일 내용으로 치환.

### 변경 파일

| 파일 | 내용 |
|------|------|
| `README.md` | SecretFilesPreprocessor 문서 추가 |
| `hoplite-core/.../SecretFilesPreprocessor.kt` | 프리프로세서 구현 (34줄) |
| `hoplite-core/.../SecretFilesPreprocessorTest.kt` | 테스트 (22줄) — **깨진 상태** |
| `hoplite-core/.../myconfig.yaml` | 테스트용 config |
| `hoplite-core/.../secrets/secretB` | 테스트용 시크릿 파일 |

## 현재 문제점 (코드 리뷰)

### 1. 패키지명 오류
```kotlin
// 현재 (잘못됨)
package com.polecatworks.kotlin.k8smicro.utils

// 수정 필요
package com.sksamuel.hoplite.preprocessor
```

### 2. 테스트 파일 import 누락
```kotlin
// 현재 — import 없음, 클래스 참조도 잘못됨
class SecretFilesPreprocessorTest : StringSpec() {
```

필요한 import:
```kotlin
import com.sksamuel.hoplite.ConfigLoaderBuilder
import com.sksamuel.hoplite.preprocessor.SecretFilesPreprocessor
import io.kotest.core.spec.style.StringSpec
import io.kotest.matchers.shouldBe
```

### 3. 불필요한 println
```kotlin
println("Secrets DIR =$x")  // 제거 필요
```

### 4. 테스트 리소스 경로
시크릿 디렉토리를 classpath 상대경로로 참조 → 테스트 환경에서 절대경로 해결 필요

## 머지 확률 분석

| 요소 | 평가 |
|------|------|
| 메인테이너 명시적 승인 | "fix the test and I can merge and release" |
| 기능 자체는 승인됨 | K8s 시크릿 관련 질문 후 만족 |
| Bengreen 1년 미응답 | 다른 기여자가 이어받을 수 있음 |
| **종합 머지 확률** | **99%** |

## 포트폴리오 가치

- Kubernetes secret management 이해 증명
- 12-factor app (config/secret 분리) 실천
- 오픈소스 협업: 다른 기여자의 작업을 이어받아 완성

## 실행 전략

### 방법 A: Bengreen 브랜치 이어받기
1. Bengreen의 브랜치를 포크
2. 패키지명, import, println 수정
3. 테스트 통과 확인
4. 새 PR 제출, 원본 PR #470 참조

### 방법 B: 처음부터 새로 작성
1. 동일한 SecretFilesPreprocessor를 올바른 패키지로 작성
2. 완전한 테스트 작성
3. PR 제출

### 예의 코멘트 (필수)
PR #470에 코멘트:
> "Hi @Bengreen, I noticed this PR has been waiting. I'd like to help fix the test so it can be merged. Would you mind if I submit a follow-up PR with the fixes?"

## 수정 체크리스트

- [ ] 패키지명 `com.sksamuel.hoplite.preprocessor`로 수정
- [ ] 필요한 import 추가
- [ ] `println` 제거
- [ ] 테스트에서 classpath 리소스 경로 해결
- [ ] `./gradlew build` 통과 확인
- [ ] README 테이블 포맷 확인
