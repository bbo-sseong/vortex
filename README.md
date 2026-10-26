# Vortex GPGPU — Warp Scheduler 연구

> **SKKU 학부생 연구학점제(URP) 2026-1** · 연구기간 2026.03.02 ~ 06.19
> 연구제목: *Vortex GPGPU 시뮬레이터 기반 Warp Scheduler · 마이크로아키텍처 병목 규명*

오픈소스 RISC-V GPGPU인 [Vortex](https://github.com/vortexgpgpu/vortex)의 사이클 단위 C++ 시뮬레이터(SimX) 위에서 워프 스케줄러 정책을 구현·분석하고, **왜 Vortex에서는 스케줄러를 바꿔도 성능이 거의 변하지 않는가**를 코드 수준에서 규명한 연구.

결론부터: 1차 원인은 **core-wide LSU 입력 큐 병목**이었다. 코어 전체가 2칸짜리 LSUQ를 공유해 in-flight load가 2개로 묶이고, 그 결과 스케줄러가 고를 수 있는 ready warp 자체가 거의 없었다. 이를 해소하면 IPC가 2배 이상 올랐고, 메모리 contention을 인위적으로 재현하자 원 논문이 보고한 스케줄러 효과(+8~9%)가 재현되었다.

---

## 팀

3인 공동 연구다. 커밋 작성자가 그대로 보존되어 있다.

| GitHub | 비고 |
|---|---|
| [`bbo-sseong`](https://github.com/bbo-sseong) | 이 저장소 소유자. SimX 스케줄러 구현 및 병목 분석 |
| [`wheel0012`](https://github.com/wheel0012) (Taejun Lee) | RTL 실험, SimX 성능 메트릭, 벤치마크 정비 |
| `lkn0403` | RTL GTO, gCAWS greedy 로직 |

원본 공동 작업 저장소: <https://github.com/wheel0012/vortex_aware_scheduler>

## 브랜치

업스트림 `master`(`31e4765`) 이후의 커밋만 센 수치다.

| 브랜치 | 커밋 | 작성자 분포 | 기간 | 내용 |
|---|---|---|---|---|
| [`feat/gto`](../../tree/feat/gto) | 3 | bbo-sseong 3 | 04-10 | GTO를 에뮬레이터 schedule 단계에 도입 (연구 시작점) |
| [`feat/gcaws`](../../tree/feat/gcaws) | 34 | bbo-sseong 26 · wheel0012 8 | 04-10 ~ 05-15 | gCAWS criticality 스케줄러, CACP, iPAWS 적응형, 워크로드 sweep |
| [`feat/gcaws_greedy`](../../tree/feat/gcaws_greedy) | 14 | bbo-sseong 13 · lkn0403 1 | 04-10 ~ 05-14 | iPAWS paper-strict 정렬, gCAWS greedy 변형 |
| [`feat/gto-mshr-aware`](../../tree/feat/gto-mshr-aware) | 20 | wheel0012 13 · bbo-sseong 7 | 04-10 ~ 05-13 | MSHR 압력을 반영한 GTO, rodinia 벤치마크 정비 |
| [`feat/issue-arbiter`](../../tree/feat/issue-arbiter) | 33 | wheel0012 16 · bbo-sseong 13 · lkn0403 4 | 05-24 ~ 06-05 | RTL issue arbiter, fetch/issue 분리, locality 메트릭, 최종 sweep |
| [`lkn`](../../tree/lkn) | 1 | lkn0403 1 | 04-09 | RTL `VX_schedule.sv` GTO |
| `master` | — | — | — | 업스트림 Vortex 원본 (`31e4765`) |

스케줄러 설계 결정의 근거와 ablation 데이터는 [`feat/gcaws` 브랜치의 `urp_analysis.md`](../../blob/feat/gcaws/urp_analysis.md)에, 실험 기록은 각 브랜치의 `worklogs/`에 있다.

## 연구 내용

### 1. Configuration 탐색 — 상용 GPU 구조를 그대로 옮길 수 없다

CCWS(GTX285)·CAWA(GTX480)·RTX3090 스펙을 조사해 RTX3090을 약 1/20로 downscale 했다(1 cluster, 16 core, 16 lane, 32 warp, 128KB L1D / 128B line / 16 MSHR / 8-way, 256KB L3). 그런데 compute-bound로 분류되는 sgemm3에서조차 L1 miss로 MSHR이 마비되고 시뮬레이션 시간이 폭증했다. 상용 GPU 구성을 그대로 따르는 것이 부적절하다는 결론을 얻고, 이후로는 **스케줄러 간 차이가 드러나는 구성**을 찾는 데 초점을 옮겼다.

### 2. 스케줄러 구현

RR(Loosely Round-Robin) · GTO(Greedy-Then-Oldest) · gCAWS(greedy Criticality-Aware)를 빌드 시 선택 가능하도록 구현하고, arbiter 거동을 사이클 단위로 시각화해 각각의 fairness · greediness · criticality 동작을 검증했다.

- **gCAWS** — CAWA(Lee et al., ISCA 2015). criticality = `nInst × CPI_avg + nStall`, argmax 선택 + oldest-ready tie-break
- **CACP** — CAWA의 캐시 관리(way reservation + SHiP-CB RRIP). 측정 결과 **모든 워크로드에서 IPC 손해(−1.5% ~ −7.5%)** 였고 FPGA 면적도 정당화되지 않아 RTL 타겟에서 제외, 빌드 토글로만 보존
- **iPAWS** — iPAWS(Lee et al., HPCA 2016). 명령 issue 패턴이 concave면 gCAWS, convex면 RR로 전환하는 적응형

iPAWS는 논문 가정이 Vortex와 어긋나는 지점들을 데이터로 확인하고 재설계했다.

| 논문 가정 | Vortex 실제 | 대응 |
|---|---|---|
| birth-order GTO (생성 순서로 oldest) | recency-based GTO — stall하면 timestamp가 리셋되어 **stall 잦은 warp가 newest로 강등** | WOI 필터는 paper-strict로 두되 본 환경에서 비활성임을 명시 |
| 시간차 warp dispatch | `wspawn` 한 사이클에 32 warp 동시 활성화 — birth-order 정보 자체가 없음 | Adapt 트리거를 periodic → **kernel boundary(wspawn)** 로 변경 |
| `btime` = barrier wait만 | 첫 구현은 scoreboard stall까지 합산 | `is_barrier_stalled()` 헬퍼로 barrier만 카운트 |
| Recover phase로 진행도 균형 | dcache miss penalty가 커서 min-warp도 진도를 못 나감 — 6가지 종료조건 모두 cycle cap으로만 종료 | 기본 비활성, ablation 재현 가능하도록 토글 유지 |

### 3. 스케줄러 무감각 원인 규명 ← 핵심 기여

기본 환경에서 RR·GTO·gCAWS의 IPC 차이는 **±2% 노이즈** 수준이었다(전구간 0.39%, region 2.15%).

**진단** — `scoreboard_block_ratio`(스케줄러가 검토한 warp 후보 중 scoreboard로 막힌 비율) 지표를 새로 만들어 측정하니 kmeans에서 **97%**. 매 사이클 ready warp 개수 분포를 보니 ready가 0인 사이클이 77~78%, 1개 이하가 90%, **2개 이상은 9%뿐**이었다. 스케줄러가 선택을 수행할 수 있는 순간 자체가 거의 없었다.

**구조적 원인** — 코드를 따라가 `LSUQ_IN_SIZE = 2`를 32개 warp 전체가 공유하는 core-wide 자원임을 확인했다. 근거는 상태 배열 크기가 `NUM_WARPS`가 아닌 `NUM_LSU_BLOCKS=1`, `LsuUnit`이 Core당 1개 생성, 큐 선택 인덱스가 `wid`가 아닌 `block_idx`, 큐가 차면 모든 warp의 load를 거절한다는 점이다. 코어 전체 동시 진행 load가 2개로 묶여 MLP가 사실상 1이었다.

| LSUQ | IPC (RR) | IPC (GTO) | scheduler idle | scoreboard_block_ratio | ready ≥ 2 |
|---|---|---|---|---|---|
| 2 | 4.43 | 4.29 | 72% | 97.3% | 9% |
| 8 | **10.45** (×2.36) | **9.82** (×2.29) | 34% | 91.8% | 20% |

**원인 배제** — fetch rate 조정, 메모리 latency 감소, DDR3 타이밍 치환, DRAM bank 수 변경, L2 비활성화는 순차 접근 워크로드에서 효과가 없었고, per-warp out-of-order ibuffer는 오히려 IPC가 1.2~3.0% 하락했다.

**contention 재현** — MSHR을 2로 인위 축소해(NUM_WARPS=8) 원 논문의 GDDR5 환경 contention을 재현하자 gCAWS가 RR 대비 **kmeans(f96) +8.18%** (IPC 7.78 → 8.42), **BFS(hh16k) +9.24%** (2.45 → 2.67). 스케줄러 효과가 메모리 서브시스템의 contention/divergence에 의존함을 정량적으로 입증했다.

### 4. 워크로드 분석

| 워크로드 | 특성 | 결과 |
|---|---|---|
| kmeans | feature 배열 연속 접근 → coalescing 큼, intra-warp locality 높음, 모든 warp가 동일 루프 횟수로 진행 | input이 커질수록 **RR이 GTO보다 우세** (예측과 반대) |
| SGEMM3 | local memory 타일링 + barrier | tile이 커지면 **GTO로 역전** (n=128, tile=16에서 +6.6%, IPC 5.53 → 5.90) |
| BFS | id 기반 불규칙 접근(uncoalesced), dcache hit 6~14% | RR ≈ GTO. GTO가 load latency는 8.3% 줄였으나 I-buffer stall이 17%p 증가 |

추가로 Vortex와 GPGPU-Sim의 arbiter 거동을 L1D 용량 sweep으로 비교했고, memory coalescer에서 **같은 cache line인데도 여러 request로 분할되는 경우를 약 3만 건** 관측해 개선 여지를 확인했다. 4-way loop unrolling을 적용하자 16KB 캐시부터 GTO의 same-warp reuse hit이 개선되었다.

### 결론

스케줄러 정책의 효과는 **워프 간 진행도 차이**를 통해 발현되며, 이는 워크로드의 구조적 비균일성과 메모리 서브시스템의 변동에 좌우된다. Vortex 기본 설정의 작은 core-wide LSUQ, 구조적으로 균일한 워크로드, 컴파일러가 노출하는 제한된 warp당 MLP, coalescer 비효율이 겹쳐 진행도 분산이 생기지 않았고, 그래서 스케줄러가 차별화될 여지가 좁았다.

향후 과제: ISA 확장을 통한 paper-faithful gCAWS, memory coalescer 개선, loop unrolling을 통한 warp당 MLP 확대, L2를 초과하는 입력으로 자연적 divergence 유도, 그리고 SimX 검증을 RTL·FPGA로 올리는 full-stack 완성.

---

## `bbo-sseong` 기여

전체 ~105개 팀 커밋 중 **43개**. 작업은 **SimX(`sim/simx/`) 쪽에 집중**되어 있다.

- RR · GTO · gCAWS 스케줄러 구현과 정책 전환 구조
- CACP (way reservation + SHiP-CB RRIP) 구현 및 ablation, 제외 결정의 근거 측정
- iPAWS 적응형 스케줄러 — paper Algorithm 1 기반 재작성, WOI 필터 · Adapt 트리거 · barrier-only btime · Recover phase를 Vortex 환경에 맞춰 재설계하고 각각 데이터로 근거 제시
- `scoreboard_block_ratio` 지표 신설, ready warp 분포 측정, CPL/CRIT_SNAP 진단 계측
- core-wide LSUQ 병목 규명 및 LSUQ sweep
- MSHR 축소를 통한 contention 재현 실험
- warp-wide memory coalescer 및 split rate 메트릭, DRAM RowPolicy 토글
- bfs hub-graph 생성기, 워크로드 입력 정비, sweep 하네스 및 결과 리포트 (`worklogs/`)
- `urp_analysis.md` 등 연구 문서

계획서상으로는 12-13주차에 RTL 설계, 14-15주차에 RTLsim 검증이 예정되어 있었으나, 실제로는 3장의 병목 규명이 연구의 중심이 되었고 full-stack RTL·FPGA는 최종 보고서에서 향후 과제로 정리되었다.

## 문서

[`docs/urp/`](docs/urp/)에 본인이 작성한 산출물만 넣었다. 제3자 논문 PDF와 강의 자료는 저작권 때문에 제외했다.

| 파일 | 내용 |
|---|---|
| `01_URP_계획서.docx` | 초기 연구 로드맵 (주차별 계획) |
| `02_중간발표_2026-04-17.pdf` | 중간 발표 |
| `03_최종발표_2026-05-31.pptx` | 최종 발표 — 병목 진단 과정 중심 |
| `04_연구결과보고서_2026-1.hwpx` | 학부생 연구학점제 최종 결과보고서 |
| `note_*.docx` | GPU 스케줄러 정리, CAWA 논문 정리, Vortex 구조 분석 |
| `gen_hub_graph.py` | warp 불균형 실험용 BFS hub-graph 생성기 |

## 참고 문헌

1. T. G. Rogers, M. O'Connor, T. M. Aamodt, "Cache-Conscious Wavefront Scheduling," *MICRO*, 2012, pp. 72–83.
2. S.-Y. Lee, A. Arunkumar, C.-J. Wu, "CAWA: Coordinated Warp Scheduling and Cache Prioritization for Critical Warp Acceleration of GPGPU Workloads," *ISCA*, 2015, pp. 515–527.
3. M. Lee, G. Kim, J. Kim, W. Seo, Y. Cho, S. Ryu, "iPAWS: Instruction-Issue Pattern-based Adaptive Warp Scheduling for GPGPUs," *HPCA*, 2016, pp. 370–381.

## 저장소 메모

- 업스트림 Vortex의 README는 [`UPSTREAM_README.md`](UPSTREAM_README.md)로 옮겼다.
- `feat/issue-arbiter`는 공동 작업 저장소에서 복제한 뒤, 실수로 커밋되어 있던 CAWA·iPAWS 논문 PDF 2개를 히스토리에서 제거했다. 그 과정에서 해당 브랜치의 마지막 13개 커밋만 SHA가 바뀌었고 작성자·날짜·메시지는 모두 보존되어 있다. 다른 브랜치와 업스트림 히스토리는 원본과 동일하다.
- `tests/opencl/bfs/graph1MW_6.txt`(61MB)는 팀이 커밋한 벤치마크 입력으로, GitHub 권장 파일 크기를 초과한다.

## License

Vortex는 Apache License 2.0으로 배포된다. [`LICENSE`](LICENSE) 참고.
원본 저장소: <https://github.com/vortexgpgpu/vortex>
