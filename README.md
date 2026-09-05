# muse-review

조직 공통 AI 코드 리뷰어. PR 이 열리면 Muse Code CLI 가 diff 를 읽고 한국어로
총평 + 인라인 코멘트를 남깁니다.

레포마다 워크플로 yml(약 110줄)과 리뷰 프롬프트(39줄)를 복사해 두던 것을 여기
한 곳으로 모은 것입니다. 도입 레포에는 호출부 10여 줄만 남습니다.

## 도입

레포에 `.github/workflows/muse-review.yml` 을 만듭니다.

```yaml
name: Muse Code Review

on:
  # PR 최초 생성 시에만 자동 실행. 이후 추가 커밋(synchronize)에는 반응하지 않습니다.
  pull_request:
    types: [opened]
  # 이후 재리뷰는 PR 에 `/review` 코멘트를 달아 수동으로 트리거합니다.
  issue_comment:
    types: [created]

jobs:
  review:
    uses: k-devcon/muse-review/.github/workflows/review.yml@v1
    secrets:
      META_API_KEY: ${{ secrets.META_API_KEY }}
    permissions:
      contents: read
      pull-requests: write
      issues: write
```

그리고 레포에 `META_API_KEY` 시크릿(Model API 대시보드에서 발급한 키)이 있어야
합니다. org 레벨 시크릿이 있으면 그것도 `secrets.META_API_KEY` 로 잡힙니다.

`secrets: inherit` 이 아니라 위처럼 하나만 명시적으로 넘기세요. inherit 은 호출
레포의 **모든** 시크릿을 중앙 워크플로에 넘깁니다.

```bash
gh secret set META_API_KEY --repo k-devcon/<repo>
```

이게 전부입니다. 대부분의 레포는 프롬프트를 따로 두지 않아도 됩니다.

## 레포별 리뷰 관점 추가 (선택)

범용 관점(버그/보안/품질/성능/테스트)만으로 부족한 레포 — 예를 들어 인프라
설정 레포처럼 봐야 할 것이 다른 경우 — 는 `.github/muse-review-focus.md` 를
두면 base 프롬프트의 `__FOCUS__` 자리에 끼워 넣습니다. 공통 규약(배지 체계,
코멘트 방식)은 그대로 물려받고 관점만 덧붙는 구조입니다.

실제 예시는 `k-devcon/k-devcon-nginx` 를 참고하세요.

**focus 파일은 호출 레포의 기본 브랜치에서만 읽습니다.** 체크아웃은 PR head 인데
프롬프트를 PR 쪽에서 읽으면, PR 하나로 리뷰어 동작 자체를 바꿀 수 있게 되기
때문입니다. 그래서 focus 를 처음 추가하는 PR 에서는 아직 반영되지 않고, 머지된
다음 PR 부터 적용됩니다.

## base 프롬프트 고치기

`prompt/base.md` 를 고치면 됩니다. 그냥 마크다운 파일입니다.

주의할 점은 두 가지뿐입니다:

- `__FOCUS__` 는 **단독 줄**로 남겨두세요. focus 파일이 있으면 그 자리에 끼워
  넣고, 없으면 빈 줄로 치환됩니다. 앞 목록과 `리뷰 방식:` 사이를 띄우는 역할도
  겸하므로 지우면 두 블록이 붙어버립니다.
- `__REPO__` / `__PR_NUMBER__` / `__HEAD_SHA__` 는 렌더 단계에서 치환됩니다.
  치환 안 된 `__대문자__` 가 남아 있으면 스텝이 실패합니다.

수정 후 로컬 검증:

```bash
ruby -ryaml -e 'y=YAML.load_file(".github/workflows/review.yml"); print y["jobs"]["review"]["steps"].find{|s|s["name"]=="Render review prompt"}["run"]' > /tmp/render.sh
mkdir -p /tmp/ws/.muse-base/prompt && cp prompt/base.md /tmp/ws/.muse-base/prompt/
RUNNER_TEMP=/tmp/x GITHUB_WORKSPACE=/tmp/ws REPO=k-devcon/foo DEFAULT_BRANCH=main \
  PR_NUMBER=1 HEAD_SHA=abc FOCUS_PATH=.github/muse-review-focus.md bash /tmp/render.sh
cat /tmp/x/muse/prompt.md
```

`base.md` 는 워크플로와 **같은 릴리스 태그**에서 읽습니다(`base-ref` 입력,
기본값 `v1`). main 에 머지해도 `v1` 태그를 옮기기 전까지는 반영되지 않습니다.

`v1` 브랜치에서 프롬프트 변경을 미리 시험하려면 호출부에서 넘기세요:

```yaml
    with:
      base-ref: my-prompt-branch
```

> **왜 자동으로 못 알아내나.** 실행 중인 reusable workflow 의 ref 를 컨텍스트에서
> 얻을 방법이 없습니다. 두 가지를 시도했고 둘 다 실패했습니다:
>
> - `github.workflow_ref` — reusable workflow 안에서도 `github.*` 컨텍스트는
>   **호출한 쪽**을 가리킵니다. 이 레포가 아니라 호출 레포를 체크아웃하므로
>   체크아웃 스텝은 성공하고 그 다음에 "base.md 없음" 으로 죽습니다.
> - `github.job_workflow_sha` — 이 런너의 컨텍스트에 **존재하지 않습니다**(빈 값).
>
> 그래서 태그를 명시적으로 고정합니다. `v2` 를 낼 때는 그 파일의 `base-ref`
> 기본값도 `v2` 로 바꿔야 합니다.

## 입력값

전부 선택입니다. 기본값으로 충분한 경우가 대부분입니다.

| 입력 | 기본값 | 설명 |
|---|---|---|
| `model` | `muse-spark-1.3-contributor` | |
| `reasoning-effort` | `medium` | |
| `max-model-steps` | `30` | |
| `focus-path` | `.github/muse-review-focus.md` | 레포별 관점 파일 |
| `base-ref` | `v1` | base 프롬프트를 가져올 ref. 보통 건드리지 않음 |
| `runs-on` | `self-hosted` | |
| `timeout-minutes` | `20` | 실측 리뷰 소요는 5~6분 수준 |

```yaml
    with:
      reasoning-effort: high
      timeout-minutes: 30
```

## 버전 고정

`@v1` 태그를 쓰세요. `@main` 으로 걸면 여기 머지되는 순간 전 레포 리뷰어 동작이
바뀝니다.

`v1` 은 이동하는 태그(moving tag)라 호환되는 개선은 자동으로 따라옵니다.
호환이 깨지는 변경은 `v2` 로 나가고, `v1` 은 그대로 둡니다.

```bash
# 릴리스
git tag -f v1 && git push -f origin v1
```

## 보안

이 레포에 push 할 수 있는 사람은 **모든 레포의 리뷰어 동작을 바꿀 수 있습니다.**
리뷰어는 self-hosted 러너에서 `--approval-mode never --sandbox-network enabled`
로 돕니다. main 브랜치 보호를 반드시 걸어 두세요.

리뷰 job 의 권한은 `contents: read` 이고 코멘트 작성용으로만
`pull-requests: write` / `issues: write` 를 씁니다. 코드를 push 하지 않습니다.

`/review` 코멘트 트리거는 `OWNER` / `MEMBER` / `COLLABORATOR` 로 제한됩니다.
외부 기여자가 코멘트로 러너를 돌릴 수 없습니다.
