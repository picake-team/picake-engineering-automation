# Picake 일일 데이터 리포트 (Discord)

| 항목      | 값                                                                                                      |
| --------- | ------------------------------------------------------------------------------------------------------- |
| 루틴 ID   | `trig_01RjTnMYKVVznyav9R4iWR2L`                                                                         |
| 스케줄    | `0 0 * * *` (UTC) = 매일 09:00 KST                                                                      |
| 체크아웃  | `picake-team/picake-engineering-automation`                                                             |
| 커넥터    | PostHog                                                                                                 |
| 허용 도구 | Bash, Read, Write, Grep, Glob, Skill, Artifact                                                          |
| 절차      | [`.claude/skills/picake-data-report`](../.claude/skills/picake-data-report/SKILL.md) 4번 자동 실행 모드 |

## 프롬프트

`<...>` 자리의 비밀값은 루틴 설정에만 넣습니다. 이 파일에는 절대 실제 값을 쓰지 마세요.

```text
# Picake 일일 데이터 리포트 (자동 실행)

저장소의 `.claude/skills/picake-data-report/SKILL.md`를 읽고 "4. 자동 실행 모드"에 따라 진행한다. 코드 수정·커밋은 하지 않는다.

## 비밀값 (이 프롬프트에만 존재 — 로그·결과물·저장소에 남기지 말 것)

관리자 API 키 (X-Admin-Api-Key 헤더): <ADMIN_API_KEY>

Discord 웹훅 URL: <DISCORD_WEBHOOK_URL_DATA_REPORT>
```
