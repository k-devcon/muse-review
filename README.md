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
    secrets: inherit
    permissions:
      contents: read
      pull-requests: write
      issues: write
```

그리고 레포에 `META_API_KEY` 시크릿(Model API 대시보드에서 발급한 키)이 있어야
합니다. `secrets: inherit` 이므로 org 레벨 시크릿이 있으면 그것도 잡힙니다.

```bash
gh secret set META_API_KEY --repo k-devcon/<repo>
```

이게 전부입니다. 대부분의 레포는 프롬프트를 따로 두지 않아도 됩니다.

> **이 레포는 비공개입니다.** 그래서 다른 레포가 호출하려면 여기 Actions 접근
> 설정이 조직에 열려 있어야 합니다. 한 번만 해두면 됩니다.
>
> ```bash
> gh api -X PUT repos/k-devcon/muse-review/actions/permissions/access \
>   -f access_level=organization
> ```
>
> 안 열려 있으면 호출 레포에서 워크플로가 시작조차 못 하고
> `workflow was not found` 류로 실패합니다.

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

base 프롬프트는 별도 `.md` 파일이 아니라 `.github/workflows/review.yml` 의
`Render review prompt` 스텝 안 heredoc(`MUSE_BASE_EOF`)에 들어 있습니다.

이 레포가 비공개라서 그렇습니다. 호출 레포의 `GITHUB_TOKEN` 은 자기 레포에만
유효해서 여기 있는 파일을 크로스레포로 읽지 못합니다. 프롬프트를 별도 파일로
두려면 조직 시크릿에 PAT 을 하나 더 만들어 돌려야 하는데, 워크플로 yml 자체는
Actions 가 알아서 가져오므로 프롬프트를 그 안에 넣으면 토큰이 아예 필요
없습니다. 덤으로 워크플로와 프롬프트가 같은 커밋에 묶여 버전이 어긋날 수 없고요.

고칠 때 주의할 점:

- heredoc 본문은 YAML 블록 스칼라 안이라 **들여쓰기가 10칸 기준**입니다.
  프롬프트에서 컬럼 0 이어야 하는 줄은 10칸, 4칸 들여써야 하는 줄은 14칸입니다.
- `__FOCUS__` 는 **단독 줄**이어야 합니다. focus 파일이 없으면 빈 줄로 치환되는데,
  앞 목록과 `리뷰 방식:` 사이를 띄우는 역할도 겸합니다. 지우지 마세요.
- `__REPO__` / `__PR_NUMBER__` / `__HEAD_SHA__` 는 렌더 단계에서 치환됩니다.
  치환 안 된 `__대문자__` 가 남아 있으면 스텝이 실패합니다.

수정 후 로컬 검증:

```bash
ruby -ryaml -e 'y=YAML.load_file(".github/workflows/review.yml"); print y["jobs"]["review"]["steps"].find{|s|s["name"]=="Render review prompt"}["run"]' > /tmp/render.sh
RUNNER_TEMP=/tmp/x REPO=k-devcon/foo DEFAULT_BRANCH=main PR_NUMBER=1 HEAD_SHA=abc \
  FOCUS_PATH=.github/muse-review-focus.md bash /tmp/render.sh
cat /tmp/x/muse/prompt.md
```

## 입력값

전부 선택입니다. 기본값으로 충분한 경우가 대부분입니다.

| 입력 | 기본값 | 설명 |
|---|---|---|
| `model` | `muse-spark-1.3-contributor` | |
| `reasoning-effort` | `medium` | |
| `max-model-steps` | `30` | |
| `focus-path` | `.github/muse-review-focus.md` | 레포별 관점 파일 |
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
