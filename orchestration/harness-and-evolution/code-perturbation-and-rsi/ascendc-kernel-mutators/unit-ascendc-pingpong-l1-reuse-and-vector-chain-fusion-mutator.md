---
category: mutator
conditions:
  hardware:
  - Ascend 910B (A2)
  - Ascend 910C (A3)
  language: Ascend C
  repository: cann/cannbot-knowledge
  revision: be5882ef29fa9943305085652d2025a038ba491c
  strategy_ids:
  - P19
  - P69
  - P71
  - P138
  - P145
created_at: '2026-10-21'
domain_path: orchestration/harness-and-evolution/code-perturbation-and-rsi/ascendc-kernel-mutators
estimate_tokens: 680
evidence_strength: software_contract
id: unit-ascendc-pingpong-l1-reuse-and-vector-chain-fusion-mutator
inherits_from:
- unit-controlled-code-perturbation-and-evolution-strategy-workflow
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/knowledge/contrib/ascendc/optimizations/evolution_strategies/cards/p19_custom_pingpong_double_buffer.md#L1-L63
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/knowledge/contrib/ascendc/optimizations/evolution_strategies/cards/p69_ub_fused_vector_chain.md#L1-L58
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/knowledge/contrib/ascendc/optimizations/evolution_strategies/cards/p71_matmul_ibshare_l1.md#L1-L50
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/knowledge/contrib/ascendc/optimizations/evolution_strategies/cards/p138_host_tiling_l0_capacity_guard.md#L1-L46
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/knowledge/contrib/ascendc/optimizations/evolution_strategies/cards/p145_multilaunch_no_same_launch_gm_rw.md#L1-L77
  promotion_status: verified
  source_locator: cann/cannbot-knowledge@be5882ef29fa9943305085652d2025a038ba491c:knowledge/contrib/ascendc/optimizations/evolution_strategies/cards/p69_ub_fused_vector_chain.md
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: Provides five concrete Ascend C code-transformation mutators (P69 UB vector
  chaining, P19 custom PingPong double-buffer, P71 Matmul IBShare L1 reuse, P138 runtime
  L0 capacity guard, and P145 multi-launch GM visibility).
tier: L4
title: 'Ascend C Five High-Yield Kernel Mutators: UB Vector Chain, Custom PingPong,
  Matmul IBShare, L0 Capacity Guard, and Multi-Launch GM Barrier'
