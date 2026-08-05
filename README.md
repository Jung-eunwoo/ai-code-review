# ai-code-review

GitHub PR 에 코드 리뷰를 자동으로 남긴다. 출력 형식은 CodeRabbit 을 따라갔다. walkthrough 하나에 흐름도를 붙이고, 심각도별로 인라인 코멘트를 단다.

의존성이 없다. `gh` CLI 와 Node 22 만 있으면 돈다. `package.json` 도 `node_modules` 도 없다.

기본 분석기는 Gemini (`gemini-flash-latest`) 다. [AI Studio](https://aistudio.google.com/apikey) 무료 키로 돌아간다. `--provider claude` 로 바꿀 수 있다.

PR 하나에 두 개를 남긴다.

- walkthrough 코멘트: 변경 요약, 역할별 Changes 표, 리뷰 난이도 추정, mermaid 흐름도, 접힌 nitpick
- 인라인 리뷰: 문제가 있는 줄에 직접. 커밋 가능한 suggestion 이 붙는다

남긴 지적은 계속 따라간다. 고쳐지면 다음 push 때 스레드를 닫는다. 답글이 달리면 읽고 판단해서, 코드를 안 고쳤어도 그렇게 한 이유가 타당하면 접는다. 다만 데이터 손실, 보안, 인증 우회는 "의도했다"로 닫히지 않는다.

| | 뜻 | 어디에 |
| --- | --- | --- |
| 🔴 `critical` | 데이터 손실·보안·크래시·의도와 반대로 동작 | 인라인 |
| 🟠 `major` | 특정 조건에서 재현되는 버그, 복구 불가 상태 | 인라인 |
| 🟡 `minor` | 리팩터, 중복 판정, 오해를 부르는 네이밍 | 인라인 |
| 🔵 `nit` | 오타·스타일 | 요약에 접어서 |

---

## 쓰는 법

```bash
cp .env.example .env      # GEMINI_API_KEY 를 채운다
gh auth login             # repo scope

node review.mjs https://github.com/OWNER/REPO/pull/115   # 확인만 (파일로만 출력)
node review.mjs 115 --repo OWNER/REPO --post             # 실제 게시
node review.mjs --self-test                              # 파서·가드 검증
```

키는 `.env` 에만 둔다. `.gitignore` 가 `.env` 를 막고 `.env.example` 만 통과시킨다. CI 는 `.env` 없이 저장소 시크릿으로만 돈다.

`--post` 를 안 붙이면 항상 dry-run 이다. `review-<번호>.md` 와 `.json` 만 로컬에 쓰고 아무것도 게시하지 않는다. 공개 저장소에 잘못 올라간 코멘트는 지워도 알림이 이미 나간 뒤라서, 되돌리기 어려운 쪽을 기본값으로 두지 않았다.

| 플래그 | 기본값 | |
| --- | --- | --- |
| `--repo owner/name` | PR URL 에서 추출 | 대상 저장소 |
| `--post` | off | 실제 게시 |
| `--provider` | `gemini` | `claude` 로 교체 가능 |
| `--model` | `gemini-flash-latest` | `gemini-pro-latest` 가 더 정확하지만 무료 한도를 빨리 쓴다 |
| `--force` | off | 같은 커밋 중복 리뷰 방지 무시 |

`--model` 을 매번 치기 싫으면 `.env` 에 `GEMINI_MODEL=` 을 넣으면 된다. 우선순위는 `--model` > `.env` > 기본값.

같은 커밋을 두 번 리뷰하지 않고, 두 번째 실행부터는 walkthrough 와 다이어그램을 재사용해 바뀐 부분만 다시 본다. walkthrough 는 새로 달지 않고 기존 것을 고친다. 아직 안 고친 지적은 이미 열려 있는 스레드가 그대로 남으므로 인라인 코멘트를 다시 달지 않고, walkthrough 의 접힌 목록에만 다시 나온다.

---

## 다른 저장소에 연결하기

**1. 키 등록** (저장소당 한 번)

[AI Studio](https://aistudio.google.com/apikey) 에서 무료 키를 받아 대상 저장소 시크릿에 넣는다. organization 시크릿으로 등록해두면 저장소를 추가할 때마다 다시 할 필요가 없다.

```bash
gh secret set GEMINI_API_KEY --repo OWNER/REPO
```

**2. 워크플로 복사**

[`examples/pr-review.yml`](examples/pr-review.yml) 을 대상 저장소의 `.github/workflows/ai-review.yml` 로 그대로 복사한다.

```yaml
- uses: Jung-eunwoo/ai-code-review@01365adee404635cbb79c25b9f6f100ff8b2daff
  with:
    pr: ${{ github.event.number }}
  env:
    GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }}
    GH_TOKEN: ${{ github.token }}
```

`GH_TOKEN` 은 워크플로가 자동 발급하므로 저장소마다 PAT 를 만들 필요가 없다. 도입 초기에는 `post: "false"` 로 두면 게시 없이 Actions 로그에만 남는다.

**전체 커밋 SHA 로 고정한다.** `@main` 은 물론이고 `@v1` 같은 태그도 안 된다. 태그는 force-push 로 옮겨지므로 결국 가변 ref 고, 그러면 붙어 있는 저장소가 검토 없이 새 코드를 키와 쓰기 권한으로 실행하게 된다. 업그레이드는 리뷰어 저장소의 diff 를 보고 SHA 를 손으로 올린다.

---

## 보안

위협 모델은 한 줄이다. 리뷰 대상 diff 는 공격자가 내용을 정하는 입력이고, 그것이 LLM 프롬프트로 들어간다. 리뷰어는 그 결과를 공개 저장소에 게시할 권한을 가지고 있다.

프롬프트 지시는 모델이 따를 때만 작동하므로, 모델이 이미 넘어갔다고 가정한 층을 따로 뒀다.

- 분석기에 도구를 주지 않는다. 기본 경로는 REST 호출이라 모델이 파일을 읽거나 명령을 실행할 수단이 구조적으로 없다.
- 자격증명을 분리한다. 게시기에 모델 키를, 분석기에 GitHub 쓰기 토큰을 넘기지 않는다.
- 게시되는 모든 텍스트가 출력 검열을 통과한다. 자격증명 모양은 가리고, 실행 중인 프로세스의 실제 비밀값이 섞이면 게시 자체를 중단한다.
- 워크플로는 `pull_request` 만 쓴다. `pull_request_target` 은 시크릿이 주입된 상태로 PR 브랜치 코드를 실행하므로 쓰지 않는다. 포크 PR 은 실행하지 않는다.

인젝션 시도를 심은 diff 로 검증했다. 미끼 비밀값 유출 없음, 승인 요구 불응, 시도 자체를 `critical`/`security` 로 보고했다.

---

## 검증

```bash
node review.mjs --self-test
```

프레임워크 없이 `node:assert` 로 9개를 확인한다. diff 앵커 계산, 범위를 벗어난 지적 처리, suggestion 줄 수 가드, 출력 검열, 자격증명 격리. 틀려도 겉보기엔 잘 도는 지점들로 골랐다.

---

## 아직 없는 것

| 생략 | 언제 추가 |
| --- | --- |
| 대용량 PR 청크 분할 | 실제 PR 이 300KB 상한을 넘을 때. 지금은 초과분을 제외하고 목록으로 남긴다 |
| 증분 리뷰(직전 리뷰 이후 커밋만) | push 마다 재리뷰가 시끄러워질 때 |
| `@봇` 멘션 대화 응답 | 리뷰 게시가 안정화된 뒤 |
| 저장소별 설정 파일 | 두 번째 저장소가 실제로 다른 설정을 원할 때. 지금은 플래그로 충분하다 |
