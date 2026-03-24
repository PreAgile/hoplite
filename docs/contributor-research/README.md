# Hoplite 기여 전략 리서치

## 목표

Kotlin 백엔드 엔지니어가 `sksamuel/hoplite`에 기여할 후보를 정리한 문서.

단순히 "쉬운 것"만 고르는 게 아니라, **"백엔드 채용팀에 강하게 어필하면서도 실제 머지 가능성이 높은 것"**을 기준으로 선별함.

## 저장소 상태

- 업스트림: `sksamuel/hoplite`
- 포크: `PreAgile/hoplite`
- 최종 업데이트: 2026-03-24

활발하게 관리되고 있다는 신호:

- 2026년 2월에 v3.0.0.RC1, RC2 연속 릴리스
- 2025 후반 ~ 2026 초반까지 외부 기여자 PR 포함 다수 머지
- 전체 기여자 90+ (포크 기준)
- Stars 1,023개, 성숙한 Kotlin 에코시스템 라이브러리

최근 머지된 PR 예시:

- YamlPropertySource println 제거 (glasser)
- foojay resolver convention 플러그인 업데이트 (TheBestPessimist)
- classpath/path 로딩 개선 (TheBestPessimist)
- sealed class discriminator YAML 수정 (meierjan)
- prefix 지정 EnvironmentVariablesPropertySource 수정 (galimru)
- AWS SDK v1 의존성 제거 (thake)

## 완료된 PR

| PR | 이슈 | 상태 | 날짜 |
|----|------|------|------|
| PR 517 | issue 500 prefix + strict 모드 unused 오류 | 🟡 OPEN (리뷰 대기) | 2026-03-23 |

## 메인테이너 성향 분석

### 핵심 메인테이너

| 이름 | GitHub | 역할 | 커밋 수 | 특징 |
|------|--------|------|---------|------|
| Sam Samuel | `sksamuel` | BDFL (사실상 1인 의사결정) | 754 (압도적) | 배치 머지 패턴, 주~월 단위로 쌓인 PR을 한꺼번에 처리 |
| Raman Gupta | `rocketraman` | COLLABORATOR | 13 | 설계 논의 주도, Slack에서 sksamuel과 직접 소통, PR 리뷰 |
| Sebastian Schuberth | `sschuberth` | 정기 기여자 | 7 | ORT 프로젝트 메인테이너, GraalVM/strict mode 이슈 활발 |
| TheBestPessimist | — | 최근 기여자 | 2 | 2026년 2월 2개 PR 머지, 적극적 자원봉사 |

### 머지 패턴

최근 머지된 PR에서 관찰된 패턴:

- **배치 머지**: sksamuel은 주~월 단위로 침묵하다가 한꺼번에 여러 PR을 머지. 2025-03-16에 5개 PR 동시 머지, 2026-02-20에 3개 동시 머지
- **평균 머지 시간**: ~18.6일 (중간값 ~9.6일). 신뢰받는 기여자(rocketraman, sschuberth)는 수 시간 내 머지
- **공식 리뷰 없음**: 30개 최근 PR 중 4개만 formal review 존재. sksamuel이 직접 인라인 코멘트 후 머지
- **테스트 필수**: PR 470에서 "테스트 고쳐주면 머지하겠다"고 명시적 요청
- PR 크기 제한 없음: +1/-1부터 +263/-265까지 모두 수용

### 코딩 컨벤션

- **2-space 인덴트** (.editorconfig 기준, gradle.kts는 3-space)
- 최대 줄 길이: 120
- trailing whitespace 제거, final newline 삽입
- Kotlin Coding Conventions 준수
- 별도 linter(detekt, ktlint) 없음
- `ConfigResult` 타입 기반 함수형 에러 처리 (Arrow 스타일)

### Stale 정책

- 60일 무활동 → `wontfix` 라벨 자동 부착
- 7일 후 자동 닫힘
- 예외: `pinned`, `[Status] Maybe Later` 라벨
- **⚠️ 이슈에 코멘트 남긴 후 방치하면 stale 처리됨. 주기적 활동 필요**

## 현재 후보 (2026-03-24 업데이트)