triggers:
  error_signatures:
  - BuffersPolicyDB
  - CFG_IBSHARE_NORM
  - GetCoreMemSize(CoreMemType::L0_C
  keywords:
  - P19 custom PingPong
  - P69 UB fused vector chain
  - P71 Matmul IBShare
  - P138 L0 capacity guard
  - P145 multi-launch GM visibility
updated_at: '2026-10-21'
---
# Ascend C Five High-Yield Kernel Mutators: UB Vector Chain, Custom PingPong, Matmul IBShare, L0 Capacity Guard, and Multi-Launch GM Barrier

## Overview

Apply these five code-level mutators (`P69`, `P19`, `P71`, `P138`, `P145`) during Ascend C operator evolution when the corresponding PMU bottleneck or stability signature is observed.

## Mutator 1 (`P69`): UB Fused Vector Chain ($2n \to 2$ GM Transfers)

- **Precondition**: Kernel executes $n \ge 2$ sequential Vector operations where stage $k$'s output feeds stage $k+1$, and currently round-trips intermediate tensors through GM (`CopyIn -> ComputeExp -> CopyOut -> CopyIn -> ComputeAbs -> CopyOut`).
- **Transformation**: Keep intermediates resident in UB (`TQue<TPosition::VECCALC>` or in-place reuse of `dstLocal`) so `MTE2` and `MTE3` execute only once per tile:
```cpp
// Before: 4 GM DMA transfers (CopyIn -> Exp -> CopyOut -> CopyIn -> Abs -> CopyOut)
// After (P69): 2 GM DMA transfers via in-UB chaining
LocalTensor<float> srcLocal = inQueueSrc.DeQue<float>();
LocalTensor<float> dstLocal = outQueueDst.AllocTensor<float>();
AscendC::Exp(dstLocal, srcLocal, tileLen);
AscendC::Abs(dstLocal, dstLocal, tileLen);  // In-UB chain: zero intermediate GM traffic
outQueueDst.EnQue<float>(dstLocal);
inQueueSrc.FreeTensor(srcLocal);
```

## Mutator 2 (`P19`): Custom Multi-Level PingPong Double Buffer (`BuffersPolicyDB`)

- **Precondition**: Standard `TQue` FIFO (`BUFFER_NUM = 2`) cannot express Q-reuse (`GetPre`) or KV-reuse (`GetReused`) across `L1 / L0A / L0B / L0C` stages in Cube-heavy attention kernels.
- **Transformation**: Replace rigid `TQue` rotation with explicit `ping_` / `pong_` buffer handles paired with explicit `SetFlag`/`WaitFlag` event arrays (`event_id` $0..7$), ensuring on-chip footprint ($2 \times \text{tileSize}$) fits within hardware L1/L0 limits:
```cpp
template<BufferType bufferType, SyncType syncType>
class BuffersPolicyDB {
public:
    __aicore__ inline void Init(BufferManager<bufferType> &bm, uint32_t size) {
        ping_ = bm.template AllocBuffer<syncType>(size);
        pong_ = bm.template AllocBuffer<syncType>(size);
        ping_.Init(); pong_.Init();
    }
    __aicore__ inline Buffer<bufferType, syncType> &Get() {
        flag1_ ^= 1; return flag1_ ? pong_ : ping_;
    }
    __aicore__ inline Buffer<bufferType, syncType> &GetPre() {
        return flag1_ ? ping_ : pong_;
    }
};
```

## Mutator 3 (`P71`): `Matmul` L1 `IBShare` Across Paired MIX Sub-Cores

- **Precondition**: In a `MIX` kernel (`1 AIC : 2 AIV`), paired sub-blocks load the exact same $A$ or $B$ matrix tile from GM into L1 via redundant `MTE2` transfers (`aic_mte2_ratio` high).
- **Transformation**: Enable `isIBShare = true` (6th template parameter of `AscendC::MatmulType`) and `CFG_IBSHARE_NORM` so the first sub-core caches the shared matrix in L1 for its sibling sub-core (cutting shared matrix `MTE2` transfers by $50\%$):
```cpp
using B_TYPE = AscendC::MatmulType<AscendC::TPosition::GM, CubeFormat::ND,
    BType, false, LayoutMode::NONE, true>;  // 6th param true = enable L1 IBShare
AscendC::Matmul<A_TYPE, B_TYPE, C_TYPE, BIAS_TYPE, CFG_IBSHARE_NORM> mm;
```

## Mutator 4 (`P138`): Host Tiling Runtime `L0_A / L0_C` Capacity Guard

- **Precondition**: Cube kernel hardcodes `groupTile` or `baseM` assuming 128 KB `L0_C`, causing intermittent `InitBuffer` overflow / random `SIGSEGV` across SoC variants (`910B2C` vs `910C`).
- **Transformation**: Query `GetCoreMemSize(CoreMemType::L0_A / L0_C)` dynamically in `op_host` tiling and halve `groupTile` until it fits hardware `L0_C`:
```cpp
uint64_t l0aSize = 0, l0cSize = 0;
platformInfo->GetCoreMemSize(platform_ascendc::CoreMemType::L0_A, l0aSize);
platformInfo->GetCoreMemSize(platform_ascendc::CoreMemType::L0_C, l0cSize);
while (l0cSize > 0 && static_cast<uint64_t>(groupTile) * 256U * sizeof(float) > l0cSize) {
    groupTile /= 2;
}
if (groupTile <= 0) { groupTile = 1; }
```

## Mutator 5 (`P145`): Multi-Launch Stream Barrier for Cross-Phase GM Workspace Visibility

- **Precondition**: Multi-phase pure-`AIV` operator writes an intermediate workspace to GM via `MTE3` and reads it back via `MTE2` inside the **same kernel launch** on Ascend 910B, causing `npuSynchronize error` or stale reads because intra-launch `PipeBarrier` does not guarantee cross-core GM L2 write-back visibility.
- **Transformation**: Split phases that communicate via GM workspace into separate host `EXEC_KERNEL_CMD` launches (`mode = 0, 1, 2, ...`) on the same NPU stream so the hardware stream boundary enforces full GM visibility between phases.
