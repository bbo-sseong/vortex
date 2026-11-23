# Vortex GPGPU · SKKU URP 포크

[vortexgpgpu/vortex](https://github.com/vortexgpgpu/vortex)를 포크해 학부 연구(URP)를 하는 저장소예요.
Vortex 자체 설명은 [upstream README](https://github.com/vortexgpgpu/vortex#readme)에 있어요.

## 2026 겨울 URP (진행 중)

- 주제: Vortex 가상 메모리(VM). 세부 주제가 정해지면 여기에 적을게요.
- 작업 브랜치: `urp/2026-winter`
- 기준 upstream 커밋: [`1061088aa`](https://github.com/vortexgpgpu/vortex/commit/1061088aa91bd6f295d91945832acdc8408223c4) (2026-11-21, upstream CI 통과). 측정 결과는 모두 이 커밋 기준이에요.

## 브랜치

| 브랜치 | 내용 | 보호 규칙 |
|---|---|---|
| `urp/2026-winter` | 겨울 URP 연구 (기본 브랜치) | 삭제, force push, merge 커밋 금지 |
| `master` | upstream `master`를 그대로 따라가는 미러 | Sync fork로만 갱신 |
| `archive/urp-2026-1` | 2026-1 URP 최종 상태 (README, 발표 자료) | 읽기 전용 |
| `exp/upstream-sched` | 2026-10 upstream 위의 issue arbiter 실험 (백업) | 없음 |
| `feat/*`, `exp/lsu-pending-pool`, `lkn` | 2026-1 팀 작업 | 없음 |
| `feature/…`, `fix/…`, `perf/…` 등 | upstream PR용. [CONTRIBUTING](https://github.com/vortexgpgpu/vortex/blob/master/CONTRIBUTING.md)의 이름 규칙을 따라요 | 없음 |

## 자주 쓰는 명령

```bash
# 처음 클론할 때 (기본 브랜치 urp/2026-winter가 체크아웃돼요)
git clone --recursive git@github.com:bbo-sseong/vortex.git && cd vortex
git remote add upstream https://github.com/vortexgpgpu/vortex.git
git config remote.pushDefault origin
git config pull.rebase true

# master를 upstream 최신으로 맞추기 (-b master를 꼭 붙여요)
gh repo sync bbo-sseong/vortex -b master && git fetch origin

# upstream PR: 최신 master에서 따서 연구 커밋만 옮겨요
git switch -c perf/<이름> origin/master
git cherry-pick -x <커밋>
git push -u origin perf/<이름>
gh pr create -R vortexgpgpu/vortex --base master
```

- 연구 브랜치에는 upstream을 merge하지 않아요. 필요한 upstream 수정만 `git cherry-pick -x`로 가져와요.
- `-b` 없이 `gh repo sync`를 돌리거나 연구 브랜치 화면에서 Sync fork를 누르면 upstream이 연구 브랜치로 merge되려고 해요. 보호 규칙이 막아 주지만, 처음부터 하지 않는 게 좋아요.
- upstream 도구(`ci/change_impact.py`, `ci/logic_gate.py --against-rev`)는 `origin/master`가 upstream master라고 가정해요. 쓰기 전에 위 동기화부터 해요.
- 이 파일(`.github/README.md`)은 포크 전용이라 upstream PR에 넣지 않아요.