| 순위 | 이슈 | 유형 | 머지 확률 | 포트폴리오 가치 | 상태 | 상세 문서 |
|------|------|------|----------|---------------|------|----------|
| ~~1~~ | ~~`issue 500` prefix + strict 모드 오류~~ | ~~버그 수정~~ | — | — | ✅ **PR 517 제출** (2026-03-23) | [issue-500-prefix-strict.md](issue-500-prefix-strict.md) |
| 2 | `issue 503` PathNormalizer 소문자 회귀 | 버그 수정 | 높음 (80-85%) | 중간 | 🟢 OPEN (코멘트 0, 미클레임) | [issue-503-pathnormalizer-lowercase.md](issue-503-pathnormalizer-lowercase.md) |
| 3 | `issue 478` enum 기본값 지원 | 기능 추가 | 높음 (75-80%) | 중간 | 🟢 OPEN (코멘트 0, 미클레임) | [issue-478-enum-default.md](issue-478-enum-default.md) |
| 4 | `issue 505` strict 모드 + 환경변수 | 버그 수정/설계 | 중상 (65-75%) | 높음 | 🟡 활발한 논의 중, 미구현 | [issue-505-strict-env-vars.md](issue-505-strict-env-vars.md) |
| 5 | `issue 460` data class 기본값 + resolver | 버그 수정 | 중간 (60-70%) | 높음 | 🟡 부분 수정됨, 근본 미해결 | [issue-460-dataclass-defaults.md](issue-460-dataclass-defaults.md) |
| 6 | `issue 493` GraalVM native image | 버그 수정 | 중간 (50-60%) | 매우 높음 | 🟡 sschuberth 조사 중 | [issue-493-graalvm-native.md](issue-493-graalvm-native.md) |
| 7 | `issue 368` 필드별 환경변수 매핑 | 기능 추가/설계 | 중간 (50-60%) | 매우 높음 | 🟡 pinned, 3년간 논의 | [issue-368-envvar-per-field.md](issue-368-envvar-per-field.md) |
| 8 | `issue 210` sealed class 파라미터 오버랩 | 버그 수정 | 중하 (40-50%) | 높음 | 🟡 pinned, 5년간 미해결 | [issue-210-sealed-class-overlap.md](issue-210-sealed-class-overlap.md) |

### 제외됨

| 이슈 | 이유 |
|------|------|
| `issue 516` 리소스 로딩 단순화 | TheBestPessimist가 "주말에 작업하겠다"고 선언 (2026-02-21) |
| `PR 470` SecretFilesPreprocessor | sksamuel이 테스트 수정 요청 → 원작자(Bengreen) 1년간 미응답. 이어받기 가능하나 원작자 관계 고려 필요 |

## 추천 실행 순서

### ~~1단계: 빠른 성과~~ ✅ 진행 중

1. ~~**issue 500 prefix + strict 모드**~~ → ✅ PR 517 제출 (2026-03-23). 이슈에 코멘트 완료. 리뷰 대기 중
2. **issue 503 PathNormalizer 소문자 변환** — 명확한 회귀 버그, 범위 좁음. issue 500 머지 후 다음 대상

### 2단계: 설계 참여형

3. **issue 505 strict 모드 + 환경변수** — rocketraman과 설계 논의 진행 중. 이슈에 코멘트로 참여 후 구현
4. **issue 478 enum 기본값** — 깔끔한 기능 추가, 자기 완결적

### 3단계: 신뢰 쌓인 후 도전

5. **issue 460 data class 기본값** — DataClassDecoder 내부 이해 필요
6. **issue 493 GraalVM** — 높은 임팩트, GraalVM 디버깅 경험 필요
7. **issue 368 필드별 환경변수** — pinned, 설계 미확정, 가장 후순위

## 주의할 이슈 (피해야 할 것들)

| 이슈 | 이유 |
|------|------|
| `issue 516` 리소스 로딩 단순화 | `TheBestPessimist`가 "주말에 작업하겠다"고 선언 (2026-02-21). 충돌 가능 |
| `PR 470` SecretFilesPreprocessor | `Bengreen`이 원작자, sksamuel이 테스트 수정 요청. 1년간 미응답이나 이어받기 시 원작자 관계 고려 |
| `issue 493` GraalVM (부분 주의) | `sschuberth`, `sgammon`이 조사 중. 기여 시 이슈에서 협력 제안 필수 |
| wontfix 라벨 이슈 전체 | stale bot이 자동 부착. 메인테이너가 직접 작업할 의사 없음. 기여 전 코멘트로 반응 확인 필수 |

## PR 전략: 묶지 말고 하나씩

각 이슈는 **별도 PR**로 올려야 함. 이유:

- sksamuel은 배치 머지 패턴 — 좁고 리뷰하기 쉬운 PR이 유리
- 관련 없는 변경을 묶으면 리뷰 부담 증가
- 하나에 수정 요청이 오면 나머지도 막힘
- 별도 PR이 기여 이력을 보기 좋게 만듦
- 머지된 PR 하나하나가 다음 PR의 신뢰도를 올려줌

## PR 제출 전 체크리스트

- [ ] 이슈에 먼저 코멘트 (선점, stale 방지)
- [ ] **2-space 인덴트** 사용 (editorconfig 준수)
- [ ] 코드 변경에는 테스트 포함 (PR 470 사례 참고)
- [ ] `./gradlew check`로 테스트 통과 확인
- [ ] PR 하나에 이슈 하나
- [ ] PR 본문에 `Closes #XXXX`로 이슈 번호 참조
- [ ] 커밋 메시지는 영문, 간결하게
