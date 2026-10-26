# picake-engineering-automation

Picake의 Claude 클라우드 루틴과 운영 자동화 스킬을 모아 둔 레포입니다. 앱 코드는 없습니다.

| 레포                                                                                    | 내용                                                  |
| --------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| [picake-engineering-backend](https://github.com/picake-team/picake-engineering-backend) | 백엔드 API 서버                                       |
| [picake-engineering-web](https://github.com/picake-team/picake-engineering-web)         | 웹 프론트엔드 (`web-user`, `web-seller`, `web-admin`) |
| [picake-engineering-app](https://github.com/picake-team/picake-engineering-app)         | Flutter 앱                                            |
| **picake-engineering-automation** (현재)                                                | 클라우드 루틴·운영 자동화                             |

## 구성

| 경로                                  | 내용                                                                |
| ------------------------------------- | ------------------------------------------------------------------- |
| `.claude/skills/picake-infra-monitor` | EC2 인프라 상태·AWS 비용 점검 (루틴: Picake 인프라 모니터링)        |
| `.claude/skills/picake-data-report`   | PostHog + 관리자 통계 종합 리포트 (루틴: Picake 일일 데이터 리포트) |
| `.claude/skills/picake-web-performance` | web-user 실사용 웹 바이탈(PostHog) + Lighthouse 점검 (루틴: Picake 웹 성능 점검) |
| `routines/`                           | 루틴별 스케줄·설정·프롬프트 원문 (비밀값은 자리표시자)              |

## 시간 기준 (세 루틴 공통)

세 루틴(인프라 모니터링, 일일 데이터 리포트, 웹 성능 점검)은 모두 **한국 시간(KST), 어제까지 완료된 일자** 기준으로 표기하고, 리포트 맨 위에 집계 기준과 점검 시각을 적습니다. 실행은 매일 09:00 KST 무렵입니다. PostHog 프로젝트 시간대는 UTC라서 쿼리에서 반드시 `Asia/Seoul`로 변환합니다.

## 루틴 동작 방식

클라우드 루틴은 이 레포를 체크아웃하고, 짧은 프롬프트로 해당 스킬의 "자동 실행 모드"를 따르게 합니다. 절차를 바꾸려면 스킬 문서를 고쳐 `main`에 머지하면 다음 실행부터 반영됩니다. 프롬프트 자체(비밀값 포함)를 바꿀 때만 루틴 설정을 수정하고, 바뀐 프롬프트 원문을 `routines/`에도 반영합니다.

## 보안

이 레포는 **public**입니다. 자격증명·웹훅 URL·API 키·인스턴스 ID 같은 운영 식별자는 커밋하지 않습니다. 비밀값은 루틴 설정에만 두고, 문서에는 `<...>` 자리표시자를 씁니다.
