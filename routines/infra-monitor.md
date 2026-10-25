# Picake 인프라 모니터링 (Discord)

| 항목      | 값                                                                                                          |
| --------- | ----------------------------------------------------------------------------------------------------------- |
| 루틴 ID   | `trig_01F23Ye1hxjAQp8ziEtoQwLd`                                                                             |
| 스케줄    | `0 0 * * *` (UTC) = 매일 09:00 KST                                                                          |
| 체크아웃  | `picake-team/picake-engineering-automation`                                                                 |
| 허용 도구 | Bash, Read, Write, Grep, Glob, Skill, Artifact                                                              |
| 절차      | [`.claude/skills/picake-infra-monitor`](../.claude/skills/picake-infra-monitor/SKILL.md) 9번 자동 실행 모드 |

## 프롬프트

`<...>` 자리의 비밀값은 루틴 설정에만 넣습니다. 이 파일에는 절대 실제 값을 쓰지 마세요.

````text
# Picake 인프라 모니터링 (자동 실행)

저장소의 `.claude/skills/picake-infra-monitor/SKILL.md`를 읽고 "9. 자동 실행 모드"에 따라 진행한다. 코드 수정·커밋은 하지 않는다.

## 비밀값 (이 프롬프트에만 존재 — 로그·결과물·저장소에 남기지 말 것)

AWS 자격증명 (이 루틴 전용 IAM 유저, 읽기전용 + 두 인스턴스 한정 SSM 명령 실행만 가능):

```
export AWS_ACCESS_KEY_ID=<AWS_ACCESS_KEY_ID>
export AWS_SECRET_ACCESS_KEY=<AWS_SECRET_ACCESS_KEY>
export AWS_DEFAULT_REGION=ap-northeast-2
```

Discord 웹훅 URL: <DISCORD_WEBHOOK_URL_INFRA>
````
