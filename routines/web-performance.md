# Picake 웹 성능 점검 (Discord)

| 항목      | 값                                                                                                            |
| --------- | ------------------------------------------------------------------------------------------------------------- |
| 루틴 ID   | `<생성 후 기입>`                                                                                              |
| 스케줄    | `0 0 * * *` (UTC) = 매일 09:00 KST                                                                            |
| 체크아웃  | `picake-team/picake-engineering-automation`                                                                   |
| 커넥터    | PostHog                                                                                                       |
| 허용 도구 | Bash, Read, Write, Grep, Glob, Skill, Artifact                                                                |
| 절차      | [`.claude/skills/picake-web-performance`](../.claude/skills/picake-web-performance/SKILL.md) 5번 자동 실행 모드 |
| 전송      | Discord FE 채널 요약 + Artifact 대시보드 링크 (웹훅 URL은 루틴 설정에만 둠)                                   |

## 프롬프트

`<...>` 자리의 비밀값은 루틴 설정에만 넣습니다. 이 파일에는 절대 실제 값을 쓰지 마세요.

```text
# Picake 웹 성능 점검 (자동 실행)

저장소의 `.claude/skills/picake-web-performance/SKILL.md`를 읽고 "5. 자동 실행 모드"에 따라 진행한다. 코드 수정·커밋은 하지 않는다.

## 비밀값 (이 프롬프트에만 존재 — 로그·결과물·저장소에 남기지 말 것)

PageSpeed Insights API 키 (PSI_API_KEY): <PSI_API_KEY>

Discord 웹훅 URL (FE 채널): <DISCORD_WEBHOOK_URL_FE>
```
