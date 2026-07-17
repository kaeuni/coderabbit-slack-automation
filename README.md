# coderabbit-slack-automation

GitHub Pull Request 이벤트와 CodeRabbit 리뷰 결과를 Slack으로 자동 알림해주는 GitHub Actions 워크플로우입니다.

## 주요 기능

PR 진행 상황에 따라 아래 4가지 케이스에 자동으로 Slack 알림을 보냅니다.

| 이벤트 | 알림 대상 | 내용 |
|---|---|---|
| PR 생성 | 단체 채널 | 🚀 PR 생성 알림 |
| CodeRabbit 정식 리뷰 제출 | 작성자 개인 DM | 🤖 리뷰 완료, 확인 요청 |
| `ready-for-review` 라벨 추가 | 단체 채널 | 🔍 검토 요청 |
| PR 병합 완료 | 단체 채널 | 🎉 병합 완료 축하 |

## 동작 원리

- `.github/workflows/slack-notify.yml` 하나의 워크플로우로 동작합니다.
- `pull_request`(opened, labeled, closed)와 `pull_request_review`(submitted) 두 가지 이벤트를 트리거로 사용합니다.
- 이벤트 종류와 조건에 따라 분기해서 Slack Web API(`chat.postMessage`)를 호출합니다.
- CodeRabbit 리뷰 알림은 GitHub 아이디와 Slack 멤버 ID를 매핑한 `slackUsers` 객체를 기준으로 개인 DM을 보냅니다.
- CodeRabbit이 리뷰를 **`pull_request_review`로 정식 제출**하면, `review.state`가 `commented` / `approved` / `changes_requested` 중 무엇이든 상관없이 무조건 DM이 발송됩니다.
- 모든 분기에 디버깅용 `console.log`가 찍혀 있어서, Actions 실행 로그에서 어떤 이벤트가 왜 매칭/스킵됐는지 바로 확인할 수 있습니다.

## 사전 준비

### 1. Slack App 및 Bot Token

- Slack App을 생성하고 `chat:write`, `im:write` 권한을 부여합니다.
  - `chat:write`: 채널/DM 메시지 전송에 필요
  - `im:write`: 봇이 사용자와 DM 대화를 새로 여는 데 필요
- 권한(Scope)을 추가했다면 **Reinstall App to Workspace**를 눌러 워크스페이스에 재설치해야 토큰에 반영됩니다.
- 발급받은 Bot Token(`xoxb-...`)을 GitHub 저장소 Secrets에 등록합니다.

```
Settings > Secrets and variables > Actions > New repository secret
Name: SLACK_BOT_TOKEN
Value: xoxb-...
```

### 2. 알림 채널 ID 설정

`slack-notify.yml`의 `SLACK_CHANNEL` 값을 알림을 받을 Slack 채널 ID로 변경합니다.

```yaml
env:
  SLACK_CHANNEL: "C0BHJV049EF"
```

### 3. 개인 DM 대상자 매핑

CodeRabbit 리뷰 완료 시 개인 DM을 받을 사람을 GitHub 아이디 → Slack 멤버 ID로 매핑합니다.

```js
const slackUsers = {
  "kaeuni": "U0BJK02GEMN"
};
```

## 설치 방법

1. 이 저장소의 `.github/workflows/slack-notify.yml` 파일을 대상 저장소에 복사합니다.
2. 위 사전 준비 항목(Secrets, 채널 ID, 사용자 매핑)을 설정합니다.
3. **워크플로우 파일을 default 브랜치(main)에 먼저 병합**합니다.
   - `pull_request` 이벤트의 `labeled`, `closed` 등 후속 액션이 정상적으로 트리거되려면 워크플로우 파일이 default 브랜치에 존재해야 합니다.
4. 이후 새로 생성되는 PR부터 정상적으로 알림이 동작합니다.

## 디버깅

알림이 오지 않을 때는 GitHub 저장소 → **Actions** 탭 → 해당 워크플로우 실행(run)을 열어 로그를 확인하세요.

- 실행 기록 자체가 없다면 이벤트가 트리거되지 않은 것 (워크플로우 파일이 default 브랜치에 없거나, GitHub 이벤트 자체가 발생하지 않은 경우)
- 로그에 `[스킵]`이 찍혀 있다면 → 어떤 조건에도 매칭되지 않은 것이니, 바로 위에 찍힌 `eventName`, `action`, `review.state` 등의 값을 보고 원인 파악
- `[2] slackUsers에 "..." 매핑이 없어 DM 스킵` 로그가 보인다면 `slackUsers` 객체에 해당 GitHub 아이디 매핑을 추가

## 주의사항

- `ready-for-review` 라벨명은 코드에 하드코딩되어 있습니다. 다른 이름을 쓰고 싶다면 워크플로우 파일의 라벨 조건을 수정하세요.
- CodeRabbit이 `pull_request_review`를 제출할 때마다(최초든 재검토든) 매번 DM이 발송됩니다.
- `slackUsers` 매핑에 없는 GitHub 아이디는 DM이 발송되지 않고 조용히 스킵됩니다.
- **워크플로우 파일은 반드시 default 브랜치(main)에 존재해야 `pull_request` 이벤트가 정상적으로 트리거됩니다.** feature 브랜치에서 PR을 최초로 열 때는 실행되지만, 이후 같은 PR에서 발생하는 라벨 변경, 병합 등 후속 이벤트는 워크플로우 파일이 default 브랜치에 없으면 실행되지 않습니다. 참고 [github/docs#39364](https://github.com/github/docs/issues/39364)