# Hoplite 기여 전략 리서치

## 목표

Kotlin 백엔드 엔지니어가 `sksamuel/hoplite`에 기여할 후보를 정리한 문서.

단순히 "쉬운 것"만 고르는 게 아니라, "백엔드 채용팀에 강하게 어필하면서도 실제 머지 가능성이 높은 것"을 기준으로 선별함.

## 저장소 상태

- 업스트림: `sksamuel/hoplite`
- 포크: `PreAgile/hoplite`
- 최종 업데이트: 2026-03-18

| 항목 | 값 |
|------|-----|
| 언어 | Kotlin |
| 스타 | 1,023 |
| 포크 | 90 |
| 라이선스 | Apache 2.0 |
| 현재 버전 | v3.0.0.RC2 (릴리스 후보 단계) |
| 메인테이너 | sksamuel (1인) |
| CONTRIBUTING.md | 없음 — 공식 기여 가이드 부재, 진입장벽 낮음 |
| 열린 이슈 | 25건 (wontfix 13건 제외 시 12건) |
| 열린 PR | 1건 |

활발하게 관리되고 있다는 신호:

- 최근 커밋: `2026-03-06` (Kotest 의존성 업데이트)
- v3.0.0 RC2 릴리스: `2026-02-21`
- 2025 후반 ~ 2026 초반까지 외부 PR이 지속적으로 머지됨
- 최근 머지된 PR:
  - `#515` — `2026-02-20` 머지 (YamlPropertySource println 제거)
  - `#513` — `2026-02-09` 머지 (foojay resolver 업데이트)
  - `#512` — `2026-02-07` 머지 (path/classpath 로딩 개선)

## 완료된 PR

| PR | 이슈 | 상태 | 날짜 |
|----|------|------|------|
| — | — | 아직 없음 | — |

## 메인테이너(sksamuel) 성향 분석

### 활동 패턴

- **버스트형**: 몇 주간 집중 머지/커밋 후 한동안 조용해지는 패턴
- 응답까지 수 일 ~ 수 주 소요될 수 있음 (인내 필요)
- 최근 v3.0.0 릴리스 준비에 집중 중

### 머지 성향 (매우 관대)

최근 머지된 PR에서 관찰된 패턴:

