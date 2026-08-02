# ai-code-review

CodeRabbit 형식의 AI 코드 리뷰를 GitHub PR 에 자동으로 남긴다.
**의존성 0개** — `gh` CLI + `claude` CLI + Node 22 만 있으면 된다. `npm install` 없음.

PR 하나에 두 개를 남긴다.

| 게시물 | 내용 |
| --- | --- |
| **walkthrough 코멘트** | 변경 요약 · Changes 표 · 리뷰 난이도 추정 · **mermaid 흐름도** · 접힌 nitpick |
| **인라인 리뷰** | 문제 줄에 직접 달리는 코멘트. `` `42-45`: _🔐 Security_ | _🔴 Critical_ `` 헤더 + 커밋 가능한 suggestion |

## 심각도

| | 뜻 | 게시 위치 |
| --- | --- | --- |
| 🔴 `critical` | 데이터 손실·보안·크래시·의도와 반대로 동작 | 인라인 |
| 🟠 `major` | 특정 조건에서 재현되는 버그, 복구 불가 상태 | 인라인 |
| 🟡 `minor` | 리팩터, 중복 판정, 오해를 부르는 네이밍 | 인라인 |
| 🔵 `nit` | 오타·스타일 | 요약에 접어서 |

---

## 로컬 실행

```bash
# 확인만 (게시 안 함) — review-115.md / .json 생성
node review.mjs https://github.com/OWNER/REPO/pull/115

# 번호 + --repo 로도 된다
node review.mjs 115 --repo OWNER/REPO

# 실제 게시
node review.mjs 115 --repo OWNER/REPO --post
```

`--post` 를 붙이지 않으면 **항상 dry-run** 이다. 되돌리기 번거로운 게시를 실수로 하지 않도록 기본값을 그렇게 뒀다.

| 플래그 | 기본값 | 설명 |
| --- | --- | --- |
| `--repo owner/name` | PR URL 에서 추출 | 대상 저장소 |
| `--post` | off | 실제 게시. 없으면 로컬 파일만 |
| `--model` | `opus` | `sonnet` 으로 낮추면 싸고 빠르다 |
| `--budget 2` | 무제한 | PR 1건당 최대 USD |
| `--force` | off | 같은 커밋 중복 리뷰 방지를 무시 |
| `--self-test` | — | diff 파서 검증 |

사전 조건: `gh auth login` (`repo` scope), `claude` 로그인 완료.

---

## 다른 저장소에 연결하기

이 저장소를 GitHub 에 올린 뒤, 리뷰를 붙일 저장소마다 워크플로 한 개만 넣으면 된다.

**1. 토큰 발급** (한 번만)

```bash
claude setup-token
```

출력된 토큰을 대상 저장소(또는 organization) 시크릿 `CLAUDE_CODE_OAUTH_TOKEN` 으로 등록한다.
organization 시크릿으로 등록하면 저장소를 추가할 때마다 다시 할 필요가 없다.

```bash
gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo OWNER/REPO
```

**2. 워크플로 복사**

[`examples/pr-review.yml`](examples/pr-review.yml) 을 대상 저장소의 `.github/workflows/ai-review.yml` 로 그대로 복사한다.

```yaml
- uses: Jung-eunwoo/ai-code-review@main
  with:
    pr: ${{ github.event.number }}
  env:
    CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
    GH_TOKEN: ${{ github.token }}
```

`GH_TOKEN` 은 워크플로가 자동 발급하므로 저장소마다 PAT 를 만들 필요가 없다.
도입 초기에 반응을 보고 싶으면 `post: "false"` 로 두면 게시 없이 Actions 로그에만 남는다.

---

## 동작 방식

```
gh pr view/diff  →  생성물 제외 · 크기 상한
                 →  앵커 맵 수집 (diff hunk 안의 줄번호)
                 →  claude -p (도구 차단, 임시 디렉토리에서 실행)
                 →  JSON 파싱 · 앵커 대조
                 →  walkthrough 코멘트 + 인라인 리뷰 게시
```

설계상 중요한 세 지점:

**앵커 검증.** GitHub 리뷰 API 는 인라인 코멘트의 줄번호가 diff hunk 밖이면 **리뷰 전체를 422 로 거절**한다. 하나만 틀려도 멀쩡한 나머지가 같이 죽는다. 그래서 diff 를 직접 파싱해 코멘트 가능한 줄을 미리 모아두고 게시 전에 대조한다. 그래도 거절되면 본문 코멘트로 폴백해서 지적을 잃지 않는다.

**조용한 누락 금지.** 앵커에 못 맞춘 지적은 버리지 않고 요약의 "위치를 매칭하지 못한 지적" 절로 강등한다. 크기 상한에 걸려 제외한 파일도 목록으로 남긴다. 사라지면 "문제 없음"으로 잘못 읽히기 때문이다.

**suggestion 안전장치.** ` ```suggestion ` 블록은 지정한 줄 범위를 그대로 치환하므로, 줄 수가 어긋난 채 커밋되면 코드가 깨진다. 줄 수가 정확히 맞을 때만 committable 로 내고, 어긋나면 커밋 버튼이 붙지 않는 일반 블록 + 경고 문구로 낮춘다. 실제 PR #115 리뷰에서 이 가드가 한 건 걸러냈다.

멱등성은 코멘트 본문의 `<!-- ai-review:<sha> -->` 마커로 처리한다 — 같은 커밋이면 건너뛴다. DB 도 상태 저장소도 없다.

```bash
node review.mjs --self-test   # diff 파서 · 앵커 · suggestion 가드 검증
```

---

## 아직 없는 것

| 생략 | 언제 추가 |
| --- | --- |
| 대용량 PR 청크 분할 | 실제 PR 이 300KB 상한을 넘을 때 |
| 증분 리뷰(직전 리뷰 이후 커밋만) | push 마다 재리뷰가 시끄러워질 때 |
| `@봇` 멘션 대화 응답 | 리뷰 게시가 안정화된 뒤 |
| 저장소별 설정 파일 | 두 번째 저장소가 실제로 다른 설정을 원할 때. 지금은 플래그로 충분 |
