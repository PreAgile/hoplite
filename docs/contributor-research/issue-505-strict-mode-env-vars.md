# Issue #505: strict() mode fails because of environment variables

## 기본 정보

| 항목 | 값 |
|------|-----|
| 이슈 번호 | [#505](https://github.com/sksamuel/hoplite/issues/505) |
| 상태 | OPEN |
| 라벨 | `bug` |
| 작성자 | rocketraman (주요 기여자, collaborator) |
| 생성일 | 2026-03-03 |
| 코멘트 | 3건 (rocketraman, TwoClocks) |
| 관련 이슈 | #500 (prefix + strict mode) |

## 문제 요약

v3.0.0의 `strict()` 모드가 환경변수가 존재하는 환경에서 실패한다. 환경변수 PropertySource가 로드한 값 중 config data class에 매핑되지 않는 것들이 "unused" 에러로 잡힌다.

## 커뮤니티 논의 (Slack + GitHub)

rocketraman이 sksamuel과 Slack에서 논의한 내용을 정리:

**strict mode 사용 시나리오 3가지:**

1. strict에 관심 없는 사용자
2. **YAML/JSON config 파일에서 stale한 값을 찾고 싶은 사용자** (주요 유즈케이스)
3. 환경 전체를 완벽하게 통제하고 싶은 사용자

**시나리오 2의 경우**: 환경변수 프로세서 때문에 strict mode가 실패할 필요 없음.

**시나리오 3의 경우**: 제한된 환경(systemd, Docker/K8s)에서도 hoplite가 사용하지 않는 환경변수가 존재할 수 있음 → 특정 환경변수만 체크하거나, 환경변수 프로세서별로 strict 동작을 설정할 수 있어야 함.

**TwoClocks의 추가 의견:**
- HOCON 파일에서 `${datahost}` 같은 참조 변수도 "used"로 인정해야 한다고 주장
- sealed class + `_type` 필드와 strict mode 조합에서도 실패하는 재현 코드 제공

## 기술적 분석

### 근본 원인 (추정)

`EnvironmentVariablesPropertySource`가 시스템의 모든 환경변수를 로드하지만, strict mode의 unused 체크는 config data class에 매핑되지 않은 **모든** 노드를 에러로 처리함.

### 수정 방향 (커뮤니티 제안 기반)

1. **PropertySource별 strict 설정**: 각 PropertySource가 strict mode에서 어떻게 동작할지 설정 가능하게
2. **환경변수 프로세서 기본 동작 변경**: 환경변수는 strict 체크에서 기본 제외
3. **화이트리스트/블랙리스트**: 특정 prefix의 환경변수만 strict 체크 대상으로

### 영향받는 코드 (추정)

- `hoplite-core/src/main/kotlin/com/sksamuel/hoplite/` 내 strict mode 검증 로직
- `EnvironmentVariablesPropertySource` 관련 코드
- `ConfigParser` 또는 `Decoding`의 unused 체크 로직

## 머지 확률 분석

| 요소 | 평가 |
|------|------|
| `bug` 라벨 | 메인테이너 인정 버그 |
| v3.0.0 RC 단계 | 정식 릴리스 전 수정 필요 |
| 주요 기여자가 보고 | rocketraman은 다수 PR 머지된 신뢰 기여자 |
| 커뮤니티 관심 | 3명이 논의 참여 |
| **종합 머지 확률** | **95%** |

## 포트폴리오 가치

- **v3.0.0 정식 릴리스 기여자**로 changelog에 이름
- Config validation/strict mode 설계 이해 증명
- 환경변수 처리 + 12-factor app 이해

## 실행 전략

1. `strict()` 관련 코드를 먼저 분석 (unused 체크 로직 위치 파악)
2. 이슈에 코멘트: 제안된 해결책 중 어느 방향으로 갈지 메인테이너 의견 확인
3. #500과 근본 원인이 같을 가능성 높으므로 동시에 수정 시도
4. 테스트 케이스: 환경변수가 있는 환경에서 strict mode 동작 검증

## 리스크

- 설계 방향이 아직 확정되지 않음 → 메인테이너 의견 먼저 확인 필수
- rocketraman이 직접 수정할 가능성 있음 → 빠르게 행동해야 함