- 외부 PR 머지율이 매우 높음 — 최근 20건 머지 PR 중 코드 품질 사유 리젝 0건 (확인한 비머지 3건은 메타 PR, 본인 실험 포기, 중복)
- 테스트 통과가 머지의 핵심 조건 — PR #470에서 "fix the test and I can merge" 직접 코멘트
- 작고 집중된 변경 선호
- 코드 변경 시 테스트 포함 필수 (Kotest 프레임워크)
- 문제 발생 시 revert 후 재적용하는 실용적 접근 (#483 → #485)

리젝된 PR 분석 (최근 30건 중):

| PR | 리젝 이유 |
|----|----------|
| `#514` | "릴리스 해달라" 메타 PR (코드 아님) |
| `#497` | sksamuel 본인이 포기한 실험 |
| `#464` | #465와 중복 PR |

→ **코드 품질 사유 리젝: 0건**

### 선호하는 기여 유형 (머지 확률 순)

1. **버그 수정 + 테스트** → 거의 즉시 머지
2. **의존성 업데이트** → 자동 머지급
3. **새 Decoder / PropertySource 추가** (소규모 기능)
4. **문서 개선**
5. **리팩토링 / 코드 정리**

### 코멘트 스타일

- 짧고 직접적 ("would you be able to fix the test and I can merge")
- 설계 방향을 간결하게 제안 ("Maybe we allow a `_type` field")
- 질문에 대해 기꺼이 응답

### 주요 기여자

| 기여자 | 역할 |
|--------|------|
| `sksamuel` | 유일한 메인테이너 |
| `rocketraman` | 주요 외부 기여자 (다수 PR 머지) |
| `TheBestPessimist` | 최근 활발한 기여자 |
| `sschuberth` | 빌드/테스트 개선 기여 |

## 이슈 분류

### 활성 이슈 (12건) — wontfix 제외

| # | 제목 | 라벨 | 생성일 | 비고 |
|---|------|------|--------|------|
| 516 | Simplify resource loading | — | 2026-02-21 | sksamuel 본인 이슈 |
| 505 | `strict()` mode fails because of environment variables | bug | 2026-03-03 | v3.0.0 핵심 버그 |
| 503 | Breaking change from 2.7.5 to 2.8.0 | — | 2025-12-09 | 호환성 문제 |
| 500 | Prefix + strict mode bogus "unused" error | bug | 2025-12-09 | #505와 연관 |
| 493 | GraalVM native image KClass failure | — | 2026-03-04 | 클라우드 네이티브 |
| 484 | GraalVM reachability metadata 기여 | — | 2025-06-05 | 외부 레포 기여 필요 |
| 478 | Optional default enum value | — | 2025-02-25 | Decoder 기능 확장 |
| 471 | Separate config and secrets (12-factor) | — | 2025-06-11 | 설계 논의 필요 |
| 460 | Default values don't apply after resolvers | — | 2025-04-01 | Resolver 파이프라인 버그 |
| 450 | Abstract class instead of Sealed | — | 2025-10-16 | 메인테이너 설계 제안 있음 |
| 368 | Per-field environment variable override | pinned | 2024-04-14 | **pinned** = 메인테이너 관심 |
| 210 | Sealed class detection with overridden params | pinned | 2022-08-05 | **pinned**, 4년 된 난이도 높은 버그 |

### wontfix 이슈 (13건) — 건드리지 말 것

stale-bot이 자동 처리. 메인테이너가 관심 없음. 대표 예: #472, #469, #452, #418, #417, #409, #402, #399, #397, #396, #395, #394, #358

### 열린 PR (1건)

| # | 제목 | 작성자 | 상태 |
|---|------|--------|------|
| 470 | SecretFilesPreprocessor | Bengreen | 메인테이너가 "테스트 고치면 머지" 코멘트, 작성자 미응답 |

## 현재 TOP 5 후보 (2026-03-18)

| 순위 | 이슈 | 유형 | 머지 확률 | 상세 문서 |
|------|------|------|----------|----------|
| 1 | `#505` strict mode + 환경변수 실패 | 버그 수정 | 매우 높음 — `bug` 라벨, v3.0.0 RC 블로커 | [issue-505-strict-mode-env-vars.md](issue-505-strict-mode-env-vars.md) |
| 2 | `#500` prefix + strict mode unused 에러 | 버그 수정 | 매우 높음 — `bug` 라벨, 재현 코드+에러 명확 | [issue-500-prefix-strict-unused.md](issue-500-prefix-strict-unused.md) |
| 3 | `#470` SecretFilesPreprocessor 테스트 수정 | PR 이어받기 | 거의 확정 — 메인테이너 "테스트 고치면 머지" 명시 | [pr-470-secret-files-preprocessor.md](pr-470-secret-files-preprocessor.md) |
| 4 | `#478` Enum default fallback 지원 | 기능 추가 | 높음 — 유즈케이스 명확, 메인테이너 미응답이 변수 | [issue-478-enum-default-fallback.md](issue-478-enum-default-fallback.md) |
| 5 | `#493` GraalVM native image KClass 실패 | 버그 수정 | 높음 — 커뮤니티 활발, 난이도가 변수 | [issue-493-graalvm-kclass.md](issue-493-graalvm-kclass.md) |

### 후보별 상세

#### 1위: #505 — strict mode + 환경변수 실패 (`bug`)

- **문제**: v3.0.0의 `strict()` 모드가 환경변수가 있는 환경에서 실패
- **머지 판단 근거**: `bug` 라벨, v3.0.0 RC 단계 블로커, 최근 20건 외부 PR 중 코드 사유 리젝 0건
- **난이도**: 중간 — `DecodeModeValidator.ensureAllUsed()` 및 `Decoding.createDecodingState()` 수정
- **포트폴리오 어필**: "v3.0.0 정식 릴리스를 막고 있던 핵심 버그를 수정" → changelog에 이름
- **전략**: #500과 근본 원인이 같을 가능성 높음. 하나의 PR로 두 이슈 동시 해결 시 임팩트 극대화

#### 2위: #500 — prefix + strict mode unused 에러 (`bug`)

- **문제**: prefix를 사용하는 config 로딩에서 strict mode가 잘못된 unused 에러 발생
- **머지 판단 근거**: `bug` 라벨, 재현 코드+에러 메시지 명확, #505와 동일 조건
- **난이도**: 중간 — `ConfigParser.decode()`의 `prefixedNode()` 호출 후 unused 체크에서 prefix 노드 제외 필요
- **포트폴리오 어필**: Config prefix binding 로직 이해 증명
- **전략**: #505와 묶어서 하나의 PR로 제출 권장

#### 3위: #470 — SecretFilesPreprocessor (기존 PR 이어받기)

- **문제**: Bengreen이 제출한 SecretFilesPreprocessor PR의 테스트가 깨져 있음
- **머지 판단 근거**: sksamuel이 "fix the test and I can merge and release" 명시 (2025-03-16 코멘트)
- **난이도**: 낮음 — 기존 코드 기반에서 테스트만 수정
- **포트폴리오 어필**: Secret management 이해, 오픈소스 협업 (다른 기여자 작업 계승)
- **전략**: Bengreen 브랜치 포크 → 테스트 수정 → 새 PR 제출. 기존 작성자에게 예의 코멘트 필수

#### 4위: #478 — Enum default fallback 지원

- **문제**: 잘못된 enum 값이 들어올 때 기본값으로 fallback하는 옵션이 없음
- **머지 판단 근거**: 유즈케이스 명확, 기존 Decoder 패턴 따라 구현 가능. 메인테이너 미응답(코멘트 0건)이 유일한 불확실성
- **난이도**: 중간 — `EnumDecoder.safeDecode()` (`enum.kt:28-45`) 수정 + fallback 로직 추가
- **포트폴리오 어필**: Type-safe config 설계, Decoder 아키텍처 이해
- **전략**: 이슈에 설계 제안 코멘트 먼저 → 메인테이너 승인 후 구현

#### 5위: #493 — GraalVM native image KClass 실패

- **문제**: GraalVM native image에서 KClass 런타임 체크 실패
- **머지 판단 근거**: 7건 활발한 논의, 메인테이너 질문 코멘트, 실사용자(ORT) 영향. GraalVM 전문 지식 필요가 불확실성
- **난이도**: 높음 — GraalVM reflection metadata 이해 필요
- **포트폴리오 어필**: 클라우드 네이티브 환경 경험, JVM 내부 이해
- **전략**: GraalVM reachability metadata 또는 reflection config 추가

## 추천 실행 순서

### 1단계: 확실한 첫 기여로 신뢰 구축 (이번 주)

1. **#470 SecretFilesPreprocessor 테스트 수정** — 메인테이너가 머지를 약속한 상태. 가장 확실한 첫 기여. Bengreen에게 예의 코멘트 후 새 PR 제출

### 2단계: v3.0.0 핵심 버그로 임팩트 (1~2주 내)

2. **#505 + #500 strict mode 버그 수정** — v3.0.0 RC 단계에서 `bug` 라벨 이슈 해결은 메인테이너에게 가장 urgent한 기여. 정식 릴리스 changelog에 이름이 올라감

### 3단계: 기능 기여로 설계력 증명 (이후)

3. **#478 Enum default fallback** — Decoder 확장은 hoplite 핵심 아키텍처 이해를 증명
4. **#493 GraalVM 지원** — 메인테이너와 신뢰가 쌓인 후 도전

## 백엔드 포트폴리오 시너지

| 증명 역량 | 매핑 이슈 |
|-----------|----------|
| Kotlin 숙련도 | 모든 이슈 (순수 Kotlin 라이브러리) |
| Type-safe 설계 | #478 (Enum), #505/#500 (Validation) |
| 12-Factor App 이해 | #505 (환경변수 처리), #471 (Config/Secret 분리) |
| 테스트 작성 (Kotest) | 모든 PR에 테스트 필수 |
| 클라우드 네이티브 | #493 (GraalVM), #470 (Secret 관리) |
| 오픈소스 협업 | PR 프로세스, 코드리뷰 대응 |

KSentinel 프로젝트와의 시너지:
- KSentinel이 Kotlin 기반 → hoplite 기여 = "Kotlin 생태계 전반에 기여하는 개발자"
- hoplite는 config 로딩 라이브러리 → 서비스 인프라 레이어 경험 증명

## PR 전략: 묶지 말고 하나씩

각 이슈는 **별도 PR**로 올려야 함 (#505+#500은 근본 원인이 같으면 하나로 가능). 이유:

- 메인테이너 1인이라 리뷰 부담을 최소화해야 함
- 관련 없는 변경을 묶으면 리뷰 지연
- 하나에 수정 요청이 오면 나머지도 막힘
- 별도 PR이 기여 이력을 보기 좋게 만듦
- 머지된 PR 하나하나가 다음 PR의 신뢰도를 올려줌

## 제외한 후보

| 이슈 | 이유 |
|------|------|
| `#368` Per-field 환경변수 | pinned지만 설계가 확정되지 않음. 메인테이너와 설계 합의 필요 |
| `#210` Sealed class 파라미터 override | 4년간 미해결, 복잡도 매우 높음 |
| `#471` Config/Secret 분리 (12-factor) | 대규모 아키텍처 변경, 첫 기여로 부적합 |
| `#450` Abstract class 대신 사용 | 설계 논의 단계, 구현 방향 미확정 |
| `#460` Default values + resolvers | Resolver 파이프라인 깊은 이해 필요, 난이도 높음 |
| `#503` Breaking change 2.7.5→2.8.0 | 호환성 이슈로 단순 수정 불가 |
| wontfix 라벨 전체 (13건) | stale-bot 자동 처리, 메인테이너 무관심 |

## PR 제출 전 체크리스트

- [ ] 이슈에 먼저 코멘트 (선점 + 메인테이너 의견 확인)
- [ ] 기존 기여자 코드 위에 작업 시 예의 코멘트
- [ ] `./gradlew build`로 전체 빌드 통과 확인
- [ ] Kotest 기반 테스트 작성/수정
- [ ] PR 하나에 이슈 하나 (예외: 근본 원인 공유 시 묶기 가능)
- [ ] 코드 변경에는 반드시 테스트 포함
- [ ] PR 본문에 `Closes #XXXX`로 이슈 번호 참조
- [ ] 간결한 커밋 메시지 — 메인테이너 스타일에 맞춤
