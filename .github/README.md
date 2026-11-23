# Vortex GPGPU · SKKU URP fork

[vortexgpgpu/vortex](https://github.com/vortexgpgpu/vortex)를 fork해 학부 연구(URP)를 진행하는 저장소다.
Vortex 자체에 대한 설명은 [upstream README](https://github.com/vortexgpgpu/vortex#readme)에 있다.

## 2026 겨울 URP (진행 중)

- 주제: Vortex 가상 메모리(VM). 세부 주제는 확정 후 갱신한다.
- 작업 branch: `urp/2026-winter`
- 기준 upstream commit: [`1061088aa`](https://github.com/vortexgpgpu/vortex/commit/1061088aa91bd6f295d91945832acdc8408223c4) (2026-11-21, upstream CI 통과). 모든 측정 결과는 이 commit을 기준으로 한다.

## Branch

| Branch | 내용 | Ruleset |
|---|---|---|
| `urp/2026-winter` | 겨울 URP 연구 (default branch) | 삭제, force push, merge commit 금지 |
| `master` | upstream `master`의 mirror | Sync fork로만 갱신 |
| `archive/urp-2026-1` | 2026-1 URP 최종 상태 (README, 발표 자료) | 읽기 전용 |
| `exp/upstream-sched` | 2026-10 upstream 기반 issue arbiter 실험 (백업) | 없음 |
| `feat/*`, `exp/lsu-pending-pool`, `lkn` | 2026-1 팀 작업 | 없음 |
| `feature/…`, `fix/…`, `perf/…` 등 | upstream PR용. [CONTRIBUTING](https://github.com/vortexgpgpu/vortex/blob/master/CONTRIBUTING.md)의 이름 규칙을 따른다 | 없음 |

## 자주 쓰는 명령

```bash
# 처음 clone할 때 (default branch인 urp/2026-winter가 checkout된다)
git clone --recursive git@github.com:bbo-sseong/vortex.git && cd vortex
git remote add upstream https://github.com/vortexgpgpu/vortex.git
git config remote.pushDefault origin
git config pull.rebase true

# master를 upstream 최신 상태로 맞춘다 (-b master 필수)
gh repo sync bbo-sseong/vortex -b master && git fetch origin

# upstream PR: 최신 master에서 branch를 만들고 연구 commit만 옮긴다
git switch -c perf/<name> origin/master
git cherry-pick -x <commit>
git push -u origin perf/<name>
gh pr create -R vortexgpgpu/vortex --base master
```

- 연구 branch에는 upstream을 merge하지 않는다. 필요한 upstream 수정만 `git cherry-pick -x`로 가져온다.
- `-b` 없이 `gh repo sync`를 실행하거나 연구 branch 화면에서 Sync fork를 누르면 upstream을 연구 branch로 merge하려 한다. Ruleset이 이를 막지만, 애초에 실행하지 않는다.
- upstream 도구(`ci/change_impact.py`, `ci/logic_gate.py --against-rev`)는 `origin/master`가 upstream master라고 가정한다. 사용 전에 위의 동기화를 먼저 실행한다.
- 이 파일(`.github/README.md`)은 fork 전용이므로 upstream PR에 포함하지 않는다.
